+++
title = "How to Identify and Bypass Certificate Pinning on Android"
translationKey = "android-certificate-pinning-identification-bypass"
date = "2026-11-11"
author = "Thiago Siqueira"
tags = ["android", "ssl-pinning", "mobile", "frida", "reverse-engineering"]
keywords = ["android", "certificate pinning", "frida", "apktool", "jadx", "okhttp"]
description = "I compare two certificate pinning implementations on Android, Network Security Config and OkHttp's CertificatePinner, to show how static analysis and dynamic instrumentation reveal different bypass points."
showFullContent = false
readingTime = true
hideComments = false
+++

In the previous article, I proved two important things:

1. an Android app can reject the Burp CA before any pinning even exists;
2. once the proxy's chain is already accepted, a `<pin-set>` can still block the connection because the SPKI presented by Burp doesn't match what the app expects.

That resolves a common confusion: not every TLS failure on Android is pinning.

But it leaves a new question:

*If two apps fail due to pinning, does that mean the check lives in the same place?*

That question matters because the visible symptom can be nearly identical: `FAILURE`, with an exception while establishing the secure connection.

Even so, the analysis and bypass strategy can change completely.

In my lab, I built two flavors that reject Burp due to pinning:

- `nscPinned`, which uses Network Security Config;
- `okhttpPinned`, which uses the OkHttp library's `CertificatePinner`.

Both call the same API, use the same pins, and fail against the same proxy.

One declares the policy in XML, for the platform to enforce. The other configures the check in the HTTP client's code.

That difference is what I want to investigate in this article:

*When the connection is rejected, where in the application is that policy being enforced?*

I didn't introduce OkHttp in the previous article because the question there was different: I wanted to separate trust store from pinning using Android's native policy. Now that distinction is already proven, I can add a second implementation without mixing the concepts.

## Before starting: what stays the same in the lab

The commands in this article assume the project root, with the APKs already built and installed, the local API running, and both the lab's and Burp's CAs installed in Android's user store.

I use an Android API 36 emulator with root, OkHttp **4.12.0**, Frida and `frida-server` **17.17.0**, frida-tools **14.10.4**, apktool **3.0.3**, and JADX **1.5.5**. `frida-server` is already running with root privileges; it's the on-device component that lets the Frida client instrument the process. Having the proxy working, by itself, doesn't provide that access.

