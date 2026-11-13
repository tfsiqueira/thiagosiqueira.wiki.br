+++
title = "Why Backup Pins and Rotation Matter in Production"
translationKey = "android-pinning-backup-pins-rotation"
date = "2026-11-13"
author = "Thiago Siqueira"
tags = ["android", "ssl-pinning", "mobile", "pki", "operations"]
keywords = ["android", "certificate pinning", "backup pin", "key rotation", "network security config"]
description = "I reproduce a server key rotation to show why a client that only knows the old pin loses the connection, while another prepared with a backup pin keeps working, and what that continuity requires from operations."
showFullContent = false
readingTime = true
hideComments = false
+++

An app can talk to its server, get the expected response, and correctly apply certificate
pinning. Some time later, the team replaces the server's TLS certificate. The domain stays
the same, the new certificate is valid, and the change is legitimate. Even so, the app stops
connecting.

What is it rejecting?

In the previous articles, I investigate trust in TLS (Transport Layer Security), separate
trust store failures from pinning failures, and show how a local check can be identified and
altered. Now I want to look at the control during a normal service change, while the app
keeps running its original policy.

The question that closes the series is: *how do you rotate the server's legitimate key without
losing the connection to clients already installed?*

To answer it, I follow a key rotation and compare the behavior of two clients. The experiment
provides the evidence, but the question is broader: a trust decision recorded in the app needs
to stay compatible with a server that will change over time.

## The connection works; what exactly does the app expect to find?

I start with an Android app that queries an HTTPS (Hypertext Transfer Protocol Secure) service
and gets a successful response. The server's certificate covers the accessed domain, is within
its validity period, and its chain reaches a certificate authority accepted by the app. I'll
call that authority CA (Certificate Authority).

Beyond those checks, the client pins the public key of the server's final certificate, also
called the leaf. I'll call that key **A**.

The pin configured in the app is a hash of the SPKI (Subject Public Key Info), the structure
that describes the public key inside the certificate. In the implementation used in this study,
that hash is computed with SHA-256. The app doesn't store the server's private key: it stores
a value that lets it recognize the expected public key.

That adds a condition to the connection. The chain can be trustworthy and the hostname correct,
yet a different key can still be rejected by pinning.

As long as the server presents a certificate with A, both conditions are satisfied: normal TLS
validation accepts the certificate, and the extra check finds the expected pin. The app gets
`HTTP 200`, the status code of a successful HTTP response.

So far, the control is working. To understand what can happen during maintenance, I first need
to separate two changes that tend to get treated as equivalent: replacing the certificate and
replacing the key.

## Question 1: does a new certificate need a new key?

In the first article, I compare different certificates that carry the same public key. That
distinction also matters during a renewal.

A certificate is a signed document: it contains the public key, the names it was issued for,
its validity period, and other fields. The fingerprint is the hash of that entire document.
The SPKI pin I'm studying corresponds to the structure of the public key.

If a certificate is reissued while preserving the same SPKI, its serial number, validity, and
signature can change, and so can the fingerprint. The pin on that key, though, still matches.
The connection will still depend on the new certificate and its chain passing the remaining
checks.

I verify that distinction before rotating the key. First, the server uses the original
certificate with A. Then, I present a reissued certificate with the same key, but with a
different serial number and validity period. The client keeps accepting only A's pin.

The SHA-256 fingerprint changes from `92:41:74:CF…` to `17:2D:64:C1…`, while the SPKI hash
stays the same. In both cases, the app gets `HTTP 200`.

| Observed evidence | Original certificate | Reissued certificate |
| --- | --- | --- |
| SHA-256 fingerprint, abbreviated | `92:41:74:CF…` | `17:2D:64:C1…` |
| Public key | A | A |
| Client that accepts only A | HTTP 200 | HTTP 200 |

