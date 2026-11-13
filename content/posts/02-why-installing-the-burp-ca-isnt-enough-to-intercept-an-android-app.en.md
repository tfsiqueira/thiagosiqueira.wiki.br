+++
title = "Why Installing the Burp CA Isn't Enough to Intercept an Android App"
translationKey = "android-burp-ca-trust-vs-pinning"
date = "2026-11-09"
author = "Thiago Siqueira"
tags = ["android", "ssl-pinning", "mobile", "burp", "network-security-config"]
keywords = ["android", "network security config", "certificate pinning", "burp suite", "trust store"]
description = "An Android lab with multiple flavors to separate, step by step, a common trust store failure from a real certificate pinning failure via Network Security Config, using Burp Suite as a MITM proxy."
showFullContent = false
readingTime = true
hideComments = false
+++

I already have the Burp CA (Certificate Authority) installed on the Android device.

Chrome can open the lab's API (Application Programming Interface).

Burp can intercept the HTTPS (Hypertext Transfer Protocol Secure) connection and show the request in its history.

So why does an Android app still fail?

That's the question that opens the second part of my SSL pinning lab.

In the previous article, I wanted to understand how a client validates a domain's authenticity during the TLS (Transport Layer Security) handshake. I walked through leaf, intermediate, trust anchor, SAN (Subject Alternative Name), SPKI (Subject Public Key Info), and cross-signing, arriving at an important mental model: *a TLS connection is only accepted when the client can build a path to a trust anchor it already trusts locally.*

That model already answers which CAs the app trusts. But there's an additional question it doesn't cover: once it trusts the CA, does the app trust any public key that CA signs? That's the question behind certificate pinning, also called SSL pinning: the technique by which an application, in addition to validating the chain up to a trusted CA, also requires the public key presented by the server to match a specific set of expected keys, the pins. That's the technique I want to investigate for the rest of the series, and this second article starts by isolating the part that gets confused most often: separating a common trust store failure from a real pinning failure.

To answer that without depending on a real service, I built a small, controlled lab.

## Before starting: the experiment's lab

I didn't want to study pinning against a real API, because that would mix my experiment with certificates, keys, rotation, and the availability of a service I don't control.

So I built a small, disposable HTTPS API. It exists only to serve as a predictable target for the lab:

- I control the hostname;
- I control the leaf certificate, that is, the final certificate presented by the server;
- I control the Root CA that signs that certificate;
- I can compute exactly the SPKI, that is, the public key structure whose hash the app must accept;
- I can change the key later, whenever I want to study rotation.

It exposes a `/health` endpoint, used to validate connectivity and TLS, and a `/profile` endpoint, queried by the app to receive a fictional payload.

I also built a minimal Android app. It has a single screen, a button to trigger the request, and an area where I can see `SUCCESS`, `FAILURE`, the HTTP status, or the captured exception. It doesn't try to simulate a real business app; it exists to make visible at which stage the connection is accepted or rejected.

The most important point is that I didn't create several apps with different logic. I created the same app with several flavors, that is, build variants. All of them use the same `MainActivity`, call the same `/profile` URL, and show the same interface. What changes between them is only the trust policy. That mattered because it let me compare one variable at a time. If two flavors call the same API, through the same path, and only one fails, the difference isn't in the backend or the network: it's in that variant's trust policy.

The lab's architecture ended up like this: **Android Emulator → (HTTPS) → Burp Suite on macOS → (HTTPS) → local API at 127.0.0.1:8443**.

The Android Emulator represents the device under analysis. Burp Suite runs on macOS and sits in the middle of the connection as a MITM (Man-in-the-Middle) proxy. The local API also runs on macOS, but on a different port, serving the certificate and the payload I control.

The connectivity endpoint lived at `https://api.ssl-pinning.127.0.0.1.nip.io:8443/health`.

In practice, I alternated between two paths:

- **Android Emulator → local API**
- **Android Emulator → Burp Suite → local API**

## The first piece of evidence

With the lab set up, I want to reproduce a situation common in mobile analysis:

1. configure Burp Suite as a proxy;
2. install the Burp CA on Android;
3. confirm that the browser accepts the connection;
4. open an Android app;
5. observe that it still fails;
6. figure out whether the failure comes from the trust store or from certificate pinning.

