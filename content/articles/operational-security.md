---
title: "Operational Security Basics"
date: 2022-05-25T23:15:00+07:00
slug: operational-security-basics
category: article
summary: How you can modify your day-to-day online practices to increase opsec?
description: 
cover:
  image: covers/opsec.jpg
  alt: 
  caption: 
  relative: true
showtoc: true
draft: false
---

_In this piece, I want to discuss how you can modify your day-to-day online practices to increase opsec (Operational Security)._

_Before diving in, take into account that it can be overwhelming at first. You don't have to adopt the best cybersecurity practices right away. The idea is to lay out different options and encourage you to take small steps that will help make incremental improvements to your usual online and on-chain activities._

---

If you've never been hacked, you've most likely never bothered thinking about opsec before. You may believe that you are completely safe because you use a hardware wallet as most people advise. But have you ever scrutinized your online routines? Have you ever sought out flaws in your current system?

Because the answer to these questions is most likely _“No”_, I want to showcase how you can safely approach 3 different areas that **every** crypto investor has to deal with regularly: wallets, passwords, and emails.

## Wallets

Since wallets hold the keys that give access to users’ funds, it is the first area that should be protected. Ultimately, any hack that ends up in stolen funds, manages to do so by accessing the user's private keys.

If you are reading this, you have probably heard the famous quote "not your keys, not your coins" more times than you can count. Because of that, I will assume that you have control of your private keys and I will not consider wallets guarded by third parties, such as centralized exchanges, in this article.

So, which options do users have when safeguarding their funds in a self-custodial manner?

### Software Wallets (Hot Wallets)

The first level of security lies in the so-called “hot wallets”. Software wallets are always connected to the internet and, therefore, are the most convenient and easy to manage. This type of wallet allows users to hold their private keys, unlocking the full potential of censorship-resistant assets. Nevertheless, the downside of these types of wallets with such levels of convenience comes at the expense of security, since if the device that holds the wallet gets hacked, the attacker could easily access its private keys and steal the wallet’s funds.

If you want to stick with this option, you should spread your funds between different wallets to minimize the impact of a potential exploit.

> **Remember:** You don’t want to put all your eggs in one single basket.

### Hardware Wallets (Cold Wallets)

The second level of security belongs to cold wallets. These types of wallets allow users to store their private keys inside an offline device (hardware). The usage of these devices ensures that unless attackers manage to physically get the device or the recovery phase (which should only be stored offline), they won't be able to get the private keys.

Despite cold wallets being safer, because they also force users to have the hardware device in hand when signing transactions, they are less convenient when compared to hot wallets. 

The most common hardware wallets are those manufactured by Ledger, Trezor, and Grid+. Although cold wallets already carry greater security, it is not a bad idea to minimize risk by spreading the funds between wallets of different brands. By doing so, you would be protected in the unlikely scenario that one of these companies is compromised. 

_**Note:** You should also take into account that it is important to prepare for a potential wallet restoration process. If you end up losing access to the device that holds private keys (i.e. broken hardware wallet), you want to be certain that you have a backup of the private keys and know how to restore the wallet. One of the cool features of Grid+ is the ability to use SafeCards, which are password-protected backups of private keys._

### Smart Contract Wallets (Gnosis Safe, Argent Vaults, Authereum, etc.)

As their name explains, smart contract wallets, are a type of wallet that is managed by a smart contract instead of a private key. This feature makes these types of wallets the most secure thanks to their flexibility. The ability to use a custom logic adds extra layers of validation (multi-sig transactions), and can even add constraints like daily transfer limits. On top of that, they also have account recovery mechanisms.

Of all the mentioned features, the most interesting is the multi-sig transactions. These properties require the signature of multiple wallets (guardians) to confirm any transaction. If you decide to use a smart contract wallet, you should set up an N of M multi-sig with N > 1. These configurations will ensure that at least 2 signatures are always required to sign a transaction, minimizing the risk of exploitation and ensuring that there won't be a successful attack with a single point of failure.

Despite smart contract wallets requiring more effort when signing transactions (either a single user has to access multiple devices or multiple users have to coordinate), they are also extremely convenient in terms of account recovery. An N of M multi-sig setup with N < M grants the possibility to still access the wallet funds despite losing access to one of the private keys (in case of loss, damage, or hack of a guardian).