The setup is described in [Lab environment](#TODO-link-github-docs-00-environment). Before running the commands, I load the Android SDK and Java paths:

```bash
source scripts/lab-env.sh
```

I keep the same environment as the previous article: **Android Emulator → Burp Suite on macOS → local HTTPS API**.

The API still exposes `https://api.ssl-pinning.127.0.0.1.nip.io:8443/profile`.

The proxy is still configured on Android:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
```

And the bridge from the emulator to the local API is still active:

```bash
adb reverse tcp:8443 tcp:8443
```

I'm also still using the same app with multiple flavors. They share the screen and the endpoint. The two pinning variants compared here, `nscPinned` and `okhttpPinned`, enforce the same pins:

- `sha256/Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=`
- `sha256/jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=`

That matters because it lets me compare only the implementation point.

The infrastructure stays the same; what changes is the policy's implementation point.

## Before comparing: the baseline still passes

Before comparing two failures, I quickly revalidate the baseline from the previous article.

The `proxyTrusted` flavor accepts the Burp CA and has no pinning:

```bash
adb shell am force-stop com.example.sslpinninglab.proxytrusted
adb shell am start \
  -n com.example.sslpinninglab.proxytrusted/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

The result is still `SUCCESS`, with `HTTP 200`.

That matters because it keeps the same conclusion as the previous article: *if another flavor fails now, the cause is no longer simply "Burp isn't working."*

## The first piece of evidence: two flavors fail against the same Burp

I start by repeating the `nscPinned` failure, which already appeared in the previous article:

```bash
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

The result is `FAILURE`, with `Exception: SSLHandshakeException` and root cause `CertificateException: Pin verification failed`.

To follow the exceptions emitted by the app, I open another terminal before the request:

```bash
adb logcat -v threadtime -s 'SslPinningLab:E' '*:S'
```

`SslPinningLab` is the tag the lab's code uses to log the full exception. The filter selects its errors and silences every other tag. It can include old records; to correlate the failure, I watch the timestamp of the new request. I close the reader with `Ctrl+C`.

The relevant excerpt shows `Caused by: java.security.cert.CertificateException: Pin verification failed`, inside `NetworkSecurityTrustManager.checkPins(...)`.

Then I open `okhttpPinned`:

```bash
adb shell am force-stop com.example.sslpinninglab.okhttppinned
adb shell am start \
  -n com.example.sslpinninglab.okhttppinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

On screen, the app also fails, with `FAILURE`, `Exception: SSLPeerUnverifiedException`, and root cause `SSLPeerUnverifiedException: Certificate pinning failure!`.

Logcat confirms `javax.net.ssl.SSLPeerUnverifiedException: Certificate pinning failure!`.

At first glance, both variants seem to tell the same story:

- Burp presents a different public key;
- the app expected the legitimate SPKIs;
- the application's HTTP request never gets sent through the TLS tunnel.

That layer distinction matters: the proxy receives an HTTP `CONNECT` to open the tunnel before TLS validation even happens. What's blocked here is the application's `GET /profile`.

In OkHttp 4.12.0, `CertificatePinner` is called after the socket handshake and hostname verification, before the connection is released for the application's request. NSC, on the other hand, enforces pins during the platform's trust validation. So failing to establish a secure connection doesn't mean both rejections happen at the same stage.

The exceptions give a hint that the implementation changed:

| Variant | Main exception | Visible point in the stack trace |
| --- | --- | --- |
| `nscPinned` | `SSLHandshakeException`, caused by `CertificateException` | `NetworkSecurityTrustManager.checkPins(...)` |
| `okhttpPinned` | `SSLPeerUnverifiedException` | `CertificatePinner.check$okhttp(...)` |

The logical problem looks the same.

The decision point still needs to be confirmed.

This is the first important idea of the article:

*The symptom can be the same, but the implementation changes the investigation.*

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/pinning-failures-comparison.png" alt="Comparison of the nscPinned and okhttpPinned flavors failing against Burp through different mechanisms" class="center" width="820">

*Both flavors fail against the same Burp, but the message already suggests the decision is being made at different points.*

At this point, I still haven't treated the exception as final proof.

A handshake failure is a clue. It helps me decide where to look, but it doesn't replace inspecting the APK.

| Evidence | What it suggests | What I still need to confirm |
| --- | --- | --- |
| Failure going through Burp | The connection was rejected before the app's normal HTTP | Which validation made the decision |
| `NetworkSecurityTrustManager.checkPins(...)` in the stack trace | The policy may come from Network Security Config | Whether the XML really contains a `<pin-set>` |
| `CertificatePinner.check$okhttp(...)` in the stack trace | The OkHttp library may be enforcing pins | Whether that call takes part in the APK's real flow |

## Question 1: where do I look first inside the APK?

When I have an APK and want to understand where pinning lives, I don't start by trying to run a universal bypass script.

I start by looking for static evidence.

For that, I use two tools with different roles:

| Tool | What I want from it |
| --- | --- |
| `apktool` | The APK's decoded manifest, resource XML, and smali |
| `jadx` | An approximate Java/Kotlin view of the bytecode |

I organize the files generated per experiment inside `/tmp/ssl-pinning-lab`:

| Directory | Content |
| --- | --- |
| `/tmp/ssl-pinning-lab/static-analysis` | Resources decoded by apktool and code exported by JADX |
| `/tmp/ssl-pinning-lab/nsc-patch` | Editable copy of the NSC APK and rebuilt APKs |

These directories are local working areas, separate from the sources in `lab/`. The names identify the experiment and can be used both in this article and when reproducing the lab. Since they live under `/tmp`, the system can remove them; I don't keep there the only copy of any evidence I want to preserve.

I decode the lab's three main APKs:

```bash
mkdir -p /tmp/ssl-pinning-lab/static-analysis/framework

apktool d -f \
  -p /tmp/ssl-pinning-lab/static-analysis/framework \
  lab/android-app/app/build/outputs/apk/proxyTrusted/debug/app-proxyTrusted-debug.apk \
  -o /tmp/ssl-pinning-lab/static-analysis/apktool/proxyTrusted

apktool d -f \
  -p /tmp/ssl-pinning-lab/static-analysis/framework \
  lab/android-app/app/build/outputs/apk/nscPinned/debug/app-nscPinned-debug.apk \
  -o /tmp/ssl-pinning-lab/static-analysis/apktool/nscPinned

apktool d -f \
  -p /tmp/ssl-pinning-lab/static-analysis/framework \
  lab/android-app/app/build/outputs/apk/okhttpPinned/debug/app-okhttpPinned-debug.apk \
  -o /tmp/ssl-pinning-lab/static-analysis/apktool/okhttpPinned
```

The `apktool d` command decodes the APK.

It doesn't recover the original project.

What it hands me is a reconstructed representation of the resources and bytecode, enough to follow references like the manifest, XML, and smali.

The most important parameters are:

| Snippet | Purpose |
| --- | --- |
| `-f` | Overwrites a previous extraction |
| `-p /tmp/.../framework` | Keeps apktool's frameworks in an isolated folder |
| `-o /tmp/...` | Sets the directory where the decoded resources get written |

Then I open the OkHttp variant in jadx:

```bash
jadx -d /tmp/ssl-pinning-lab/static-analysis/jadx/okhttpPinned \
  lab/android-app/app/build/outputs/apk/okhttpPinned/debug/app-okhttpPinned-debug.apk
```

To locate the NSC reference in the decoded manifests, I use:

```bash
rg -n 'networkSecurityConfig' /tmp/ssl-pinning-lab/static-analysis/apktool/*/AndroidManifest.xml
```

The search points to the following attribute in the `AndroidManifest.xml` of all three variants:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ... />
```

That still doesn't prove pinning.

It only proves there's an active Network Security Config worth reading.

Finding a file called `network_security_config.xml` by itself wouldn't be enough. What matters is confirming the manifest actually points to it.

## Question 2: what does the XML reveal about `nscPinned`?

I follow the manifest's reference to the decoded resource:

```bash
cat /tmp/ssl-pinning-lab/static-analysis/apktool/proxyTrusted/res/xml/network_security_config.xml
cat /tmp/ssl-pinning-lab/static-analysis/apktool/nscPinned/res/xml/network_security_config.xml
```

In the `proxyTrusted` baseline, the XML extracted by apktool shows:

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

There's no `<pin-set>`.

This variant accepts the Burp CA, but doesn't require a specific SPKI.

In `nscPinned`, the XML is almost the same, but it gains a `domain-config` with pins:

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

In this case, the APK tells me almost everything without my having to open a single Java class:

1. the app accepts both system and user-store CAs;
2. the API's domain is explicitly covered by pins;
3. the hashes appear directly in the XML;
4. the blocking point is a declarative Android policy.

That also explains why the stack trace pointed to `NetworkSecurityTrustManager.checkPins(...)`.

It's not a class written by the app that enforces the decision.

It's the Android platform itself, reading the policy from the XML.

At this point, the most natural bypass already starts to show up: *if the pin lives in the XML, removing the pin from the XML is a reasonable hypothesis.*

## Question 3: what if the XML has no `<pin-set>`?

Now I open `okhttpPinned`'s XML:

```bash
cat /tmp/ssl-pinning-lab/static-analysis/apktool/okhttpPinned/res/xml/network_security_config.xml
```

It's the same as the baseline:

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

There's no `<pin-set>`.

If I stopped here, I'd draw the wrong conclusion: *this app has no pinning.*

But I already saw the app fail with `SSLPeerUnverifiedException: Certificate pinning failure!`.

That tells me the decision lives somewhere else.

In the files exported by JADX, I look both for where the pins are built and where they're attached to the client:

```bash
rg -n 'CertificatePinner|certificatePinner' /tmp/ssl-pinning-lab/static-analysis/jadx/okhttpPinned/sources
```

The search points to `MainActivity`. Finding the library's class bundled into the APK wouldn't be enough: I need to follow how it's used by the client that actually runs the request.

To inspect that snippet in a graphical interface, I can also open the APK directly:

```bash
jadx-gui lab/android-app/app/build/outputs/apk/okhttpPinned/debug/app-okhttpPinned-debug.apk
```

In the code tree, I open `com.example.sslpinninglab.MainActivity` and locate `requestWithOkHttpPinning`. The relevant snippet, formatted for readability, is:

```java
CertificatePinner certificatePinner = new CertificatePinner.Builder()
        .add(BuildConfig.API_HOST, BuildConfig.LEAF_PIN, BuildConfig.BACKUP_PIN)
        .build();

OkHttpClient client = new OkHttpClient.Builder()
        .certificatePinner(certificatePinner)
        .build();
```

Now the difference is clear:

| Case | Where the pin lives | Who enforces the validation |
| --- | --- | --- |
| `nscPinned` | `res/xml/network_security_config.xml` | Android / Network Security Config |
| `okhttpPinned` | Java/Kotlin code | `okhttp3.CertificatePinner` |

The Network Security Config's `<pin-set>` declares the pins in XML; NSC also configures other network policies and can exist without pinning. `CertificatePinner`, on the other hand, configures pins programmatically.

In both cases, the app is asking for the same thing: *accept only a chain that contains an expected SPKI.*

But the way that rule enters the APK is different.

And that difference changes the bypass.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/jadx-okhttp-certificate-pinner.png" alt="JADX showing the CertificatePinner being built inside MainActivity" class="center" width="820">

*In JADX, the pinning policy shows up in the app's own code: `CertificatePinner.Builder().add(...)` assembles the pins and the OkHttp client receives that configuration.*

## Question 4: why don't I start with a universal script?

After finding `CertificatePinner`, it would be tempting to download a ready-made script and hope it works.

Sometimes it does.

But that doesn't teach where the decision happens.

In my lab, I want to prove the smallest change needed. So I start from the stack trace.

The OkHttp flavor's failure shows `SSLPeerUnverifiedException: Certificate pinning failure!`, inside `okhttp3.CertificatePinner.check$okhttp(...)`.

That detail matters, but not because `check$okhttp` is some universal API.

It's the method that shows up in this version of OkHttp and in this lab's execution path. In another version, another build, or an obfuscated app, the name or signature could change.

`CertificatePinner` has more than one method related to this check:

- `check(java.lang.String, java.util.List)`
- `check(java.lang.String, java.security.cert.Certificate[])`
- `check$okhttp(java.lang.String, kotlin.jvm.functions.Function0)`

The stack trace already indicates which method took part in the failure. Now I want to confirm it's available in the process with the signature I plan to intercept. Enumerating methods confirms they exist; it doesn't, by itself, show they were executed.

So I write an inspection script:

```javascript
Java.perform(function () {
  const CertificatePinner = Java.use("okhttp3.CertificatePinner");
  const methods = CertificatePinner.class.getDeclaredMethods();

  console.log("[*] okhttp3.CertificatePinner declared methods:");
  for (let i = 0; i < methods.length; i += 1) {
    console.log("    " + methods[i].toString());
  }
});
```

This script only inspects the methods. It answers the question: *which methods does this class actually expose in the running process?*

I run it:

```bash
frida -U -n 'SSL Lab - OkHttp Pinned' -l lab/frida/inspect-okhttp.js -q
```

The parameters mean:

| Snippet | Purpose |
| --- | --- |
| `-U` | Connects to the USB device, in this case the emulator |
| `-n 'SSL Lab - OkHttp Pinned'` | Attaches to the already-open process by name |
| `-l lab/frida/inspect-okhttp.js` | Loads the JavaScript script |
| `-q` | Runs in quiet mode, keeping the output cleaner |

The output shows `[*] okhttp3.CertificatePinner declared methods:`, followed by:

- `public final void okhttp3.CertificatePinner.check(java.lang.String,java.util.List)`
- `public final void okhttp3.CertificatePinner.check(java.lang.String,java.security.cert.Certificate[])`
- `public final void okhttp3.CertificatePinner.check$okhttp(java.lang.String,kotlin.jvm.functions.Function0)`

Now I have two pieces of evidence pointing to the same place:

1. the failure's stack trace goes through `check$okhttp`;
2. the runtime confirms that method exists with the expected signature.

With the failure located and the signature confirmed, I can prepare the targeted hook.

The script also stays in the lab at `lab/frida/inspect-okhttp.js`.

That way, the screenshot's output doesn't become the only reproducible evidence of the investigation.

## Question 5: what's the smallest hook that proves the hypothesis?

Before the hook, one precondition matters: in this flavor, the Burp CA is already accepted by the app's trust policy.

If that weren't true, removing just the pinning wouldn't be enough.

The hook I'm about to test doesn't turn an invalid chain into a valid one. It only blocks the extra SPKI check done by `CertificatePinner`.

If `check$okhttp` is the point where the library rejects the presented SPKI, I don't need to rewrite the whole TLS stack.

I just need to neutralize that decision.

The hook looks like this:

```javascript
Java.perform(function () {
  const CertificatePinner = Java.use("okhttp3.CertificatePinner");

  const checkOkHttp = CertificatePinner["check$okhttp"].overload(
    "java.lang.String",
    "kotlin.jvm.functions.Function0",
  );

  checkOkHttp.implementation = function (hostname, peerCertificatesFn) {
    console.log("[+] Bypassing CertificatePinner.check$okhttp for " + hostname);
    return;
  };

  console.log("[*] Targeted OkHttp CertificatePinner hook installed");
});
```

To understand why this hook works, I need to look at the original method's contract. `check$okhttp` doesn't return a boolean saying whether the pins were accepted. It returns normally when the check allows the connection to continue and throws `SSLPeerUnverifiedException` when it rejects the pins.

The second parameter, `Function0`, is a Kotlin function with no arguments that supplies the list of certificates for the check. By replacing the entire method, my hook never calls that function and never runs the original comparison.

The `return;` ends a method whose Java return type is `void`. For the caller, the invocation finishes normally, without the exception that would interrupt the connection. That's what neutralizes the restriction in this flow; there's no new certificate validation inside the hook.

The script does four things:

1. waits for the Java runtime and runs the instrumentation function with `Java.perform`;
2. gets the `okhttp3.CertificatePinner` class;
3. selects exactly the overload observed at runtime;
4. replaces the implementation with an empty return.

I attach the hook:

```bash
frida -U -n 'SSL Lab - OkHttp Pinned' -l lab/frida/targeted-okhttp-hook.js
```

This second script is also versioned in the lab at `lab/frida/targeted-okhttp-hook.js`.

I keep that session open and wait for confirmation that the hook was installed: `[*] Targeted OkHttp CertificatePinner hook installed`.

Only then do I tap **REQUEST PROFILE** again, on the screen of the already-instrumented app. That triggers a new call without ending the process Frida is attached to. I don't use `force-stop` at this point, because that would also end the instrumented process.

Frida shows `[*] Targeted OkHttp CertificatePinner hook installed` again and, right after, `[+] Bypassing CertificatePinner.check$okhttp for api.ssl-pinning.127.0.0.1.nip.io`.

And the app's state changes to `SUCCESS`, with `HTTP 200`.

The payload shows up on screen again.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/okhttp-after-hook.png" alt="okhttpPinned flavor working through Burp after hooking CertificatePinner" class="center" width="420">

*After the hook, the same flavor responds again with `HTTP 200` and the API's payload.*

I also check Burp's HTTP history. The entry shows `GET /profile`, `User-Agent: okhttp/4.12.0`, and the response `HTTP/1.1 200 OK`. The JSON's `time` field, `2026-11-11T22:57:50.385Z`, matches the payload shown by the app after the hook. That correlation ties the instrumented run to the request and response readable in the proxy.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/burp-okhttp-after-hook.png" alt="Burp's history showing GET /profile with User-Agent okhttp/4.12.0 and an HTTP 200 response whose time matches the success after the hook" class="center" width="1000">

*Entry 2856 in Burp logs the OkHttp call after the hook. The time in the JSON is the same one observed in the app.*

I removed the extra pinning check done by the library. The handshake still happens, and the Burp CA is still accepted by the app's trust policy; what changed was the local decision that compared the SPKIs.

This confirms `CertificatePinner`'s role in this specific flow. Failures in other TLS checks still require their own investigation.

### What happens when I restart without the hook?

To check whether the change stayed limited to the process, I close the Frida session with `Ctrl+C` and stop the app. Then I open the same APK again, without attaching the script:

```bash
adb shell am force-stop com.example.sslpinninglab.okhttppinned
adb shell am start \
  -n com.example.sslpinninglab.okhttppinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

The result goes back to `FAILURE`, with `SSLPeerUnverifiedException: Certificate pinning failure!`.

The proxy and the APK stay the same. Once the instrumented process ends, the original implementation is used again on the next run. The sequence observed is:

| State | Result |
| --- | --- |
| `proxyTrusted`, no pinning, via Burp | `SUCCESS`, HTTP 200 |
| `okhttpPinned`, before the hook | `FAILURE`, `Certificate pinning failure!` |
| `okhttpPinned`, with the hook running | `SUCCESS`, HTTP 200 |
| Same APK restarted without the hook | `FAILURE`, `Certificate pinning failure!` |

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/okhttp-restarted-without-hook.png" alt="okhttpPinned flavor rejecting Burp's pins again after restarting without instrumentation" class="center" width="420">

*Restarting without the hook restores the rejection. The earlier instrumentation didn't change the installed APK.*

The screens, the XMLs, and the timeline of this validation are in the [supplementary evidence](#TODO-link-github-evidence-article-03-review-readme). The Frida log records the hook's execution for the API's hostname; the response shown by the app contains the time of the new request. The capture of Burp's history lets me correlate that same payload with the HTTP traffic observed in the proxy.

## Question 6: does `nscPinned`'s bypass follow the same logic?

The logic is still to neutralize the extra restriction, but I choose a different intervention point.

In `nscPinned`, the policy is declared in XML and enforced by Android platform code, as shown by `NetworkSecurityTrustManager.checkPins`. For this experiment, the most didactic bypass is to remove the `<pin-set>` before running the app.

That doesn't mean NSC can only be bypassed by patching, or that OkHttp can only be bypassed by hooking. It would also be possible to investigate the platform's validation at runtime, or to alter the code that configures OkHttp inside the APK. Here I choose two interventions to compare an in-memory change with a change to the artifact.

I alter the artifact, rebuild the APK, and only then run the app again.

I set up a separate area from the static analysis and decode the APK:

```bash
mkdir -p /tmp/ssl-pinning-lab/nsc-patch/framework

apktool d -f \
  -p /tmp/ssl-pinning-lab/nsc-patch/framework \
  lab/android-app/app/build/outputs/apk/nscPinned/debug/app-nscPinned-debug.apk \
  -o /tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded
```

The relevant file shows up at `/tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded/res/xml/network_security_config.xml`.

The original snippet contains:

```xml
<pin-set expiration="2035-01-01">
    <pin digest="SHA-256">Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=</pin>
    <pin digest="SHA-256">jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=</pin>
</pin-set>
```

I remove only that block and keep the rest of the policy unchanged. To check the scope of the edit, I compare the project's original XML with the resource I just modified:

```bash
diff -u \
  lab/android-app/app/src/nscPinned/res/xml/network_security_config.xml \
  /tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded/res/xml/network_security_config.xml
```

The comparison may include apktool formatting differences; the policy change I want to see is the removal of the `<pin-set>`, with the domains and trust anchors preserved.

In the lab APK I'm rebuilding, this tests the effect of stripping the policy from the modified artifact. It doesn't change the original APK that existed before the edit.

Then I rebuild the APK:

```bash
apktool b -p /tmp/ssl-pinning-lab/nsc-patch/framework \
  /tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded \
  -o /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched-unsigned.apk
```

The rebuilt APK still needs to be aligned and signed. Before running the commands, I separate each step's role:

| Command | What it does in this experiment |
| --- | --- |
| `zipalign` | Adjusts the alignment of the uncompressed data inside the ZIP that forms the APK, allowing efficient access to those files. Doesn't sign the package. |
| `apksigner sign` | Signs the rebuilt APK with the debug keystore's private key and includes the signer's certificate. The signature lets Android verify the package's integrity and identify the key that signed it. |
| `apksigner verify --print-certs` | Verifies the APK's signature and shows information about the signing certificate. Doesn't test the app's behavior or its TLS pinning. |

I run the steps in this order: alignment, signing, and verification.

```bash
zipalign -f -p 4 \
  /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched-unsigned.apk \
  /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched-aligned.apk

apksigner sign \
  --ks ~/.android/debug.keystore \
  --ks-key-alias androiddebugkey \
  --ks-pass pass:android \
  --key-pass pass:android \
  --out /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched.apk \
  /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched-aligned.apk

apksigner verify --print-certs \
  /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched.apk
```

Altering the signature-protected resource invalidates the previous signature; that's why the rebuilt APK needs to be signed again. I align before signing, because alignment modifies the package and, if done afterward, would invalidate the signature produced by `apksigner`. The final verification checks the result before installation. This order is described in the official references for [zipalign](https://developer.android.com/tools/zipalign) and [apksigner](https://developer.android.com/tools/apksigner).

The APK's signing certificate plays a different role than the API's TLS certificate: the first identifies the key used to sign the package; the second ties the server's public key to the domain in TLS authentication. Signing the altered APK doesn't change the API's certificate or the pins the app expects to find.

In the lab, the original build and the patched APK use the same debug keystore, `~/.android/debug.keystore`.

That's why `adb install -r` can replace the installed app without requiring an uninstall.

In a real app, a modified APK normally couldn't replace the official install, because the attacker wouldn't have the production signing private key. The bypass stays relevant when the precondition includes control over the install, the device, or the test environment.

Then I install the altered APK:

```bash
adb install -r /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched.apk
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

The result changes to `SUCCESS`, with `HTTP 200`.

Again, I didn't make Burp "more trusted."

I only removed the extra decision that rejected the SPKI presented by the proxy.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/nsc-patch-before-after.png" alt="Comparison of nscPinned before and after removing the pin-set from the APK" class="center" width="820">

*Before the patch, the declarative policy still rejects Burp's SPKI. After removing the `<pin-set>` and reinstalling the APK, the same request passes.*

## What comparing the two bypasses reveals

The lab's central difference comes down to:

| Variant | Where pinning lives | Static evidence | Dynamic evidence | Bypass demonstrated |
| --- | --- | --- | --- | --- |
| `nscPinned` | XML | `<pin-set>` in `network_security_config.xml` | `NetworkSecurityTrustManager.checkPins` | Remove the `<pin-set>` and rebuild the APK |
| `okhttpPinned` | Java/Kotlin code | `CertificatePinner.Builder().add(...)` | `CertificatePinner.check$okhttp` | Frida hook on the real overload |

That table answers the article's main question: *two apps can fail for the same logical reason and still offer different intervention points.*

The techniques used here depend on the implementation and the access available. Version changes, obfuscation, custom classes, or native validations may call for other intervention points. The pairing of XML with patching, or OkHttp with hooking, describes these experiments; it isn't an exclusive rule for analyzing any APK.

What stays valid across all these cases is the methodology:

1. identify the implementation;
2. locate the exact decision point;
3. choose the smallest bypass that proves the hypothesis;
4. validate whether the behavior changed.

## Sources to check the mechanisms

- [Android — Network Security Configuration](https://developer.android.com/privacy-and-security/security-config): trust and pinning policies declared in XML.
- [OkHttp 4.12.0 — CertificatePinner](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/CertificatePinner.kt): pin comparison, normal return, and the rejection exception.
- [OkHttp 4.12.0 — RealConnection](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/internal/connection/RealConnection.kt): order between handshake, hostname verification, and pinning.
- [Frida — JavaScript API](https://frida.re/docs/javascript-api/): using `Java.perform`, `Java.use`, and method replacement.

## What these bypasses prove

By the end of the experiment, I can prove:

1. `nscPinned` and `okhttpPinned` block the same MITM through different mechanisms;
2. static analysis is what reveals where the pinning rule lives;
3. the stack trace helps formulate the hypothesis about the real dynamic point of failure;
4. a targeted Frida hook can neutralize a library's decision at runtime;
5. an APK patch can remove a declarative policy before execution;
6. pinning raises the cost of a MITM, but isn't an absolute barrier against someone who controls the process or the APK.

## Final mental model

After this stage of the lab, the most useful way to think about bypass is:

- *The visible error tells me there's a failure.*
- *The stack trace helps me locate where to look.*
- *The APK confirms how it was implemented.*
- *The bypass needs to target exactly that point.*

Or, even shorter:

- *The implementation reveals where to intervene.*
- *The available access guides how to intervene.*
- *The observed change tests the hypothesis.*

The value isn't in memorizing a script.

The value is in observing the failure, locating the policy, confirming how it takes part in the flow, and only then choosing the test technique.

After proving the bypass depends on the implementation, the next question stops being "how do I remove pinning?" and becomes: *what happens when the server's legitimate key needs to change?*

That's the investigation I want to open in the next article.

## Quick reference for the terms used

| Term | Meaning |
| --- | --- |
| APK (Android Package) | Installable package of an Android application |
| apktool | Tool for decoding and rebuilding APKs |
| jadx | Decompiler that reconstructs a Java/Kotlin view of the bytecode |
| Network Security Config | Android XML that defines trust anchors, cleartext, and pins |
| `<pin-set>` | List of accepted SPKI pins for a domain |
| `CertificatePinner` | OkHttp class used to enforce programmatic pinning |
| Frida | Dynamic instrumentation toolkit |
| Hook | Replacing a behavior at runtime |
| Overload | A method variant with a different signature |
| `check$okhttp` | Internal OkHttp method observed in the lab's runtime |
| `SSLPeerUnverifiedException` | Exception for verifying the peer's identity; in this flow, the message and stack trace point to pin rejection |
| `NetworkSecurityTrustManager.checkPins` | Android platform method that enforces NSC pins |
| `zipalign` | Tool that aligns the APK before signing |
| `apksigner` | Tool that signs and verifies APKs |
| Debug keystore | Signing key used in Android debug builds |