The thread running through the article becomes:

- *What is the app rejecting?*
- *A CA it doesn't trust?*
- *Or a public key that doesn't match the expected pin?*

Before testing any app, I validate both paths in Android's Chrome.

Without a proxy:

```bash
adb shell settings put global http_proxy :0
adb shell am start -a android.intent.action.VIEW \
  -d 'https://api.ssl-pinning.127.0.0.1.nip.io:8443/health?mode=android-direct'
```

With a proxy:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
adb shell am start -a android.intent.action.VIEW \
  -d 'https://api.ssl-pinning.127.0.0.1.nip.io:8443/health?mode=android-burp'
```

The commands above do two different things:

| Fragment | Function |
| --- | --- |
| `settings put global http_proxy :0` | Removes Android's global proxy |
| `settings put global http_proxy 10.0.2.2:8080` | Makes the emulator send traffic to Burp on the host |
| `10.0.2.2` | Special alias that, inside the emulator, points to the macOS host |
| `am start -a android.intent.action.VIEW -d <url>` | Asks Android to open the URL in the browser |

I use the hostname `127.0.0.1.nip.io` because I need the same DNS name to work both in the emulator and in Burp, while keeping TLS hostname validation intact.

On Android, the bridge to the local API depends on a port forwarded from the host: `adb reverse tcp:8443 tcp:8443` redirects port 8443 on the emulator to the same port on macOS, where the local API is listening.

In both cases, Chrome shows the API's health JSON:

```json
{
  "status": "ok",
  "service": "ssl-pinning-lab-api",
  "time": "..."
}
```

In the second case, the request also shows up in Burp's HTTP history.

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/chrome-and-burp-health.png" alt="Chrome opening the health endpoint while Burp logs the same request" class="center" width="820">

*Chrome accepts the intercepted connection and the request shows up in Burp's history.*

To interpret this test, there's an important detail: in the set of certificates installed by the user on Android, which I'll later call the user store, I had already installed two different CAs:

- the lab's Root CA, which signs the certificate served directly by the API;
- the Burp CA, which signs the certificates generated dynamically during the MITM.

When I access the API directly, Android needs to trust the lab's Root CA. When I go through Burp, Android no longer sees the API's original certificate and instead sees a certificate issued by the Burp CA.

This proves five things:

1. the API is reachable;
2. the hostname is correct;
3. Chrome accepts the lab's Root CA on the direct path;
4. Chrome accepts the Burp CA on the intercepted path;
5. the proxy can intercept Chrome's HTTPS on the device.

But that still only proves Chrome's behavior and the infrastructure's. The Android app can still make a different trust decision.

So I move on to the app.

## Before testing the flavors: where does the trust policy come in?

Since all the flavors use the same `MainActivity` and call the same `/profile` URL, I need to find where the trust policy actually enters the app.

Before running the first variant, I look for that declaration in `AndroidManifest.xml`. That's where I find the reference to a file called `network_security_config`:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

Network Security Config, or NSC, is Android's declarative mechanism for defining trust anchors, cleartext rules, pins, and debug exceptions.

The manifest is common to all flavors. What changes per build is the resource that occupies that path: `app/src/<flavor>/res/xml/network_security_config.xml`.

In other words, the Java code keeps opening the same URL, but Android starts consulting a different trust policy depending on which flavor is installed.

Chrome uses `/health` only to validate the infrastructure. From here on, the app's flavors call `/profile`, always through the same path, changing only the trust XML.

That sets up the comparison I make from here on:

- same screen;
- same URL;
- same Burp;
- same API;
- **different trust XML**.

## Question 1: what happens with a flavor that only trusts system CAs?

I start with the `systemOnly` variant.

It uses `HttpsURLConnection`, the platform's native HTTPS client in Java/Android, and a minimal network configuration:

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

The name already gives it away: this flavor only trusts system CAs.

I run it with Burp still set as the proxy:

```bash
adb shell am force-stop com.example.sslpinninglab.systemonly
adb shell am start \
  -n com.example.sslpinninglab.systemonly/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

This command pattern repeats across the next flavors:

| Fragment | Function |
| --- | --- |
| `am force-stop <package>` | Kills the previous instance so the test starts clean |
| `am start -n <package>/<activity>` | Opens exactly the desired flavor |
| `--ez auto_request true` | Sends a boolean extra that makes the app fire the request on open |

