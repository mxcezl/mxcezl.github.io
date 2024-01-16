---
title: "[Intro][PwnMe 2023][OSINT] Social Media Goes Brrrr"
date: "2023-05-07"
categories: [ "CTF", "PwnMe 2023 CTF", "OSINT" ]
slug: "social-media-goes-brrrr"
ShowToc: false
draft: false
weight: 1
---

## Introduction

This challenge is an OSINT challenge from the PWNME CTF.

### Context explanation

John Droper is a Franco-British individual who leaves an enormous digital footprint.

(Note: He speaks both English and French, and some information can only be found through one of these languages.)

### Directive

You have to find one of his main social media

## Solution

So here we only have one information about the person, his name. So we will have to search for it on the internet.

Since we have to find a social media, I first checked if he had a Facebook account. I searched for his name on Facebook and found a profile with the same name and a sketchy profile picture.

![Facebook profile](/images/pwnme2023/osint/john_droper_facebook_profile.png)

By looking under the about section, we can see in the details that there is the flag.

![Facebook flag](/images/pwnme2023/osint/john_droper_facebook_details.png)

Flag : `PWNME{TG9uZyBsaXZlIHRoZSB0cmFpbnMsIGxvbmcgbGl2}`
