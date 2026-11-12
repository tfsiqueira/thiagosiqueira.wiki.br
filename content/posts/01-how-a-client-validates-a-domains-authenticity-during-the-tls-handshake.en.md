+++
title = "How a Client Validates a Domain's Authenticity During the TLS Handshake"
translationKey = "tls-handshake-domain-validation"
date = "2026-10-10"
author = "Thiago Siqueira"
tags = ["tls", "pki", "cryptography", "openssl", "networking"]
keywords = ["tls", "pki", "certificates", "openssl", "handshake"]
description = "A real PoC against mercadopago.com.br to understand, step by step, how OpenSSL validates a domain's certificate chain and what it actually means to trust a local trust anchor."
showFullContent = false
readingTime = true
hideComments = false
+++

When we open `https://mercadopago.com.br` in a browser, or when an application makes an HTTPS (Hypertext Transfer Protocol Secure) request to an API (Application Programming Interface), there's a question that needs to be answered before any sensitive data is exchanged.

Before the connection is accepted, client and server go through the handshake, the initial TLS negotiation in which cryptographic parameters are agreed upon and the server's identity is validated. That's the moment this question comes up:

*How does the client know it's really talking to the legitimate server for that domain?*

It's common to summarize TLS (Transport Layer Security) as "the protocol that encrypts the connection." That definition isn't wrong, but it's incomplete. Encryption alone doesn't prove identity. An attacker can also establish an encrypted connection with the client. What matters is knowing whether the public key presented on the other side actually belongs to the domain the client intended to reach.

This mechanism depends on a public key infrastructure, or PKI. Within it, certificate authorities, or CAs, sign certificates to bind domain names to public keys.

This article is based on a real PoC (Proof of Concept) I ran against `mercadopago.com.br`. The point isn't to memorize the specific chain observed, since public certificates change over time. The point is to follow the investigation step by step and answer, at each stage, what the client is actually verifying.

The thread running through all of it will always be the same:

- *What did the server send?*
- *What did the client verify?*
- *At what point did it decide to trust?*

The focus of this investigation is certificate and hostname validation. That's one part of TLS authentication; the handshake also needs to demonstrate that the server controls the private key matching the certificate. I come back to that distinction after settling the chain.

## Before starting: what I set out to reproduce

My goal is to capture the chain presented by a real server and understand how OpenSSL manages to validate it locally.

The experiment uses:

- `openssl s_client` to open a TLS connection;
- `openssl x509` to inspect certificates;
- `openssl verify` to reproduce the validation;
- OpenSSL's local trust store, installed via Homebrew — that is, the local repository of CAs that this particular OpenSSL already accepts as trusted.

I create a working directory:

```bash
mkdir -p ~/tmp/cert-lab-mercadopago
cd ~/tmp/cert-lab-mercadopago

openssl version -a
```

On the machine used for the exercise, the relevant output was `OPENSSLDIR: "/opt/homebrew/etc/openssl@3"`.

That detail looks small, but it matters later. Homebrew's OpenSSL uses its own CA bundle. It doesn't necessarily consult the same trust store used by native macOS APIs, by Chrome, by Java, or by an Android application.

## Question 1: what does the server actually send?

I start by opening a TLS connection to the server and asking OpenSSL to show every certificate presented:

```bash
openssl s_client \
  -connect mercadopago.com.br:443 \
  -servername mercadopago.com.br \
  -showcerts \
  </dev/null 2>&1 | tee s_client.txt
```

Each part of the command has a role:

| Snippet | Purpose |
| --- | --- |
| `-connect mercadopago.com.br:443` | Sets the host and port |
| `-servername mercadopago.com.br` | Sends SNI (Server Name Indication) so the server picks the right certificate |
| `-showcerts` | Shows the certificates sent by the server |
| `</dev/null` | Prevents the command from waiting on interactive input |
| `tee s_client.txt` | Saves the output for later analysis |