The screen shows `FAILURE`, with `Exception: SSLHandshakeException` and root cause `CertPathValidatorException: Trust anchor for certification path not found.`

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/system-only-burp-failure.png" alt="systemOnly flavor failing through Burp with CertPathValidatorException" class="center" width="420">

*`systemOnly` fails through Burp before any pin comparison even happens.*

Logcat, that is, Android's log stream, confirms the same cause: `Caused by: java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.`

At this point, the first important conclusion shows up: *this failure still isn't pinning.*

The app doesn't even get to comparing SPKIs.

It fails earlier, because the chain presented by Burp doesn't end at a CA accepted by that variant.

In fact, `systemOnly` also fails without Burp. The local API's own Root CA is also installed in the user store, and this flavor doesn't accept any CA from there. That reinforces the point: `systemOnly` doesn't exist to prove pinning; it exists to prove that an app can reject a connection before any pin even exists.

## Question 2: why doesn't the Burp CA installed on Android enter the validation?

The Burp CA is installed on Android, but it lives in the user store, that is, in the set of certificates added by the user.

That detail matters because modern Android apps don't trust user-added CAs by default.

For apps with `targetSdkVersion >= 24`, that is, apps that target Android 7.0 or higher, the default behavior becomes trusting system CAs and not automatically trusting CAs added by the user.

In the lab, `systemOnly` makes that decision explicit with `src="system"`. `proxyTrusted`, on the other hand, opts in the opposite way, deliberately adding `src="user"`.

That explains the apparent paradox:

| Client | What it accepts | Result |
| --- | --- | --- |
| Android Chrome | User-installed CA | Passes |
| `systemOnly` | Only system CAs | Fails |

The right question isn't *is the Burp CA installed?* The right question is *did this app decide to trust CAs added by the user?*

For `systemOnly`, the answer is no.

This is an important point in the lab because it avoids a common mistake: *seeing a handshake failure and immediately concluding there's pinning.*

In this case, the failure comes only from the trust anchor policy.

## Question 3: what changes when the app starts accepting user-added CAs?

To answer that, I run the `proxyTrusted` variant.

It calls the same endpoint, uses the same Burp, and keeps the same network flow. The only change is in the Network Security Config:

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" overridePins="false" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

Now the app accepts:

- system CAs;
- user-added CAs.

The `overridePins="false"` attribute makes explicit a decision that would already be the default outside `debug-overrides`. It says a CA coming from the user store can take part in normal chain validation, but doesn't get permission to bypass pins if any exist. In `proxyTrusted`, there's still no `<pin-set>`, so this detail doesn't change the result. It documents the intent I'll keep for the comparison with `nscPinned`; omitting the attribute here wouldn't change its default value.

I run:

```bash
adb shell am force-stop com.example.sslpinninglab.proxytrusted
adb shell am start \
  -n com.example.sslpinninglab.proxytrusted/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

The result is `SUCCESS`, with `HTTP 200`.

On screen, the app also shows the payload returned by the API:

```json
{
  "user": "alice.demo",
  "accountTier": "lab",
  "message": "Fictional data for SSL pinning study only"
}
```

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/proxy-trusted-burp-success.png" alt="proxyTrusted flavor working through Burp and returning HTTP 200" class="center" width="420">

*`proxyTrusted` trusts the Burp CA and is able to read the API's payload.*

Logcat also logs: `proxy-trusted request succeeded`.

This variant is the baseline I need.

It proves that:

1. Burp is working;
2. the Burp CA is trusted by that app;
3. the route to the API is correct;
4. the TLS handshake can finish successfully;
5. any later failure in another variant can no longer simply be blamed on the proxy.

It also passes on the direct path, because `src="user"` accepts the API's local Root CA. In other words, `proxyTrusted` is the first flavor where both the direct path and the path through Burp work.

That's `proxyTrusted`'s most important role in the lab.

It doesn't exist to show a "vulnerable" app.

It exists to prove that the interception infrastructure is correct before talking about pinning.

## Question 4: what happens when I add a `<pin-set>` to the same trust base?

After validating `proxyTrusted`, I run the `nscPinned` variant to answer exactly that question.

It uses exactly the same trust base:

```xml
<trust-anchors>
    <certificates src="system" />
    <certificates src="user" overridePins="false" />
