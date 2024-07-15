---
title: "Not so Secret: Analysis of KakaoTalk's Secret Chat E2EE Feature"
date: 2024-07-15T15:49:58+02:00
lastmod: 2024-07-15T15:49:58+02:00
author: "stulle123"
summary: "Finding flaws in KakaoTalk's end-to-end encryption feature Secret Chat."
tags: 
- Android
- mitmproxy
- Applied Cryptography
- E2E Encryption
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

In this blog post, I show a couple of weaknesses in KakaoTalk's end-to-end encryption (E2EE) feature *Secret Chat*.

One of the issues is a *well-known* MITM attack where an attacker who has full access to the server could replace public keys without immediate user notification.

This missing warning message in KakaoTalk allows a MITM attacker to secretly read end-to-end encrypted *Secret Chat* communications.

While most of the presented flaws here seem obvious, I argue that users should know about them to make an informed decision whether to trust KakaoTalk or not.

## TL;DR

Don't use *Secret Chat* if you're a high-risk user. Use a more robust E2EE chat app instead (e.g., Signal). Ideally, run your own messaging server if you can (e.g., [Signal server](https://github.com/signalapp/Signal-Server)).

If you're required to use *Secret Chat*, always compare the other party's public key fingerprint over a secure out-of-band channel. For example, meet up in person and scan each other's QR code.

Also, create a new *Secret Chat* chat room each time you start a new conversation. Do not reuse previously created *Secret Chat* chat rooms. And in case KakaoTalk shows a warning message, stop using it immediately regardless of the message.

In case you're using [Kakao Work](https://play.google.com/store/apps/details?id=com.kakaoenterprise.kakaowork), you're most probably affected as well since the app uses the flawed E2EE chat protocol I describe here.

Finally, [here](https://github.com/stulle123/kakaotalk_analysis/tree/main/scripts/mitmproxy) you find a couple of [mitmproxy](https://mitmproxy.org/) scripts to poke at KakaoTalk's chat protocol.

## Disclaimer

I already analyzed *Secret Chat* [back in 2016](https://kth.diva-portal.org/smash/get/diva2:1046438/FULLTEXT01.pdf) in KakaoTalk's Android version `5.5.5`. Last year, I revisited version `10.4.3` and discovered that most protocol weaknesses still exist.

I'm not making a statement that Kakao Corp. is actively monitoring E2EE chat messages or otherwise implying any malicious intent.

I'm making the point that Kakao Corp. *could* have the possibility to do so (as described below). But this is true for any chat app of course (as the paranoid among us are already aware :wink:).

## Background

With more than 100 million downloads from the Google Playstore, KakaoTalk is South Korea's most popular chat app. Similar to apps such as WeChat, KakaoTalk is an "all-in" app that includes various services in a single app (payment, ride-hailing services, shopping, e-mail, etc.).

End-to-end encrypted (E2EE) messaging is not enabled by default in KakaoTalk. Users commonly prefer regular chat rooms. Here, Kakao Corp. can access messages in transit. KakaoTalk does have an opt-in E2EE feature called *Secret Chat* but it doesn't support features such as voice calling or polls, events, and chat room albums.

The *Secret Chat* feature was added in 2014 after [revelations that KakaoTalk cooperated with the Korean government to spy on communications](https://www.bbc.com/news/blogs-trending-29555331). Around 400,000 active users left then to other more secure E2EE chat apps. As a response, KakaoTalk quickly added *Secret Chat* on top of its existing chat protocol to get users back. In the [Appendix](#appendix-political-context), we provide a timeline of the political events that led to the development of *Secret Chat*.

## Technical Analysis

In this section we describe how the LOCO chat protocol and *Secret Chat* work. For this, we reverse engineered KakaoTalk's Android app (version `10.4.3`).

### The LOCO Protocol

In April 2011, Kakao [announced](https://web.archive.org/web/20140422050306/http://blog.kakao.com/311) the *Fast Bull* project in which KakaoTalk implemented a major redesign of its chat protocol. The name *LOCO* is presumed to be the internal code name of this project. No, they probably don't mean the [JPEG LOCO algorithm](https://en.wikipedia.org/wiki/Lossless_JPEG#LOCO-I_algorithm), the [Korean rapper](https://en.wikipedia.org/wiki/Loco_(rapper)), nor the Spanish word *loco* which means crazy :wink:

Most of the LOCO protocol reverse engineering work has been already done [by Brian Pak in 2012](https://web.archive.org/web/20240325014628/https://www.bpak.org/blog/2012/12/kakaotalk-loco-%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C-%EB%B6%84%EC%84%9D-1/). Since then, the core of the protocol hasn't changed that much. There's also an open-source implementation of it called [KiwiTalk](https://github.com/KiwiTalk/KiwiTalk).

The LOCO chat protocol is simple and uses Binary-JSON ([BSON](https://en.wikipedia.org/wiki/BSON)) data packets to exchange messages between two or multiple parties.

Each participant in the messaging system can send a LOCO command which is tied to a specific action. For example, the `SWRITE` command writes an end-to-end encrypted message to a *Secret Chat* chat room and the `LOGINLIST` command is used to authenticate a KakaoTalk client against a LOCO messaging server.

{{< box info >}}
Here are a couple of example LOCO packets (raw and yaml) [here](https://github.com/stulle123/kakaotalk_analysis/tree/main/scripts/mitmproxy/tests/data).
{{< /box >}}

Most users use KakaoTalk's *Regular Chat* chat room where LOCO chat messages are encrypted with an AES key that is shared with Kakao Corp. That means that the company can read all user chat. A detailed description of the LOCO protocol can be found in my [thesis from 2016](https://kth.diva-portal.org/smash/get/diva2:1046438/FULLTEXT01.pdf#page=77).

### Secret Chat

*Secret Chat* is built on top of the existing LOCO protocol which means that E2EE messages are encapsulated into regular LOCO packets. The only difference is that E2EE messages are encrypted with an additional key. This key is not accessible to Kakao Corp. and doesn't leave the device (assuming the app is trusted).

![e2e](/img/secret_chat.png)

 *Secret Chat* is opt-in only and not enabled by default in the KakaoTalk mobile app. Additionally, most users might not even use *Secret Chat* as it doesn't have the same feature set as *Regular Chat* (e.g., no voice calling support).

#### E2EE Protocol

{{< box info >}}
The description provided here is super simplified.

You can find a detailed explanation of the protocol [here](https://kth.diva-portal.org/smash/get/diva2:1046438/FULLTEXT01.pdf#page=83) (might be a bit outdated).

If you are into reading messy code instead, here's a [mitmproxy script](https://github.com/stulle123/kakaotalk_analysis/blob/main/scripts/mitmproxy/mitm_secret_chat.py) which implements parts of the *Secret Chat* protocol.
{{< /box >}}

The message communication flow of the *Secret Chat* E2EE messaging protocol is illustrated in the figure below. For a sender, it takes the following three requests from creating a new *Secret Chat* room until sending an E2E encrypted message:

1. The sender sends a `SCREATE` LOCO packet to the messaging server to create the *Secret Chat* room on the server-side as well as to retrieve the recipient's RSA public key.
2. In the second request, the sender sends a `SET_SK` LOCO message to the server. The packet includes a shared secret value which is encrypted with the recipient's RSA public key.
3. During the third request, the sender sends a `SWRITE` LOCO message to the server. This message includes the E2E encrypted chat message. The server accepts the `SWRITE` message with a status code of `0x0`, transforms the `SWRITE` packet into a `MSG` packet and relays it to the recipient.

![e2e](/img/e2e.svg)

#### Trust Establishment

To guarantee that public keys are authentic and really belong to the intended user, KakaoTalk uses an authority-based trust model in combination with optional manual key-fingerprint verification.

By an authority-based trust model I mean that Kakao Corp. holds a central database that maps the user's mobile phone's device UUID to their public keys. Public-key ownership is verified through login credentials and SMS verification when users install KakaoTalk on their phones.

## Weaknesses

Kakao Corp. claims that it cannot read end-to-end encrypted messages sent via the *Secret Chat* chat room (according to their [privacy policy](https://privacy.kakao.com/protect#protectService)). However, as I outline in this section, this claim is questionable.

### Missing Ciphertext Integrity

All KakaoTalk communications are relayed through the LOCO messaging backend which decrypts packets, creates new ones, and forwards them re-encrypted to the destination. Due to the interception and transformation of messaging packets, there is no integrity verification implemented on the client-side. Otherwise, clients would constantly throw an exception that messages have been tampered with during transmission. 

The lack of integrity protection means that any local or global network MITM attacker could manipulate the ciphertext without being detected.

And since KakaoTalk LOCO packets are encrypted with a [malleable](https://en.wikipedia.org/wiki/Malleability_(cryptography)) block cipher mode (AES-CFB), targeted [bit-flipping](https://en.wikipedia.org/wiki/Bit-flipping_attack) attacks may be possible if parts of the plaintext are known ([see EFAIL attack from 2018](https://efail.de/)).

I've developed two simple mitmproxy scripts to demonstrate this issue:

- [replace_loco_message.py](https://github.com/stulle123/kakaotalk_analysis/blob/main/scripts/mitmproxy/replace_loco_message.py) -> Replace a LOCO message with another one to show missing integrity protection
- [flip_ciphertext_bits.py](https://github.com/stulle123/kakaotalk_analysis/blob/main/scripts/mitmproxy/flip_ciphertext_bits.py) -> a POC for showing the CFB malleability of LOCO packets

### Missing Freshness

While I didn't test this in depth, I found evidence that the protocol may not provide sufficient freshness to prevent replay attacks. In a MITM scenario I was able to replay chat messages and reorder packets.

### Missing E2EE Security Goals

In the following, I show a couple of missing security goals in KakaoTalk's *Secret Chat* protocol.

#### E2E messaging By Default

Instead of encrypting all messages end-to-end by default, KakaoTalk's *Secret Chat* feature is opt-in only and users need to explicitly choose to chat in an end-to-end encrypted manner.

#### Forward Secrecy

After a certain amount of time, KakaoTalk refreshes the E2E encryption key and presents a warning message to the user mentioning that the chat room is no longer available as the user's `encryption information has expired`.

However, the RSA key-pair, which is used to exchange a new secret value that in turn is used to generate a new message encryption key, is long-lived and is not replaced automatically (on Android, it's stored on the phone in the `TalkKeyStore.preferences.xml` Shared Preferences or `KakaoTalk.db` database).

An attacker who has obtained access to the user's private RSA key and to the provider-shared AES key may be therefore able to decrypt previously collected end-to-end encrypted messages. For this reason, KakaoTalk does not support the concept of [Forward Secrecy](https://en.wikipedia.org/wiki/Forward_secrecy).

#### Backward/Future Secrecy

Similarly, KakaoTalk does not provide [Backward/Future Secrecy](https://en.wikipedia.org/wiki/Double_Ratchet_Algorithm) since the user's RSA key-pair remains static.

#### Open-source, Open Documentation, Independent Security Audit

KakaoTalk's end-to-end encryption protocol is closed-source and does not provide an open protocol specification. Also, to the best of my knowledge, the application and protocol have not been subject to audit by an independent party.

#### Metadata protection

KakaoTalk does not protect the communication metadata, but instead (possibly) makes use of it by creating complex [social graphs](https://en.wikipedia.org/wiki/Social_graph). However, it is worth to note that all of the popular E2EE messaging apps that exist today do not provide metadata protection.

### Central Public-Key Database

As already outlined above, Kakao Corp. uses a central public-key database which makes the E2E encryption messaging system prone to MITM attacks on the operator/server-site. This attack may remain undetected if users do not compare their public-key fingerprints or if they don't get a notification when they have changed.

{{< box info >}}
Most of the E2EE chat apps operate a public-key database and as such they could also replace your public keys (e.g., WhatsApp, Signal, or iMessage). So, this is **not** a KakaoTalk-only issue.

I'm aware that there are plenty of other easier ways for Kakao Corp. to snoop on *Secret Chat* messages. They could just ship a bug/backdoor with a new KakaoTalk update for instance.
{{< /box >}}

MITM attacks can be also performed *after* two clients have agreed on a shared secret key since Kakao has the ability to delete *Secret Chat* chat rooms on the server-side. In this case the user would need to create a new chat room and the attacker would then be able to substitute a user's public key during the initial secret key exchange. 

### Missing User Notification

In addition, KakaoTalk does not immediately notify users if the other parties' public key has changed (on WhatsApp or Signal for example, users get notified when a contact has reinstalled the app or got a new phone).

After a potential MITM attack has occurred there's no immediate warning message that is prompted to the user. Only if both parties go to `Chatroom Settings` -> `Public Key` and compare their public-key fingerprints, the attack can be detected.

This missing notification allows an attacker with access to the server to secretly replace public keys and MITM communications. Even though this is a well-known attack, the missing user notification is still a critical issue. [In the section below](#man-in-the-middle-poc) I show a PoC to demonstrate the problem.

### No LOCO Server Authentication

Another design flaw in KakaoTalk's messaging system is the absence of server authentication in the LOCO messaging backend. While Kakao's Web API backend utilizes the HTTPS protocol to guarantee server authentication, the LOCO messaging backend uses the LOCO protocol which does not support such a security concept.

Since there is no server authentication, the KakaoTalk client would blindly trust a malicious endpoint. Moreover, a malicious server could send legitimate LOCO commands to the KakaoTalk client application to possibly retrieve sensitive information or change the communication flow.

## Man-in-the-Middle PoC

I've created a [simple script](https://github.com/stulle123/kakaotalk_analysis/tree/main/scripts/mitmproxy) to man-in-the-middle *Secret Chat* communications with [Frida](https://frida.re/) and [mitmproxy](https://mitmproxy.org/). It demonstrates a well-known server-side attack in which the operator (i.e. Kakao Corp.) or an attacker with access to the server could spoof a client's public key to intercept and read E2E encrypted chat messages (see figure below).

![e2e_msg](/img/mitm.svg)

This is how one can run the PoC:

- Assumption: You've already set up your test environment (see setup description [here](https://github.com/stulle123/kakaotalk_analysis/tree/main/scripts/mitmproxy))
- Wipe all entries in the `public_key_info` and `secret_key_info` tables from the `KakaoTalk.db` database
- Start `mitmproxy`: `$ mitmdump -m wireguard -s mitm_secret_chat.py`
- Start `Frida`: `$ frida -U -l debug_secret_chat.js -f com.kakao.talk`
- Create a new *Secret Chat* room in the KakaoTalk app and send a message
- View message in `mitmproxy` terminal window

How it works:

- Server-side `GETLPK` packet gets intercepted -> Inject MITM public key and grab recipient's public key
- Server-side `SCREATE` packet gets intercepted -> Remove an already existing shared secret (if any)
- Sender sends a `SETSK` packet -> `mitmproxy` script grabs shared secret and re-encrypts it with the recipient's original public key
- Using the shared secret, the script computes the E2E encryption key
- `MSG` and `SWRITE` packets are decrypted and dumped in the `mitmproxy` terminal

Demo:

![Secret Chat Demo](/img/secret_chat_demo.gif)

## Responsible Disclosure

I [reported my findings back in 2016](https://github.com/stulle123/kakaotalk_analysis/blob/main/report/kakaotalk_responsible_disclosure_2016.pdf), but as we've seen in this analysis most of them were not fixed. [Here's](https://github.com/stulle123/kakaotalk_analysis/blob/main/report/kakaotalk_correspondence_2016.pdf) the reply we got from Kakao Corp. back then.

## Conclusion

In this blog post I've shown a number of weaknesses in KakaoTalk's *Secret Chat* protocol. Most of the flaws are due to the underlying LOCO protocol which lacks ciphertext integrity and server authentication. Kakao Corp. had to use the legacy LOCO protocol to ship *Secret Chat* as quick as possible (see [Appendix](#appendix-political-context) below). I think that's the real lesson learned here: Developing an E2EE protocol is a complex topic and requires a lot of time. You cannot add a E2EE feature to a legacy messaging app without doing a lot of stuff from scratch again.

As future work, it would be interesting to check out how the nonces for the AES CTR mode are generated[^0]. As we can see [here](https://github.com/stulle123/kakaotalk_analysis/blob/main/scripts/mitmproxy/lib/crypto_utils.py#L144), the `msgId` or `chatId` is the only random element that goes into the nonce computation. How is the `msgId` or `chatId` generated? Could we maybe achieve a counter collision? :thinking:

## Appendix: Political Context

Here, I'm describing a timeline of events that happened in South Korea which in the end led to the development of the *Secret Chat* feature.

{{< box info >}}
I copied most of the following text from my own [thesis from 2016](https://kth.diva-portal.org/smash/get/diva2:1046438/FULLTEXT01.pdf).
{{< /box >}}

### The MV Sewol Ferry Disaster

There was a big disaster in April 2014, when a ferry carrying mostly school students capsized and more than 300 passenger and crew members died[^1]. The official handling and response to the ferry disaster provoked widespread criticism towards the government of president Park Geun-Hye and led to major demonstrations throughout the country. Worryingly, the criticism prompted a series of legal and political consequences.

On September 16, president Park Geun-Hye told in a cabinet meeting that `"profanity towards the president had gone too far"`, that `"insulting the president is equal to insulting the nation"`, and that false rumors `"divided [the] society"`[^2]. As a result, Park Geun-Hye pledged to prosecute people spreading rumors about her or the government on Twitter, social media and instant messenger services such as KakaoTalk.

Shortly after Park's speech, the Public Prosecutor's Office established an anti-defamation special investigation team which monitored and censored inappropriate public and private online content and also prosecuted suspects. The investigation team specifically monitored KakaoTalk to identify violations by reading chat messages in real-time. Kakao denied that it provided real-time wiretapping access. Nevertheless, Kakao showed collaboration by holding a closed-door meeting with prosecutors to discuss how online rumours and criticism could be suppressed[^3].

### Kakao's Cooperation in Wiretapping Users

Kakao's cooperation and the active monitoring of KakaoTalk led to unjustified and indiscriminate data collection and surveillance of protesters for violations of the Assembly and Demonstration Act[^4].

In October 2014, Jin Woo Jung, a vice representative of the Labor Party, announced in a press conference that he was accused by the government of `"causing public unrest"` for organizing a demonstration. Jung said that prosecutors had accessed his private KakaoTalk conversations as well as the personal details of his 3,000 contacts over a period of 40 days.

In another case, Hye In Yong, a university student who organized a solidarity march for the victims of the ferry tragedy, was also a subject of KakaoTalk surveillance. The search and seizure warrant against her included personal information, message content (texts, pictures, and videos) and metadata (e.g., KakaoTalk ID, phone number used for authentication, IP addresses, and other metadata). Among the metadata was supposedly also her cellular modem's MAC address that may had been used by authorities for tracking Yong's location[^5][^6]. Similar to Jung's case, the surveillance of Yong led to unjustified and indiscriminate data collection of innocent people because all of her contacts' conversations and metadata were also seized.

The revelations of the different surveillance cases and Kakao's participating role in them caused major concerns in the South Korean public. As a reaction, around 400,000 KakaoTalk users and 1.5 million South Koreans signed up for more secure non-South Korean-based chat applications within seven days[^7]. In addition, asked by a public survey conducted between October and November 2014, more than seven out of ten South Koreans answered that they were worried about secret government surveillance[^8]. 

In order to regain trust, Kakao's co-CEO Sirgoo Lee reacted with a press conference saying `"We stopped accepting prosecution warrants to monitor our users' private conversations from 7 October, and hereby announce that we will continue to do so."`[^9][^10]. Kakao also said that it had to comply with the law and that it was not able to deny governments' requests due to search and seizure warrants[^11].

The next day after Lee's press conference, Kakao announced to add end-to-end encryption support to KakaoTalk. The new opt-in feature *Secret Chat* was added to the Android version on 8 December and to the iOS version on 17 December 2014[^12].

### References

[^0]: AES CTR is used for encrypting chat messages in *Secret Chat* end-to-end.
[^1]: https://en.wikipedia.org/wiki/Sinking_of_MV_Sewol
[^2]: https://www.hani.co.kr/arti/politics/bluehouse/655420.html
[^3]: https://www.hani.co.kr/arti/economy/it/657974.html
[^4]: https://web.archive.org/web/20170915035340/https://apnews.com/97c92b056482488abd990db9a4acb388/s-korea-rumor-crackdown-jolts-social-media-users
[^5]: https://web.archive.org/web/20190512140836/https://freedomhouse.org/report/freedom-net/2015/south-korea
[^6]: http://transparency.kr/wp-content/uploads/2015/10/Korea-Internet-Transparency-Report-2015.pdf
[^7]: https://www.bbc.com/news/blogs-trending-29555331
[^8]: https://web.archive.org/web/20200217204555/https://freedomhouse.org/report/freedom-world/2015/south-korea
[^9]: https://www.accessnow.org/south-korean-im-app-takes-bold-stand-against-police-abuses/
[^10]: https://web.archive.org/web/20220306142229/http://www.businesskorea.co.kr/news/articleView.html?idxno=6846
[^11]: https://www.techinasia.com/daum-kakao-snooped-stopped-after-protest
[^12]: https://www.kakaocorp.com/page/detail/7652