The [full values and public certificates](#TODO-link-github-evidence-article-04-certificate-comparison)
let you check the comparison. The reissue changed the document but preserved the pin match.
That's why the claim "the renewal broke pinning" needs a follow-up question: **did the public
key also change?**

| Change | What happens to the server key's pin |
| --- | --- |
| Certificate reissued with the same SPKI | The SPKI hash still matches |
| Certificate issued with a different public key | The client needs to know a pin compatible with that other key |
| Certificate expired or invalid for the domain | A matching pin doesn't waive the certificate's normal validation |

To investigate the second case, I replace A with another key pair. I'll call the new public
key **B**. The server gets a certificate that contains B and uses the matching private key
during the connection.

The new certificate is still valid for the same domain and is issued by the same CA. What needs
to change for pinning is the hash of the presented public key.

## Question 2: why does the app reject a server that's still legitimate?

After the rotation, the client that only knows A stops getting a response. The screen shows a
failure, with `Exception: SSLHandshakeException` and root cause `CertificateException: Pin
verification failed`.

<img src="/images/posts/04-por-que-backup-pins-e-rotacao-importam-em-producao/profile-b-nscPinnedPrimaryOnly.png" alt="Client that only knows A showing FAILURE in red and Pin verification failed after the legitimate rotation to key B" class="center" width="400">

*After the legitimate rotation to B, the client that only knows A rejects the connection because the pins don't match.*

This result resembles the proxy rejection studied in the second article. But this time, the
connection goes straight to the legitimate server. There's no intermediary swapping the
certificate along the way.

I check both certificates: both are still valid for the hostname and for use as a TLS server,
with a chain that terminates at the CA accepted by the clients. The relevant difference for the
app's rule is the public key.

The server now presents B. The client still expects A.

- **Before the rotation:** client expects A → server presents A → connection accepted.
- **After the rotation:** client expects A → server presents B → pinning rejects.

The CA states that the new certificate is valid for that domain. The client, in turn, keeps
applying a more specific restriction: it only accepts a chain that matches the pins it received.

That rule can't tell apart the intent behind the change. An unexpected key can come from an
interception or from legitimate maintenance. In both cases, the expected match is missing.

That's why configuring pinning correctly isn't enough to guarantee continuity. By distributing
A's pin, I also create a dependency between the already-installed app and the server's future
configuration.

## Question 3: could that interruption have been avoided?

To answer that, I compare the result with another client. It queries the same service and runs
the same certificate checks, but already knows two alternatives: the pins for A and for B. That
configuration was installed before the server's rotation.

As long as the server still uses A, both clients work. After B is promoted, the behavior splits:

| Policy installed on the client | Server with A | Server with B |
| --- | --- | --- |
| Accepts only A's pin | Connects | Rejects due to pinning |
| Accepts both A's and B's pins | Connects | Connects |

In the previous question, the client that only knows A rejected the connection. The capture
below shows the client that already knows A and B, connecting directly to the same server after
the same legitimate rotation.

<img src="/images/posts/04-por-que-backup-pins-e-rotacao-importam-em-producao/profile-b-nscPinned.png" alt="Client that already knows A and B getting HTTP 200 after the same key rotation on the server" class="center" width="400">

*The client stays connected because B was already an alternative accepted by the installed policy.*

The second client keeps getting `HTTP 200`. I don't need to remove the pin check or install a
new CA between the two states. B satisfies an alternative the app had already received.

It's this alternative, prepared ahead of time, that I call a **backup pin**.

The word "backup" might suggest a mechanism that stays off until the main key fails. In the
configuration used here, the pin set works as a list of accepted alternatives. There's no
priority forcing Android to try A before allowing B.

B's pin already participates in the decision as soon as the policy is in use. Its backup role
comes from planning: the server still uses A, but the client is prepared to accept B. The
platform code applies that match in
[`NetworkSecurityTrustManager.checkPins`](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/security/net/config/NetworkSecurityTrustManager.java).

It's also not necessary for the server to present both keys at the same time for this
experiment to work. It moves from a certificate with A to another with B; the client's policy
accepts both possibilities.

Here, the pins correspond to the leaf's keys. Pinning a CA's key would change the scope of the
restriction: other certificates with new server keys could stay compatible with that CA. This
article's comparison keeps the pin on the server's key to make the rotation's effect explicit.

## Question 4: is a second hash enough to have a backup?

The result shows the value of accepting B ahead of time. But it opens another question: if I
drop any extra hash into the app, do I already have a usable alternative?

To accept the connection, the app needs to find a match with a pin. To offer that connection,
the server needs more than that: B's private key and a valid certificate that contains the
matching public key.

If I configure a hash without access to the associated private key, I can't simply turn it into
an identity the server can use. The same problem shows up if I have the key but can't obtain or
install its certificate when I need to make the switch.

In the experiment, B is a key pair separate from A. B's certificate is issued using the public
key that corresponds to the backup pin already configured. The SPKI comparison confirms that
relationship before the promotion.

That order matters: if I generate another key at the moment of the change, it will have a
different SPKI and won't match the backup I distributed.

So preparing the backup involves two related responsibilities. The client's policy needs to
accept B, and the server's operation needs to be able to use B. That includes preserving the
private key properly, controlling access to it, and having a viable path for issuing and
installing the certificate.

The local experiment demonstrates the match between key and pin. In a real service, the ability
to execute that rotation needs to be backed by key and certificate management.

## Question 5: does the backup also protect clients that haven't updated yet?

The client with A and B survives the change because it received B's pin before it happened.
That observation imposes an important limit: the backup only helps installations that already
know it.

Imagine an old version was shipped with only A. Later, I publish a version that accepts A and B.
As long as the server stays on A, both can connect. That period lets me distribute the new
configuration without immediately requiring the new key.

When the server switches to B, though, the old version still expects only A. Publishing the
update doesn't change the policy of installations that haven't received it yet.

There are users who delay updates, devices that go offline, and versions that stop receiving new
releases. The decision to promote B needs to account for which of those clients are still
supported. There's no universal number of days that guarantees everyone updates.

I start to see rotation as a change coordinated between two sides: the server can adopt B
quickly, but knowledge of B needs to reach a population of clients with different update paces.

It's that difference in pace that turns the backup pin into something operationally important.
It opens a compatibility window. On its own, it doesn't guarantee the whole base is inside that
window.

## Question 6: once B becomes the current key, what happens at the next rotation?

After promoting B, I could keep the A+B policy indefinitely. It would keep accepting the current
key, but wouldn't prepare a future key C.

To repeat the transition with continuity, I need to distribute a policy that accepts B and C
while the server still uses B. The same logic that allowed the first rotation now guides the
next one:

| Moment | Key used by the server | Policy I prepare on the clients |
| --- | --- | --- |
| Before the first rotation | A | Accept A and B |
| After promoting B | B | Distribute a version that accepts B and C |
| After preparing clients for C | C | Keep an alternative for the next change |

While the server uses B, A+B and B+C clients share one accepted key. That lets me update the
configuration without immediately breaking the connections of whoever is still on the earlier
version.

But when the server switches to C, clients stuck on A+B run into the same problem again.
Continuity depends on distributing the next alternative before requiring it.

There's also a consequence for removing old keys. Switching the server to B doesn't remove A
from the already-installed A+B policy. The old pin stays a recognized alternative in that
configuration, subject to the certificate's other checks.

If the motivation for the rotation is a compromise of A, that matters. Stopping the use of a key
on the server and stopping accepting it on the clients are different changes. Policy updates,
certificate validation, and incident response procedures need to be considered together.

So planning doesn't end when B goes into use. It needs to include the next key and the fate of
keys that are no longer desirable.

## Question 7: does expiring the pins solve the incompatibility?

There's another mechanism that might look like a way out for clients that don't update: pin
expiration.

On Android, the Network Security Config (NSC) lets you set a date in the `expiration` attribute
of the pin set. After that date, those pins stop being applied; without that attribute, the set
doesn't get an expiration date at all. The
[NSC reference](https://developer.android.com/privacy-and-security/security-config#pin-set)
documents that behavior.

That expiration is different from the server certificate's validity. If the certificate is still
valid, but the connection was failing only because no pin matched, stopping the pin enforcement
can let it work again. If the certificate is expired or doesn't represent the accessed domain,
pin expiration doesn't fix that other problem.

This creates a trade-off between continuity and how long the extra restriction lasts. A client
that never receives new pins can stop requiring that match after the configured date. At the
same time, it loses the extra protection the pins provided.

I don't treat expiration as automatic renewal, nor as a substitute for distributing the next
keys. It needs to be part of the maintenance policy: how long does the team want to enforce that
restriction on a client that stayed on the old configuration?

## Question 8: what changes when the rotation has to be an emergency?

In the planned transition, B is already prepared and its pin has already been distributed. An
emergency can shrink that room to maneuver. The backup key might be unavailable, certificate
issuance might take too long, or the required change might demand a different key altogether.

The extra pin doesn't remove those dependencies. To count on B as a recovery path, I need to
know whether I can bring it into use within the needed time and which clients will be able to
recognize it.

I also need to think about the way back. If a planned rotation runs into a problem, temporarily
returning to A might be an option. If A was compromised, that same rollback can work against the
purpose of the change. "We have rollback" is only a useful answer when I know which prior state
is still acceptable.

Another concern shows up when I consider updating pins remotely. If the app needs to fetch its
new configuration through the same endpoint whose key it's rejecting, it can end up without the
channel it needs to receive the fix. A remote configuration solution would need its own design
for authenticity, integrity, and recovery.

Removing chain or hostname validation to restore communication also changes the problem: the
connection loses basic guarantees about authenticating the server. Recovery needs to preserve
those guarantees and have a path the affected clients can actually use.

## How would I notice this failure in a deployed app?

In the experiment, the exception is visible. For a user, the same event might just look like
"couldn't connect." The API can stay healthy and serving other clients while a specific app
version starts rejecting its certificate.

That's why looking only at server availability isn't enough to interpret the incident. I need
to correlate the client version, the policy it received, and the key actually presented by the
TLS endpoint it connected to.

A change rolled out gradually across several servers can also produce intermittent failures:
the client accepts the key served by one of them and rejects another's. To investigate, I need
to know whether every serving path is presenting certificates compatible with the versions still
supported.

The telemetry channel deserves the same attention. If error events depend on the connection that
just failed, the absence of logs doesn't prove the clients are working. It might be exactly the
part of the connectivity loss I need to detect.

Those consequences broaden what it means to operate the control: knowing which keys should be
accepted, tracking how that policy is distributed, and being able to recognize when client and
server have stopped being compatible.

## After this investigation, when is pinning worth adopting?

The experiment shows that a legitimate rotation can be accepted by a prepared client and
rejected by another that only knows the previous key. The gain from restricting trust comes with
the need to keep that configuration compatible with the service.

That cost helps explain the warning in the Android documentation, which
[doesn't recommend certificate pinning for Android apps](https://developer.android.com/privacy-and-security/security-ssl#restricting-your-app-to-specific-certificates)
because of the risk that server changes leave clients without connectivity. When the restriction
is used, the document highlights backup pins and expiration among the operational precautions.

This risk isn't unique to Android: the web already went through it at real scale. HTTP Public
Key Pinning (HPKP), standardized in [RFC 7469](https://www.rfc-editor.org/rfc/rfc7469), was
deprecated by major browsers around 2018 after operators locked themselves out of their own
domains by configuring pins without a viable backup key. This article's experiment reproduces,
at lab scale, the same failure mechanism that led to that standard being abandoned.

To evaluate an adoption, I need to start from the concrete threat I'm trying to make harder.
Then, I need to check whether I control the keys, the certificates, the TLS termination, and the
distribution needed to keep the supported versions working.

The third article shows the limit against someone who controls the process or alters the app's
package. This last one adds the responsibility of keeping the control working through legitimate
changes. Both perspectives need to be part of the same decision.

Authentication, authorization, and backend validations are still necessary. An extra restriction
on the channel doesn't turn the client into a trustworthy environment, nor does it replace those
controls.

## For anyone who wants to reproduce the investigation

The A → B demonstration was run with two Android clients and certificates issued by a local CA.
The [optional reproduction guide](#TODO-link-github-reproducao-rotation)
gathers the commands, variant names, setup, and expected results. None of these steps are
necessary to follow this article's explanation.

The [new evidence](#TODO-link-github-evidence-article-04-readme) includes the screenshots, the
public certificates, and the logs from the reissue with the same key A and from the A → B
rotation. The discussions about C, emergencies, expiration, and observability develop their
operational consequences without representing additional production tests.

## What closes out the series

I start the series by investigating how a client decides to trust a domain's certificate. Then,
I separate the choice of CAs from the extra restriction of pins and follow the points where that
restriction can be applied and altered.

In this final investigation, the question becomes how to keep that trust during a legitimate
change to the service. The client that knows A and B stays connected because the alternative
arrived before it was needed. The client that only knows A loses the connection, even though the
server keeps presenting a valid certificate.

The mental model I end up with is:

- *The certificate can change without the key changing.*
- *If the key changes, the client needs to recognize a compatible alternative.*
- *The backup only helps whoever received that pin before the rotation.*
- *The next alternative needs to arrive before the server requires it.*

The pin distributed today becomes part of the conditions for changing the service tomorrow.
Justifying the adoption of pinning requires understanding the threat it makes harder, the
attacker who can still alter it, and the maintenance needed to keep your own users connected.

## Quick reference for the terms used

| Term | Meaning in this article |
| --- | --- |
| TLS (Transport Layer Security) | Protocol used to protect and authenticate the connection |
| CA (Certificate Authority) | Certificate authority accepted during chain validation |
| Leaf | Final certificate presented to represent the server |
| SPKI (Subject Public Key Info) | Public key structure whose hash is used in the pins |
| Fingerprint | Hash of the entire certificate |
| NSC (Network Security Config) | Android's declarative network configuration mechanism |
| Backup pin | Pin for an alternative key prepared before a rotation |
| Rotation A → B | Replacing key A with key B on the server |
| Expiration | Date after which the NSC stops enforcing that pin set |
| Rollback | Returning to a prior state while it's still acceptable |
