---
title: "My (failed) attempt to unsubscribe from UnderArmour emails"
date: 2022-02-08T11:50:00-06:00
summary: "I started getting promotional emails from UnderArmour in my personal email address, even though I never bought anything from them nor subscribed to any promotional newsletter. I tried to unsubscribe, but the link was broken. This is my story."
---

About 6 months ago, I started getting promotional emails from UnderArmour in my personal email address, even though I never bought anything from them nor subscribed to any promotional newsletter.

So, what’s the deal? I just need to click on the included unsubscribe button, right?

![UnderArmour Unsubscribe Page](/my_failed_attempt_to_unsubscribe_from_underarmour_emails/button.jpeg)

Well… it’s not that easy, since that button takes to a web page that doesn’t work.

## Gmail filters to the rescue

Now what? I can’t unsubscribe and I am getting about multiple emails every week filling my inbox. I could just mark those emails as spam and move on with my life, but since Gmail has been throwing some important emails into the spam folder and I check it regularly to make sure I don’t miss something important; I didn’t want to see UnderArmour’s emails anymore.

The workaround I found was to use [filters](https://support.google.com/mail/answer/6579) that automatically delete all emails from _underarmour@e.underarmour.com_ and prevent them from ever reaching my inbox again.

I was satisfied with this solution since I didn’t get to see those emails anymore, but all this time I was annoyed with the fact that I never gave them my email and they wouldn’t let me unsubscribe.

## Investigating the broken unsubscribe link

Being a curious student and bothered by this, my only option was to look into this and stop those emails from coming in.

The unsubscribe button in the email takes you to _trk.e.underarmour.com_ but responds with a 302 code and redirects to _pages.e.underarmour.com_, so let’s keep going.

The next URL to test is _pages.e.underarmour.com_, and at first glance, it seems that this is related to DNS.

![](/my_failed_attempt_to_unsubscribe_from_underarmour_emails/cant_connect.jpeg)

Initially, I thought it might have something to do with my [PiHole](https://pi-hole.net/) since it sometimes breaks some pages, but even with PiHole deactivated I couldn’t access the page that would supposedly let me unsubscribe.

## The missing record

A quick search in [securitytrails.com](https://securitytrails.com) shows that an A record for _pages.e.underarmour.com_ indeed existed, but was deleted 2 years ago.

![](/my_failed_attempt_to_unsubscribe_from_underarmour_emails/dns.png)

So now what? It seemed like a dead-end but there was one last thing I wanted to try.

## Searching all underarmour.com’s subdomains

Fortunately, there are websites that already keep track of domains like [this one](https://subdomains.whoisxmlapi.com/). The only problem is that there are 344 subdomains for _underarmour.com_.

I tried the ones that included the word email or some variant of it, and after trying only 3 subdomains I stumbled across one that seemed promising: _pages.emails.underarmour.com_.

Then, I replaced _pages.**e**.underarmour.com_ (the link that the unsubscribe button sent to) with _pages.**email**.underarmour.com_, keeping the path of the URL.

Et voilà, I could finally unsubscribe from UnderArmour emails. Or not…

![UnderArmour Unsubscribe Page](/my_failed_attempt_to_unsubscribe_from_underarmour_emails/cancel.png)

## Wrapping up

I want to give UnderArmour the benefit of the doubt and to think that they just made a mistake and were redirected to the wrong subdomain, but it seems that there are other people with this problem. I have continued to receive promotional emails even though I have submitted the form to unsubscribe multiple times.

More and more companies are adopting these kinds of tactics to retain users, but for what purpose? After this experience, I would never want to buy something from them; in fact, I would try to dissuade anyone from doing it.

**What should we do to stop receiving your emails, UnderArmor?**