</trust-anchors>
```

But adds a `pin-set` for the API's domain:

```xml
<domain-config cleartextTrafficPermitted="false">
    <domain includeSubdomains="false">api.ssl-pinning.127.0.0.1.nip.io</domain>
    <domain includeSubdomains="false">api.ssl-pinning.10.0.2.2.nip.io</domain>
    <pin-set expiration="2035-01-01">
        <pin digest="SHA-256">Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=</pin>
        <pin digest="SHA-256">jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=</pin>
    </pin-set>
</domain-config>
```

These pins don't need to be memorized or invented by hand. They're the same SPKI hash I already learned to extract in the previous article, just in a different format. There I used:

```bash
openssl x509 -in cert.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256
```

to get the SPKI hash in hexadecimal. The Network Security Config expects that same hash in base64, so the command gains one extra step:

```bash
openssl x509 -in cert.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | openssl base64
```

The difference is only the encoding: `-binary` skips the hexadecimal output, and `openssl base64` converts those bytes into the format the `<pin digest="SHA-256">` attribute expects. The concept stays the same: it's the hash of the SPKI, not of the whole certificate, that goes into the pin.

The two values in the `<pin-set>` don't mean two keys are being used by the API at the same time. The first is the pin for the server's current key. The second is a backup pin, reserved for a future rotation. In this article, the important point is just that the certificate generated by Burp doesn't match either of them.

On the direct path, without the proxy, `nscPinned` passes. The local API presents exactly the key whose SPKI hash is in the XML. The test I want to observe is the path through Burp, where the chain can still be trusted, but the public key presented changes.

First I remove Android's global proxy and run the flavor directly against the API:

```bash
adb shell settings put global http_proxy :0
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

The direct result is `SUCCESS`, with `HTTP 200`.

Then I put Burp back as the proxy and repeat the exact same flavor:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

The result through Burp is `FAILURE`, with `Exception: SSLHandshakeException` and root cause `CertificateException: Pin verification failed`.

Logcat shows the exact point of failure: `Caused by: java.security.cert.CertificateException: Pin verification failed`, inside `NetworkSecurityTrustManager.checkPins(...)`.

Now the situation is different from `systemOnly`.

In `systemOnly`, Burp's chain doesn't even reach a trusted anchor.

In `nscPinned`, the Burp CA is already trusted.

The problem now is different: *the public key presented by Burp doesn't match the SPKIs the app expects.*

This is the point where pinning actually kicks in.

Without pinning, the app's question is: *does this chain end at a CA I trust?*

With pinning, the question becomes: *does this chain end at a CA I trust, and does it also contain an expected public key?*

The lab's central comparison ends up as:

| Variant | User CA trusted? | Pinning? | Direct | Via Burp | Cause of failure via Burp |
| --- | --- | --- | --- | --- | --- |
| `systemOnly` | No | No | Fails | Fails | Trust anchor not accepted |
| `proxyTrusted` | Yes | No | Passes | Passes | No failure |
| `nscPinned` | Yes | Yes | Passes | Fails | SPKI doesn't match |

That table summarizes the distinction I want to prove.

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/nsc-pinned-direct-vs-burp.png" alt="On the left, the nscPinned flavor passes on the direct request to the API; on the right, it fails when the request goes through Burp" class="center" width="820">

*On the left, `nscPinned` passes on the direct request to the API. On the right, the same flavor fails when the request goes through Burp and the presented SPKI changes.*

## Question 5: at what point in the flow does each failure happen?

Both failures show up as TLS problems to the app, but they don't mean the same thing.

In the explicit-proxy scenario, the app first establishes the TCP (Transmission Control Protocol) connection with Burp and requests a tunnel using the HTTP (Hypertext Transfer Protocol) `CONNECT` method. Only then does it negotiate TLS over that tunnel. So there's already proxy-control HTTP before TLS; what still hasn't been sent is the application's own request, such as `GET /profile`.

For the flavors that use `HttpsURLConnection` and the NSC pins in this experiment, the flow is: **TCP with the proxy → CONNECT → TLS handshake → chain and NSC pin validation → application's HTTP request inside the TLS tunnel.**