The `-servername` parameter deserves attention. SNI (Server Name Indication) is the TLS extension that sends the desired hostname right at the start of the connection. Since multiple domains can share the same IP (Internet Protocol) address, the server uses this information to decide which certificate to present.

Here, `-servername` selects the hostname sent via SNI; by itself, it doesn't trigger comparison of that name against the certificate. What's more, `s_client` is a diagnostic tool that can keep the connection going even after validation errors. So successfully capturing the certificates doesn't prove authentication was accepted. At this stage I'm just collecting material; further on, I use `openssl verify -verify_hostname` to explicitly test the chain and the hostname.

If I want the connection test itself to reject validation errors and check the name, I can use `-verify_return_error -verify_hostname mercadopago.com.br` alongside the connection options. These behaviors are described in the [s_client documentation](https://docs.openssl.org/3.6/man1/openssl-s_client/).

The `s_client.txt` file contains the certificates in PEM (Privacy-Enhanced Mail) format, a textual representation of the certificate's binary data. To split each certificate into its own file, I use:

```bash
awk '
/-----BEGIN CERTIFICATE-----/ {n++; file=sprintf("cert-%02d.pem", n)}
file {print > file}
/-----END CERTIFICATE-----/ {close(file); file=""}
' s_client.txt
```

The result was `cert-01.pem`, `cert-02.pem`, and `cert-03.pem`.

At this point, the only thing I know is that *the server sent three certificates*.

I still don't know which one is the leaf, which is the intermediate, and whether any of them is a root.

Before going further, it's worth fixing the terminology. The leaf is the end-entity certificate, the one closest to the domain being accessed. An intermediate is an intermediate CA, used to sign leaf certificates. A root is the CA at the top of the hierarchy, normally used as the anchor of trust.

## Question 2: how do I figure out each certificate's role?

I inspect the `subject`, `issuer`, `serial`, and validity dates of each one:

```bash
for cert in cert-*.pem; do
  echo "===== $cert ====="
  openssl x509 -in "$cert" -noout \
    -subject \
    -issuer \
    -serial \
    -dates
  echo
done
```

The observed output was:

- **cert-01.pem** — `subject=CN=www.mercadopago.com.br`; `issuer=C=US, O=Amazon, CN=Amazon RSA 2048 M04`
- **cert-02.pem** — `subject=C=US, O=Amazon, CN=Amazon RSA 2048 M04`; `issuer=C=US, O=Amazon, CN=Amazon Root CA 1`
- **cert-03.pem** — `subject=C=US, O=Amazon, CN=Amazon Root CA 1`; `issuer=C=US, ST=Arizona, L=Scottsdale, O=Starfield Technologies, Inc., CN=Starfield Services Root Certificate Authority - G2`

From this I can already put together a first reading:

| File | Likely role | Why |
| --- | --- | --- |
| `cert-01.pem` | Leaf | The `subject` is the end domain |
| `cert-02.pem` | Intermediate | The `subject` is the CA that signed the leaf |
| `cert-03.pem` | Amazon Root CA 1 certificate | The `subject` is `Amazon Root CA 1` |

In the fields above, `subject` identifies the entity the certificate represents, and `issuer` identifies who issued it. The `CN` (Common Name) attribute appears inside those names and was historically used to indicate the certificate's primary name.

But notice the word "likely." Looking at `subject` and `issuer` helps understand the chain, but it isn't cryptographic proof by itself.

The `issuer` field is just a statement inside the certificate. What actually proves the relationship between two certificates is the digital signature.

In other words:

*Cert-01 claims it was issued by Amazon RSA 2048 M04, but the client still needs to verify whether the signature was really made with that CA's private key.*

## Question 3: how do I know the leaf is valid for the domain I accessed?

The leaf certificate isn't accepted just because it's at the start of the chain. It needs to represent the correct hostname.

I check the leaf's SAN (Subject Alternative Name):

```bash
openssl x509 -in cert-01.pem -noout -ext subjectAltName
```

Output: `X509v3 Subject Alternative Name: DNS:www.mercadopago.com.br, DNS:mercadopago.com.br`.

This answers an important question:

*Does the leaf certificate cover the hostname I accessed?*

Yes.

SAN is the certificate extension that lists the domain names the certificate is valid for. The `DNS:` prefix in the output indicates those values are accepted domain names for the certificate. Since I accessed `mercadopago.com.br`, the client expects to find that hostname in the leaf's SAN. If the certificate had been issued for a different domain, the connection should be rejected, even if the entire signature chain were valid.

Historically, the `CN` (Common Name) field was also used to indicate the primary domain. In modern validation, however, the client must compare the hostname against the SAN, not just the CN.

There are also wildcard certificates, such as `*.mercadopago.com.br`. In that case, the wildcard covers a single level on the left, like `api.mercadopago.com.br`, but doesn't cover something like `a.b.mercadopago.com.br`. Hostname matching rules are stricter than simply looking for a similar-looking suffix.

This is an important point: TLS authentication isn't just "the chain was signed by a trusted CA." It's also:

*Does this trusted chain actually represent the hostname I requested?*

## Question 4: how does a chain like this come to exist?

Before continuing with the PoC, it's worth understanding how these certificates were created.

A certificate authority, or CA, starts by generating a key pair: a CA private key and a CA public key.

The private key is used to sign certificates. The public key is distributed inside certificates so clients can verify those signatures.

At the top of the hierarchy, a root CA — the root certificate authority — creates a certificate for itself, with `subject = Root CA` and `issuer = Root CA`.

This is a self-signed certificate, meaning it's signed by the private key of the very CA it represents.

Next, an intermediate CA generates its own key pair and creates a CSR (Certificate Signing Request) containing the intermediate CA's own public key. The root signs that intermediate certificate.

Finally, the server generates its key pair, creates a CSR for the desired domain containing the server's own public key, and the intermediate CA signs the leaf certificate.

A CSR isn't a finished certificate. It's an issuance request that normally carries:

- the requester's public key, which will go into the certificate;
- the desired subject;
- the desired SANs;
- a signature made with the corresponding private key, to prove the requester really holds that key.

For a public domain certificate, the CA still has to validate control of that domain before issuing the leaf. The CA doesn't put its own public key into the leaf; it puts the server's public key in and signs the certificate with the CA's private key.

The picture looks like this:

**Root CA → signs → Intermediate CA → signs → Leaf certificate**

In Mercado Pago's case:

**Amazon Root CA 1 → signs → Amazon RSA 2048 M04 → signs → www.mercadopago.com.br**

In cryptographic terms:

- leaf signature → made with the intermediate's private key
- intermediate signature → made with the root's private key

During validation, the client goes the other way:

- uses the intermediate's public key to validate the leaf
- uses the root's public key to validate the intermediate

But validation doesn't stop at signatures. The client also needs to check:

- whether the accessed hostname appears in the leaf's SAN;
- whether the certificates are within their validity period;
- whether the CA certificates carry extensions compatible with a CA role;
- whether the chain respects path-building constraints, i.e. whether each certificate is allowed to occupy its role in the chain and sign the next one;
- whether the path ends at a trust anchor accepted locally.

I'll come back to these rules after settling the PoC's main question: which path did OpenSSL actually choose to validate this chain?

## Question 5: did the server send the final root?

At first glance, `cert-03.pem` looks like a root because its `subject` is `Amazon Root CA 1`.

But it isn't a self-signed root.

Look again: `subject=C=US, O=Amazon, CN=Amazon Root CA 1`; `issuer=C=US, ST=Arizona, L=Scottsdale, O=Starfield Technologies, Inc., CN=Starfield Services Root Certificate Authority - G2`.

If it were self-signed, I'd expect `subject = Amazon Root CA 1` and `issuer = Amazon Root CA 1`.

But the `issuer` is Starfield.

That already raises a new question:

*How can a certificate represent Amazon Root CA 1 while being signed by Starfield?*

But for now, the answer to the current question is already in the `issuer`: the last certificate sent by the server still points to another CA. So it isn't the final self-signed root.

The answer is:

*The server didn't send the final root that OpenSSL used as a trust anchor — that is, the endpoint already trusted locally to close the validation.*

## Question 6: if the server didn't send the final root, how did validation pass?

Now I combine the auxiliary certificates sent by the server:

```bash
cat cert-02.pem cert-03.pem > intermediates.pem
```

And ask OpenSSL to validate the leaf:

```bash
openssl verify \
  -purpose sslserver \
  -show_chain \
  -verify_hostname mercadopago.com.br \
  -untrusted intermediates.pem \
  cert-01.pem
```

`-purpose sslserver` requests verification for use as a TLS server, including compatibility of the certificate's usage extensions, such as Extended Key Usage (EKU), when present. Without that option, `openssl verify` doesn't automatically apply that same purpose. The hostname is checked separately via `-verify_hostname`.

The relevant output is `cert-01.pem: OK`. The chain (`Chain`) shown is:

- `depth=0: CN=www.mercadopago.com.br (untrusted)`
- `depth=1: C=US, O=Amazon, CN=Amazon RSA 2048 M04 (untrusted)`
- `depth=2: C=US, O=Amazon, CN=Amazon Root CA 1`

This output is extremely interesting.

The leaf and the intermediate show up as `untrusted` because they were supplied to OpenSSL as auxiliary material for building the chain, not as trust anchors. In this experiment, that material came from the server. That doesn't mean they're malicious.

But `depth=2`, `Amazon Root CA 1`, doesn't show up as `untrusted`.

That indicates OpenSSL found a trusted `Amazon Root CA 1` locally and closed the chain there.

In other words:

The server sent: **Leaf → Amazon RSA 2048 M04 → Amazon Root CA 1 cross-signed**.

OpenSSL validated: **Leaf → Amazon RSA 2048 M04 → Amazon Root CA 1 local**.

I still need to carefully explain what `cross-signed` means. For now, the important point is that the certificate sent by the server wasn't the certificate that closed the validation.

This is one of the most important ideas in the whole article:

*The chain presented by the server doesn't have to be identical to the chain chosen by the client.*

The server hands over material for the client to build a path.

The client chooses the path it can validate up to a local trust anchor.

## Question 7: where did that local trust anchor live?

Early in the PoC, the `openssl version -a` command had already shown that the Homebrew-installed OpenSSL was using `OPENSSLDIR: "/opt/homebrew/etc/openssl@3"`.

Now that clue starts to matter. Inside that directory, the bundle of trusted CAs used by this OpenSSL lived at `/opt/homebrew/etc/openssl@3/cert.pem`.

I make the validation explicit by using that bundle directly:

```bash
openssl verify \
  -no-CApath \
  -no-CAstore \
  -CAfile /opt/homebrew/etc/openssl@3/cert.pem \
  -purpose sslserver \
  -show_chain \
  -verify_hostname mercadopago.com.br \
  -untrusted intermediates.pem \
  cert-01.pem
```

The result was still `OK`, confirming that the trust anchor OpenSSL used came from that local bundle.

This point matters because there's no such thing as a universal trust store.

It's worth separating the two terms. A trust store is the local set of trusted CAs available to the client. A trust anchor is the specific certificate or key, within that set, that closes a particular validation.

| Environment | Primary trust store |
| --- | --- |
| Native macOS | Keychain |
| Homebrew OpenSSL | Its own bundle in `cert.pem` |
| Android | System store, user store, and application-embedded anchors |
| Firefox | Its own NSS-based store |
| Java | `cacerts` |

A chain can be valid in the browser and invalid in OpenSSL. It can be valid on macOS and invalid in a JVM. It can be accepted by the system Android store and rejected by an application that embeds its own anchors.

The trust anchor is always a local decision made by the client.

It's also worth keeping a nuance in mind: a trust anchor is a role in validation, not just a type of certificate. The locally self-signed `Amazon Root CA 1` acted as the trust anchor in the path I observed. The cross-signed `Amazon Root CA 1` sent by the server, in the legacy path, would act as an additional intermediate certificate up to Starfield.

## Question 8: how do I prove there was a local Amazon Root CA 1?

I extract the certificates from the OpenSSL bundle:

```bash
mkdir -p roots

awk '
/-----BEGIN CERTIFICATE-----/ {n++; file=sprintf("roots/root-%03d.pem", n)}
file {print > file}
/-----END CERTIFICATE-----/ {close(file); file=""}
' /opt/homebrew/etc/openssl@3/cert.pem
```

Then I look for `Amazon Root CA 1`:

```bash
for cert in roots/root-*.pem; do
  subject=$(openssl x509 -in "$cert" -noout -subject)
  if echo "$subject" | grep -q "Amazon Root CA 1"; then
    echo "$cert"
    openssl x509 -in "$cert" -noout -subject -issuer -serial -dates
  fi
done
```

I find `roots/root-022.pem`, with `subject=C=US, O=Amazon, CN=Amazon Root CA 1` and `issuer=C=US, O=Amazon, CN=Amazon Root CA 1`.

The matching `subject` and `issuer` indicate a *self-issued* certificate, issued in the name of the entity itself. To conclude it's *self-signed*, I also need to check its signature using its own public key. I make that check explicit:

```bash
openssl verify \
  -no-CApath -no-CAstore \
  -CAfile roots/root-022.pem \
  -check_ss_sig \
  roots/root-022.pem
```

The output is `roots/root-022.pem: OK`. The `-check_ss_sig` flag requests verification of the root's own signature, which normally isn't necessary to decide whether to trust an anchor that's already configured. In this test, I'm choosing the certificate as the anchor in order to check its signature; that alone doesn't justify trusting it. The trust decision remains external to self-signing.

That confirms the local self-signed version of `Amazon Root CA 1`.

An important distinction shows up here:

*A root isn't trustworthy because it's self-signed; it's trustworthy because it was added to the client's trust store.*

Anyone can generate a self-signed certificate. What turns a root into a trust anchor is the local decision by the system, runtime, or application to trust it.

## Question 9: how can Amazon Root CA 1 exist as two different certificates?

Now I get to the most subtle point of the PoC.

I compare the certificate sent by the server with the local root:

```bash
echo "===== cert-03.pem sent by the server ====="
openssl x509 -in cert-03.pem -noout -subject -issuer -fingerprint -sha256

echo
echo "===== local root ====="
openssl x509 -in roots/root-022.pem -noout -subject -issuer -fingerprint -sha256
```

The conceptual result was, comparing the certificate sent by the server (`cert-03.pem`) with the local root (`roots/root-022.pem`):

| Field | Sent | Local |
| --- | --- | --- |
| Subject | Amazon Root CA 1 | Amazon Root CA 1 |
| Issuer | Starfield Services Root Certificate Authority - G2 | Amazon Root CA 1 |
| Fingerprint SHA-256 | `87:DC:D4:DC:74:64:0A:32:2C:D2:05:55:25:06:D1:BE:64:F1:25:96:25:80:96:54:49:86:B4:85:0B:C7:27:06` | `8E:CD:E6:88:4F:3D:87:B1:12:5B:A3:1A:C3:FC:B1:3D:70:16:DE:7F:57:CC:90:4F:E1:CB:97:C6:AE:98:19:6E` |

This proves they are different certificates.

Fingerprint is the hash of the entire certificate. If any detail of the certificate changes — issuer, serial number, validity, or signature — the fingerprint changes too.

But there was still a question left to answer:

*If they're different certificates, why do both work to validate the same CA?*

## Question 10: what's the same between them?

I compare the hash of the SPKI (Subject Public Key Info) for both:

```bash
echo "===== SPKI hash for cert-03.pem ====="
openssl x509 -in cert-03.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256

echo "===== SPKI hash for local root ====="
openssl x509 -in roots/root-022.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256
```

Output, identical in both cases:

`SHA2-256(stdin)= fbe3018031f9586bcbf41727e417b7d1c45c2f47f93be372a17b96b50757d5a2`

The fingerprint of the whole certificate was different.

The SPKI hash was the same.

This means both certificates carried the same public key.

SPKI is the certificate structure that describes the public key: which algorithm it uses and what the key's bytes actually are.

That's the fundamental difference:

- **Fingerprint**: hash of the entire certificate
- **SPKI hash**: hash of the public key structure

If any field of the certificate changes, the fingerprint changes:

- issuer;
- serial number;
- validity;
- extensions;
- signature.

But if the public key stays the same, the SPKI stays the same.

## Question 11: what does cross-signed actually mean?

A self-signed Amazon Root CA 1 root has `subject = Amazon Root CA 1` and `issuer = Amazon Root CA 1`.

It's a certificate issued by Amazon Root CA 1 itself, for its own public key.

The cross-signed version, on the other hand, has `subject = Amazon Root CA 1` and `issuer = Starfield Services Root Certificate Authority - G2`.

It carries the same Amazon Root CA 1 public key, but it was signed by Starfield.

The conceptual process is:

1. Amazon generates the Amazon Root CA 1 key pair.
2. Amazon creates a self-signed certificate for that key.
3. To get a cross-signed version, Amazon prepares a certification request to Starfield, typically represented by a CSR.
4. That CSR carries Amazon Root CA 1's public key and is signed with the matching private key, proving Amazon controls that key.
5. Starfield uses that request to issue another certificate for the same Amazon Root CA 1 public key.
6. That second certificate is signed by Starfield.

A note on language here: it isn't the certificate that "generates" the CSR. It's the entity operating the CA — in this case, Amazon — that creates that request using the private key associated with Amazon Root CA 1. The important conceptual point is that Starfield never receives Amazon's private key; it receives a request that contains the public key and a proof of possession of the matching private key.

So two different certificates emerge: the **Amazon Root CA 1 self-signed** and the **Amazon Root CA 1 cross-signed**.

In this case, they represent the same logical CA because they share the same subject, carry the same public key, and are both CA certificates.

They're different certificates because they were issued in different contexts and signed by different authorities.

## Question 12: why would anyone do this?

For compatibility.

When Amazon Root CA 1 first appeared, modern clients could receive that root in their trust stores and trust it directly.

But older clients might not have had that root installed yet.

Those older clients already trusted older roots, like Starfield.

So cross-signing builds a bridge:

Modern client: **Leaf → Amazon RSA 2048 M04 → local Amazon Root CA 1**.

Legacy client: **Leaf → Amazon RSA 2048 M04 → cross-signed Amazon Root CA 1 → Starfield Root**.

On the path I observed, OpenSSL ends at the local Amazon Root CA 1. The path up to Starfield shows the alternative that cross-signing makes possible on a compatible client that trusts that other anchor; it's not a second run demonstrated in this PoC.

"Modern" and "legacy" are shorthand here for two trust scenarios. The choice doesn't depend only on the client's age: it also depends on the anchors available, the policies in place, and the path-building rules. The presence of the cross-signed certificate doesn't guarantee acceptance by just any old client.

## Question 13: why can the modern client stop at the local root instead of continuing through the cross-signed version?

Because the intermediate's signature was made with Amazon Root CA 1's private key.

To verify that signature, the client needs the matching public key.

Both the self-signed version and the cross-signed version carry the same public key.

So:

**Amazon Root CA 1 self-signed public key = Amazon Root CA 1 cross-signed public key**

That's why both are able to validate the `Amazon RSA 2048 M04` signature.

The client doesn't literally swap one certificate for another. The path builder finds a trusted local candidate with the same subject and the same public key, and builds an alternate path to it.

The modern client doesn't need to continue up to Starfield because it already has a local Amazon Root CA 1 trust anchor that carries that same public key.

The legacy client, meanwhile, uses the cross-signed certificate as an additional intermediate:

1. Validates the leaf with Amazon RSA 2048 M04's public key.
2. Validates Amazon RSA 2048 M04 with Amazon Root CA 1's public key.
3. Validates the cross-signed Amazon Root CA 1 with Starfield's public key.
4. Closes the chain at the local Starfield root.

## One more layer: what else does the client validate?

At this point, I've solved the central mystery of the PoC: the server sent one chain, but OpenSSL built a different one up to a compatible local trust anchor.

Still, it's worth returning to an observation left open in Question 4. A chain isn't accepted just because the signatures match. The client also checks whether each certificate is allowed to occupy the role it's occupying.

One of the most important extensions for this is `basicConstraints`.

If I want to inspect it in the captured certificates, I can use:

```bash
openssl x509 -in cert-01.pem -noout -ext basicConstraints
openssl x509 -in cert-02.pem -noout -ext basicConstraints
```

What I'm looking for in that output is the difference between `CA:FALSE` and `CA:TRUE`.

A leaf normally shows up with `CA:FALSE`. That means it can represent a server, but it can't be accepted as a CA capable of signing another certificate beneath it in a valid chain.

An intermediate normally shows up with `CA:TRUE`. That means it can be accepted as a certificate authority and sign leaf certificates.

This distinction stops someone from taking an ordinary server certificate and trying to use it as if it were an intermediate CA. The matching private key could still produce a cryptographically valid signature, but the chain should be rejected, because that certificate isn't authorized to act as a CA.

On top of that, a CA can carry a constraint like `pathlen:0`. In that case, it can sign leaf certificates, but it can't sign another intermediate below it. This is a way to limit how deep the chain can go and prevent the hierarchy from growing indefinitely.

Another related extension is `keyUsage`. Even if a certificate has `CA:TRUE`, it normally also needs to permit certificate signing to actually act as a CA.

Finally, this article doesn't cover revocation mechanisms such as CRL (Certificate Revocation List) and OCSP (Online Certificate Status Protocol). They're another important part of validation in some clients, but weren't necessary to answer this PoC's central question.

## The certificate is public; how does the server prove it controls the key?

Up to this point, I'm checking whether a certificate ties a public key to the expected domain via an accepted trust path. But anyone can copy those public certificates. Presenting them, by itself, doesn't demonstrate control of the server's private key.

In TLS 1.3 certificate-based authentication, the server sends a `CertificateVerify` message with a signature tied to the content of the negotiation. The client uses the leaf's public key to verify it. That demonstrates possession of the matching private key and binds that proof to the handshake in progress. The `Finished` message also plays a role in protecting the integrity of the negotiation. This mechanism is described in [RFC 8446, section 4.4](https://www.rfc-editor.org/rfc/rfc8446.html#section-4.4).

So there are two distinct checks: trusting the association between domain and public key, and confirming that the party on the connection controls the matching private key. Validating the PEMs with `openssl verify` tests the first part; it doesn't reproduce a proof of possession on a live connection.

## Materials and references to reproduce this

I preserved the public certificates and the original output in the [observed chain package](#TODO-link-github-tls-chain-snapshot-zip). A fresh capture of the domain may show different certificates. To study these exact files, I extract the package and use the preserved PEMs; the name `root-022.pem` was only its position in the original bundle, not a stable identifier for the CA.

In the package, the local root is named `amazon-root-ca-1-local.pem`. From the extracted folder, I reproduce the validation with:

```bash
openssl verify \
  -no-CApath -no-CAstore \
  -CAfile amazon-root-ca-1-local.pem \
  -purpose sslserver \
  -verify_hostname mercadopago.com.br \
  -untrusted intermediates.pem \
  -show_chain cert-01.pem
```

For this revision, I repeated the validation of the preserved files with OpenSSL 4.1.0 and got `OK`. The [validation log](#TODO-link-github-validation-txt) includes that run, the fingerprints, and the check of the root's signature. Certificates have a validity period; `OK` describes the run on record, not a promise about future validity.

To check the rules used:

- [OpenSSL — verification options](https://docs.openssl.org/3.6/man1/openssl-verification-options/): path building, purpose, and trust anchors.
- [OpenSSL — s_client](https://docs.openssl.org/3.6/man1/openssl-s_client/): capture and behavior in the face of errors.
- [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280.html): X.509 certificate profile and validation.
- [RFC 8446](https://www.rfc-editor.org/rfc/rfc8446.html): TLS 1.3 authentication and handshake.

## What the experiment actually taught me

The investigation shows a concrete difference between the chain presented and the chain validated: the server offers the cross-signed Amazon Root CA 1, but OpenSSL closes the path at the self-signed version available locally. The hostname matches the SAN, and explicit validation confirms the path for use as a TLS server.

The different fingerprints identify different documents; the matching SPKI hash reveals the same public key. That combination explains why distinct certificates can take part in alternate paths to the same logical CA.

## Final mental model

After walking through the experiment, I break the responsibilities down below. This list is a conceptual model, not the mandatory order of execution for every TLS client's checks:

1. The server presents certificates.
2. The client checks the hostname against the leaf's SAN.
3. The client validates signature by signature.
4. The client tries to build a path to a local trust anchor.
5. The chain used by the client can differ from the chain presented by the server.
6. On a certificate-authenticated connection, the server also demonstrates possession of the matching private key; the public certificates, on their own, aren't enough.

And the most important idea of all:

**A certificate is not the same thing as a public key.**

A certificate is a signed document.

A public key is the cryptographic material that document associates with an entity.

Because of that, two certificates can:

- have different issuers;
- have different serial numbers;
- have different fingerprints;
- and, when they preserve the same public key and remain CA certificates, still represent the same certificate authority.

In TLS, trusting a domain doesn't mean trusting a single isolated file. It means verifying that the requested hostname appears in the leaf, and trusting a path of signatures that ends at a trust anchor already accepted locally by the client.

## Quick reference for the terms used

| Term | Meaning |
| --- | --- |
| HTTPS (Hypertext Transfer Protocol Secure) | HTTP carried over a TLS connection |
| Handshake | The initial negotiation where client and server agree on parameters and validate identity |
| TLS (Transport Layer Security) | Protocol that protects the connection's confidentiality, integrity, and authenticity |
| PKI (Public Key Infrastructure) | Infrastructure that organizes keys, certificates, and certificate authorities |
| CA (Certificate Authority) | Certificate authority that signs certificates |
| SNI (Server Name Indication) | TLS extension that tells the server which hostname the client wants to reach |
| PEM (Privacy-Enhanced Mail) | Text-based format used to represent certificates and keys |
| Leaf | The end-entity certificate, closest to the domain being accessed |
| Intermediate | Intermediate CA used to sign leaf certificates |
| Root CA | The CA at the top of the trust hierarchy |
| CN (Common Name) | Legacy name attribute inside the certificate |
| SAN (Subject Alternative Name) | Extension listing the hostnames the certificate is valid for |
| CSR (Certificate Signing Request) | Issuance request carrying the requester's public key and the desired attributes for the certificate |
| basicConstraints | Extension indicating whether a certificate can act as a CA and, when present, limiting how many additional CAs can exist below it |
| keyUsage | Extension indicating which cryptographic operations the certificate's key can be used for |
| CRL (Certificate Revocation List) | List published by a CA with revoked certificates |
| OCSP (Online Certificate Status Protocol) | Protocol for querying a certificate's revocation status |
| Trust store | The local set of trusted CAs available to the client |
| Trust anchor | The certificate or key trusted enough to close a given validation |
| SPKI (Subject Public Key Info) | The certificate structure that describes the public key |
| Fingerprint | Hash of the entire certificate |
| pathlen | A constraint on a CA certificate limiting how many additional intermediate CAs can exist below it in the chain |
| Path building | The process by which the client assembles a path of certificates, from the leaf to a local trust anchor, from the available material |
| Self-signed | A certificate signed by the private key of the very entity it represents |
| Cross-signed | A certificate from one CA signed by another CA |