{{< figure src="/images/opsec/eoa-vs-ms.jpg" align=center caption="_Comparison between EOAs (accounts controlled by regular wallets) and Smart Contract Wallets._" >}}

For example, a strong setup would be using a Gnosis Safe that required 2 signatures from the following 4 types of wallets:

- **Personal 1:** A software wallet (Metamask) in a computer that is only used for crypto-related stuff.

- **Personal 2:** A hardware wallet that is stored at home, so it is easy to access.

- **Backup 1:** A second hardware wallet safeguarded by a trusted person.

- **Backup 2:** A paper seed phrase safeguarded by another person (a different one) of trust.

Although you could be using a quite strong setup as described, there would still be some measures that you could eventually take and further increase its security. For example, you could also replace the paper wallet with something less prone to damage such as a CryptoSteel wallet. You could also set up an address whitelist, a daily transaction limit, and even a freezing period when moving funds.

_**Note:** If you have multiple guards, they shouldn’t be aware of the existence of the other guards. Otherwise, they could decide to cooperate and exploit your safe._

### Final thoughts on wallets:

Each of the listed wallet types offers different levels of security, but it is important to understand that they also come with different levels of convenience and ease of use. Because of that, even if you move up the security ladder, it makes sense to combine the different options (for example having most of the funds in a safe but also having a hot wallet to mint NFTs).

## Passwords

Most crypto investors limit their security practices to wallets but forget that hackers can get plenty of insights by accessing their accounts. How should you prevent your accounts from being hacked?

### Password Security

Password “entropy” or randomness, is a measurement of how unpredictable a password is. This measure is based on the characters used (lowercase, uppercase, numbers, and symbols) as well as the length. Password entropy predicts how difficult a given password would be to crack through guessing, brute force, or other common methods.

Weak passwords are flawed due to a lack of randomness. For example, have you ever used the name of a relative or a pet in a password? Maybe a special date? Not very random, is it? Remember that passwords that are not randomly generated can be easily cracked if hackers get access to your private information.

Although the typology of the characters increases the entropy of a password, the variable which has the biggest effect is the length. Therefore, you should move away from the old password standards and start prioritizing length over special characters. In summary, you should move away from passwords that are hard to remember for humans but easy to crack, and start embracing easy-to-remember passwords that are hard to crack.

{{< figure src="/images/opsec/pwd.jpg" align=center caption="_Comic by xkcd.com_" >}}