The point that matters for diagnosis is that TCP communication with the proxy and the tunnel can succeed while TLS validation rejects the connection. That blocks sending `GET /profile`, without meaning every prior exchange failed.

In `systemOnly`, the failure happens when Android tries to find a trust anchor: **TCP OK → TLS handshake starts → Burp's chain doesn't reach a trusted CA → handshake aborts → `GET /profile` is never sent through the TLS tunnel.**

In `nscPinned`, the chain is already trusted, but Android applies the extra pin check: **TCP OK → TLS handshake starts → Burp's chain reaches a trusted CA → `NetworkSecurityTrustManager.checkPins` rejects the SPKI → handshake aborts → `GET /profile` is never sent through the TLS tunnel.**

This difference matters because the visible symptoms can look alike: both show up as `FAILURE` with `SSLHandshakeException`. But the correct diagnosis depends on the root cause:

- in this case, `CertPathValidatorException` with the message `Trust anchor for certification path not found` → absence of an accepted trust anchor;
- `Pin verification failed` → real pinning failure.

## Question 6: does a debug build with `<pin-set>` also reject Burp?

After separating trust store from pinning, I want to test a situation that often confuses Android analyses: what happens when a debug build contains pins, but also declares a specific CA in `debug-overrides`?

The `debugOverrides` variant has a `pin-set`:

```xml
<pin-set expiration="2035-01-01">
    <pin digest="SHA-256">Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=</pin>
    <pin digest="SHA-256">jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=</pin>
</pin-set>
```

But it also has:

```xml
<debug-overrides>
    <trust-anchors>
        <certificates src="@raw/burp_ca" overridePins="true" />
    </trust-anchors>
</debug-overrides>
```

I run:

