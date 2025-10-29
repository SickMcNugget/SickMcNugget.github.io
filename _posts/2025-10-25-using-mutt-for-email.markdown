---
layout: post
title: "Using Mutt For Email"
date: 2025-10-29 14:12:55 +08:00
categories: tutorial linux email
published: false
---

I've been reading many a [Linux Mailing List] as of late. I'm always impressed that a huge project such as Linux is able to operate entirely through the use of emails. They attach code, create threads, CC others, reference parts of previous emails and sometimes draw pretty images all with plaintext emails.

This system is able to handle thousands of emails a day. For example, in October of 2025, the Linux Kernel Mailing List received on average 1269 emails per day (collected from [LKML](https://lkml.org/lkml/2025/10)).

I'm interested enough in some of these mailing lists that I'd like to subscribe to them, and have a convenient way to read the content of the messages. To do so, I'd like to set up the [mutt] email client. This is an old tool now, originally written in 1995, but is still the preferred email client for many kernel maintainers, although I've ripped that fact straight from an LLM, so who knows if it's true at all. There is a [page in the Linux docs discussing email clients](https://docs.kernel.org/process/email-clients.html#mutt-tui), where mutt is regarded quite highly.

I'm not that familiar with the internals of how email works, other than the difference between IMAP (server-stored) and POP3 (local-stored). As such, I'd also like to use this short journey to save that information so I can just refer back to it later.

# The innards of email, and other fun terminology
MTA: Mail Transport Agent
SMTP: Simple Mail Transport Protocol
IMAP: Internet Message Access Protocol
POP3: Post Office Protocol v3

## The order of events
How are emails send from start to end? [A CloudFlare Article](https://www.cloudflare.com/en-gb/learning/email-security/what-is-smtp/) covers this:
1. A "HELO" or "EHLO" is sent from the client to the server
2. The client sends a series commands followed by the email header, the email body and any additional components (does this mean attachments?)
3. On the server, a Mail Transfer Agent (MTA) is running. The MTA checks if the sender and the recipient have the same domain name. If they do, the message is sent directly, otherwise a DNS lookup is performed and then the mail is forwarded to IP retrieved.
4. The recipient uses IMAP/POP3 to "pull" this message from their mailbox server.

Note that SMTP serves as a "push" protocol (it sends messages TO mailboxes), whereas IMAP/POP3 are "pull" protocols (they receive messages FROM mailboxes).


# Using mutt for email
The first step to using any software is *installing* it. I'm going to use my distribution's version of the software to start with, but I may install from source somewhere down the track.
```bash
sudo apt install -y mutt
```

While we're at it, we need an actual Mail Transport Agent to send the mail.
```bash
sudo apt install -y sendmail
```


To *configure* mutt, I'm following the guide by


[Linux Mailing List]: https://lore.kernel.org
[mutt]: http://mutt.org
