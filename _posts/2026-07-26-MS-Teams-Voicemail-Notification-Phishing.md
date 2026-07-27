---
title: "Phishing through MS Teams Voicemail Notification E-Mails"
date: 2026-07-26 10:00:00 +0200
categories: [Social Engineering, Phishing]
tags: [Attack Engineering]
---

A while ago I stumbled over the voicemail e-mails sent out by <noreply@skype.voicemail.microsoft.com> as a result of a missed Microsoft Teams call when the caller left a voice message. I was intrigued and started playing around with the functionality, also because I couldn't find any related articles online talking about the feature from a cyber security perspective. I suspected some potential for phishing abuse since the e-mails emanated trust and appeared to be internally routed. Also the features "Voicemail for inbound calls" and "Voicemail transcription" are enabled by default in the MS Teams admin center. The content in the message is a result of a voice transcription of whatever the caller left on voicemail and is thus user controlled.

MS Teams phishing is nothing new. Threat actors have learned that many organizations have hardened their e-mail infrastructure and their phishing awareness of their users in regard to external e-mails. But far less so for collaboration tools such as MS Teams and others. For instance, APT Cloaked Ursa previously used MS Team calls and chats to social engineer users into confirming MFA prompts by initiating chat interactions and impersonating IT Support personnel (Source: [https://unit42.paloaltonetworks.com/microsoft-teams-phishing/](https://unit42.paloaltonetworks.com/microsoft-teams-phishing/)). Even [fake MS Teams voicemail phishing](https://itnsgroup.com/threats/microsoft-teams-voicemail-phishing-2026-06-28) was observed - but why fake it if you can deliver the real deal?

## Why are these e-mails special?

My investigation went in two directions: Firstly, how different is the e-mail flow to conventional, external e-mails? Secondly, how much can the caller influence the message that is supplied to the potential victim? 

### Mail flow

Analysing the path that the voicemails took revealed that they bypassed the traditional e-mail gateway infrastructure where most of an enterprise's e-mail security is placed in terms of content and reputation checks. These e-mails were delivered internally from M365-Teams directly to the user's O365-Mailbox. This implies that there is little control to scan such messages before they reach a client's MUA (unsure if Microsoft O365 even scans such messages with the usual mailbox protection since they come from a Microsoft address - this I did not test in detail). As a result, the e-mail gateway cannot flag the e-mail as \[External\] to warn the user, despite being internally routed, since the message body content is caller controlled.

### Content manipulation

The message left by a caller is likely transcribed from voice-to-text via Azure's speech services, which are part of Microsoft Foundry Tools (formerly Azure AI Speech). I performed various tests to test for prompt injection and HTML injection - to achieve more enticing baits. However, these tests were mostly fruitless. What I quite easily managed was to inject and transmit URLs as the example below shows. Despite Outlook Desktop and Web not rendering the links, Outlook Mobile was kind enough to make them clickable for the user.

![Delivered ClickFix phishing lure](/assets/img/posts/2026-07-26-MS-Teams-Voicemail-Notification-Phishing/voicemail_bait.jpg)

## Attack scenario

The scenario that I tested and envision has the following steps:

1. Enumerate valid MS Teams VoIP numbers. There used to be functional projects for this (e.g. [https://github.com/waffl3ss/KnockKnock](https://github.com/waffl3ss/KnockKnock)). However, I found checking a target organization's public contact phone number often gives a good starting point to "brute-force" legit MS Teams connected numbers, since they normally have a registered range - similar to an IP subnet.  
2. Attacker calls a victim's MS Teams phone number and goes to voicemail if the call is not answered. Ideally, the attacker calls during the night in the victim's timezone to trigger this.
3. Attacker plays a recording that is then transcribed by Microsoft’s backend. This could contain a malicious instruction with a link as seen in the example below. I used [Narakeet.com/app/text-to-audio/](https://www.narakeet.com/app/text-to-audio/) for this step with the subsequent input. Commas are placed to extend the vocal pause between words. Some special symbols such as "/" are written out (forwardslash) to ensure proper URL transcription:

```
You missed a call from your Cyber Defense Center, Dear User, we noticed a security issue with your machine,. Since we could not reach you, please go to https evil.com , forwardslash , clickfix , and follow the instructions to secure your device. Thank you for protecting the enterprise from further damage.
```

4. Victim sees an email without an [External] tag that seemingly originates from an internal, trusted source – namely Microsoft. The content is thus treated with more trust, and the user is more likely to follow the instructions in the transcription.

This entire attack chain can of course be automated for scalability to deliver phishing e-mails to users while bypassing normal e-mail gateways and sender reputation checks. One could also start playing around with the attached `.mp3` file. Deep fake CEO voice calls would come to mind, but there are likely other creative ways.  

## Mitigations

To harden your environment and protect your users, the following measures can reduce the exposure:

- Likely the easiest and fastestest to mitigation: Implement a transport rule under Mail flow rules in the Exchange Online admin center which detects messages of type voicemail and pre-fixe a message on them to inform the user about the risks. Also, the known \[External\] tag can be supplied to the subject (Credit: [Ben Pyett - MS Techcommunity](https://techcommunity.microsoft.com/discussions/microsoftteams/voicemails-marked-as-external-messages---prevent-phishing-but-how/3298567)).
- Implement a reroute of the voicemail notification e-mails to your e-mail gateway to apply the same e-mail security scrutiny before forwarding them to the user's M365 mailbox.
- Change the calling policies for "Voicemail for inbound calls" and "Voicemail transcription" in your MS Teams admin center and potentially disable them outright. (Source: [https://learn.microsoft.com/en-us/answers/questions/2152889/where-is-the-setting-to-send-voicemails-as-an-email](https://learn.microsoft.com/en-us/answers/questions/2152889/where-is-the-setting-to-send-voicemails-as-an-email))
- Check on Exchange Online what kind of content checks are enabled for "trusted" senders.

## TL;DR

**In summary, these characteristics make the voicemails interesting for social engineering attacks:**
- Voicemail notification e-mails have a trusted Microsoft sender address, namely <noreply@skype.voicemail.microsoft.com>
- For most users, these e-mails look like internal e-mails and treat them with more trust, despite the content being externally controlled.
- The e-mail content is not routed via the e-mail gateway and thus undergoes likely less security checks.
- Transcribed URLs are properly parsed and even made clickable by Outlook Mobile.
- Even if Teams federation is disabled, which limits chat-based B2B invitations, the VoIP calls still go through.