If you want to be like the guy in the picture, I recommend using [diceware](https://www.diceware.net/) to create strong random passwords. By doing so, you’ll only have to remember a couple of words that will produce a high-entropy and secure password. On top of that, you can periodically add extra words as you are certain that you have already internalized your current words.

### System Robustness

The overall safety of a system is determined by its weakest component. Because of that, there is a key principle that should be taken into account when building safe systems: **compartmentalization**.

If a system is built with components that are isolated from each other, that system is going to be more resistant to external attacks. Even if a component gets compromised, the attacker won’t be able to access the other ones.

Because of that, it is important to focus on the security of all of your accounts and have a different password for each of them. Note that password derivatives do not count as independent passwords.

_How can you deal with completely random credentials for each of your accounts?_ By using password managers.

Password managers generate random, strong, and unique passwords that are stored in a vault, and encrypted (usually using AES or SHA 256-bit encryption) with one master password. They ensure high security and users only need to remember one strong password to access all their credentials.

#### Third-party password managers (LastPass, Bitwarden, 1Password, etc.)

Third-party password managers are user-friendly and offer features such as multi-device synchronization and a nice user interface with features such as password autofill. Most of them are freemium, offering more advanced features in the paid tiers.

A key distinction that is worth making is that whereas LastPass and 1Password are closed source, Bitwarden is open source. Because of that, it is always easier to trust publicly available software where anyone can audit its code.

_Note that Bitwarden can be self-hosted on your own server, giving you full control over your data and where it is stored. However, self-hosting requires technical expertise and server resources, and may not be suitable for everyone._

#### Local password manager (KeePass, KeePassXC, etc.)

KeePass, KeePassXC, and the rest of KeePass forks are free and open-source software with active development communities that have been around since the early 2000s.

The main difference between third-party password managers and local password managers lies in where data is stored. Whereas third-party software stores the data on cloud servers, local password managers such as KeePass do it locally.

Although third-party managers ensure that data is fully encrypted before syncing in the cloud (to ensure privacy and security), this method is less secure than strictly storing the information locally. On top of that, some may argue that despite having teams of security experts, these companies are honeypots for hackers and more likely to receive attacks.

Another important difference is that local password managers are way simpler and have a worse user experience than their third-party equivalents. To add extra features and improve the UX, third-party plugins must be installed. Although this feature makes it highly customizable, it also requires a higher technical knowledge.

_**Bonus Tip:** If possible, you should always enable 2FA (Two Factor Authentification) in all your accounts. When dealing with 2FA you should never use SMS, since attackers could get a copy of your SIM card. It is best to use apps such as Authy or a hardware device like a YubiKey._

## Emails

Passwords are the key that unlocks your accounts, but emails are what ultimately link your real identity with your online activity. You can think of emails as a second password. If the attacker is unable to figure out your email, they can’t access you.

### Email provider

The first thing that you should worry about when dealing with emails, is which email provider will you decide to use. An email provider implements email servers to send, receive, accept and store email on their users’ behalf.

I recommend using [Protonmail](https://protonmail.com), an open-source privacy-focused email service provider. Since the company is based in Switzerland, they don’t have to stick to EU/US laws and can implement end-to-end encryption.  E2EE is a communication system that prevents third parties from accessing the transferred data. It is capable of doing so by encrypting the messages with the receiver’s public key. This mechanism ensures that only the receiver will read the message’s content.

{{< figure src="/images/opsec/proton-e2e.jpg" align=center caption="_Diagram of how Protonmail’s E2EE works._" >}}

On top of that Protonmail is open-source, has a neat UX/UI, and a quite generous free tier service.

---

As discussed before, compartmentalization plays a key role in the robustness of a system. Because of that, the same principles that were described earlier apply here as well. To minimize the impact of an exploit, you should ideally have a separate email for each account.

Since creating a brand new email for each account can be time-consuming and hard to deal with, there are a couple of alternatives that you should consider:

### Clustering

Account clustering aims to help you find a sweet spot between the burden of creating a new account for every purpose and reusing the same account for everything. Its main principle consists of isolating the different activities of your online life from each other. If one of your accounts gets leaked or compromised, I won’t impact the other areas of online life.

### Aliasing

Email aliases are a special type of address that forward to your master accounts all the emails addressed to them.

Although email aliases can't completely stop phishing attacks, they act as a proxy and, therefore, add an extra layer of security. Using email aliases gives you agency over the information that you give away, and protects your master email address from third parties. This is especially convenient against data leaks and also against companies that collect and/or sell your data.

{{< figure src="/images/opsec/alias.jpg" align=center caption="_Diagram of how aliases work._" >}}

Services such as SimpleLogin or AnonAddy are great open-source aliasing tools that easily help you deactivate compromised or perished aliases. They also offer additional features such as subdomains and even allow for multiple master emails. On top of that, SimpleLogon has been recently acquired by the privacy-focused Swish company Proton, which is always a reassurance of their great product.

Although strong passwords can protect you from brute force cracking attacks, they won't protect you against fishing. An underrated feature that aliases (and email compartmentalization) provide is that they not only reduce spam but also help you identify suspicious emails that shouldn't be received in a given account.

Imagine that you receive an email that wants you to click on a link to reschedule an order that can't be delivered at your place, but the destination address is your alias account for newsletters. If that was about to happen, you could be certain that the link was malicious.

_**Bonus Tip:** Ad blockers are also a great tool to prevent malicious attacks when browsing. uBlock Origin and AdBlockPlus are free, powerful, and open-source options._

## Closing Thoughts

There usually exists an inverse correlation between security and convenience. Ultimately, a system will only be secure if you continuously stick to the best practices, so it’s best for you to slowly climb the security ladder and slowly build new habits. The goal of this piece is to explain different alternatives and encourage you to find your personal sweet spot.

I just want you to take into account that, as your portfolio grows, and more funds are at risk, you should ensure a stronger opsec. If you have never taken any measures before, it can be quite challenging to suddenly implement big changes, so it is best to have the discussed principles into account from the very beginning.

In life, some lessons are worth learning from other people's experiences. In my personal opinion, suffering an exploit because you have bad opsec practices is not a lesson that you want to learn firsthand. Because of that, every crypto investor, and anyone who cares about privacy, should be able to control their digital footprint (know all the accounts they own) by using their password manager.

_If that’s not your case yet, what are you waiting for?_