```bash
adb shell am force-stop com.example.sslpinninglab.debugoverrides
adb shell am start \
  -n com.example.sslpinninglab.debugoverrides/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

The result is `SUCCESS`, with `HTTP 200`.

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/debug-overrides-burp-success.png" alt="debugOverrides flavor working through Burp despite the pin-set" class="center" width="420">

*`debugOverrides` keeps passing through Burp because the debug CA can bypass the pins.*

Logcat confirms: `debug-overrides request succeeded`.

That doesn't mean pinning "accepts" Burp's certificate.

What's actually happening is:

1. the build is debuggable, that is, a build flagged for debugging;
2. the Burp CA is declared in `debug-overrides`;
3. the connection uses that debug CA;
4. `overridePins="true"` tells Android to ignore pins for that anchor.

This pin dispensation is the NSC's own. It doesn't guarantee that an extra check implemented in a library or in custom code will also be skipped.

One caveat is worth noting: this behavior depends on the build's `android:debuggable` attribute, not on the flavor's name itself. Any variant built as debuggable that declares `debug-overrides` would behave the same way, regardless of which flavor it represents. In the lab, `debugOverrides` just isolates that configuration to make it easy to observe.

In other words: *pinning doesn't lose; it's deliberately skipped in the debug configuration.*

This detail matters because an analysis done only on a debug build can lead to a wrong conclusion: *"the app accepts Burp, so it doesn't have pinning"*.

In reality, the release build can still be protected, because `debug-overrides` only takes part in the policy when the application is debuggable.

## What the experiment actually taught me

The difference between the flavors makes the diagnosis observable: `systemOnly` rejects the chain, `proxyTrusted` accepts it, and `nscPinned` adds a restriction that rejects the proxy's pins. `debugOverrides` shows a deliberate exception to NSC pin enforcement in a debug build.

So the same visible result, `FAILURE`, can hide different decisions. The root cause, the configuration, and the comparison against the baseline are what let me tell these decisions apart.

## A note for whoever implements pinning

The lab so far looks through the lens of the analyst. But whoever configures pinning on a real app will make some of these same decisions from the other side:

- `debug-overrides` with `overridePins="true"` is useful for development, but it's only safe if the `android:debuggable` flag really never leaks into the release build;
- on `src="user"`, spelling out `overridePins="false"` documents the intent, but that's already the default outside `debug-overrides`. Inside `debug-overrides`, the default is `true`. NSC pin bypass happens when the chain uses a source configured with `overridePins="true"`;
- a `<pin-set>` with a single pin is an operational risk: if the server's private key needs to rotate without warning, the app stops working. The backup pin exists to cover exactly that scenario.

These points will carry more practical weight once the series gets to pinning instrumentation and bypass, but they already stand as a checklist for reviewing an app's pinning configuration before that.

## Materials and references to reproduce this

The experiments depend on this lab's API and flavors. The supporting files include the [HTTPS API](#TODO-link-github-lab-api-readme), the [Android app](#TODO-link-github-lab-android-app-readme), the [environment setup](#TODO-link-github-readme-environment-setup), and the [complete reproduction walkthrough](#TODO-link-github-readme-full-walkthrough).

To build your own copy, generating new keys isn't enough: the pins in the XMLs need to track the SPKI of your keys, and the Burp CA declared in `debugOverrides` needs to be your proxy's. The hashes and captures in this text belong to the documented run; they aren't values any fresh install should copy without checking. For the flavors in this article, I change the pins declared in `<pin-set>` to match the generated keys and rebuild the APKs before comparing results.

To check the platform's behavior:

- [Android — Network Security Configuration](https://developer.android.com/privacy-and-security/security-config): anchors, pins, and debug configurations.
- [Android — overridePins attribute](https://developer.android.com/privacy-and-security/security-config#certificates): default values and pin dispensation per CA source.
- [Android — Android 7.0 changes](https://developer.android.com/about/versions/nougat/android-7.0-changes#tls-ssl): default trust in user-added CAs.
- [Android Emulator — networking](https://developer.android.com/studio/run/emulator-networking): what `10.0.2.2` means inside the emulator.

## Final mental model

After going through the experiment, I break down these responsibilities. It's a conceptual model for this lab, without imposing a universal order of execution among hostname, chain, and pins:

1. first the app decides which CAs to trust;
2. then Android validates hostname, validity, and chain;
3. if the chain reaches a trusted anchor, the connection can still face an extra check;
4. pinning compares the presented public key against the expected SPKIs;
5. a failure before that stage isn't pinning.

Or, even shorter: *trust store answers "which CAs do I trust?"; pinning answers "is this public key one of the expected ones?"*

To attribute this failure to pinning, I need to separate chain rejection from pin rejection. The baseline is what lets me isolate that difference in this experiment.

That doesn't mean an app that can't accept the proxy's CA is incapable of also having pinning. It can have both controls, with the first rejection keeping execution from ever reaching the pin check. The presence of pinning and the cause of the observed failure are different questions.

So far, I've only proven that pinning blocks a network MITM while the app keeps running its own trust policy untouched. I still haven't proven what happens when someone controls the app's process or alters the APK.

After proving that `nscPinned` fails due to real pinning, the next question stops being "why does the connection fail?" and becomes: *where does that decision actually live inside the APK (Android Package)?*

That's the investigation I want to open in the next article.

## Quick reference for the terms used

| Term | Meaning |
| --- | --- |
| Android Emulator | Emulator used to reproduce the Android device |
| Burp Suite | Interception proxy used to observe HTTPS traffic |
| MITM (Man-in-the-Middle) | Intermediation of the connection between client and server |
| CA (Certificate Authority) | Certificate authority that signs certificates |
| Trust store | Local set of trusted CAs available to the client |
| Trust anchor | Trusted certificate or key that ends a concrete validation |
| System store | Set of CAs trusted by the Android system |
| User store | Set of CAs added by the user on the device |
| User-added CA | CA manually installed by the user |
| Flavor | Build variant of the same Android application |
| `adb reverse` | Bridge that forwards a port from the emulator to the host |
| Network Security Config | Android XML that defines trust anchors, cleartext, and pins |
| APK (Android Package) | Installable package of an Android application |
| `src="system"` | Source of system CAs in the NSC |
| `src="user"` | Source of user-added CAs in the NSC |
| `<pin-set>` | List of accepted SPKI pins for a domain |
| SPKI (Subject Public Key Info) | Certificate structure that describes the public key |
| `debug-overrides` | Special trust configuration for debuggable builds |
| `overridePins` | Defines whether an anchor is allowed to bypass pins |
| `SSLHandshakeException` | Exception indicating a failure in the TLS negotiation |
| `CertPathValidatorException` | Exception for validating the chain up to a trust anchor |
