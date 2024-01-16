---
title: "[Medium][PwnMe 2023][OSINT] French Dream"
date: "2023-05-07"
categories: [ "CTF", "PwnMe 2023 CTF", "OSINT" ]
slug: "french-dream"
ShowToc: false
draft: false
weight: 3
---

## Introduction

This challenge is an OSINT challenge from the PWNME CTF.

### Background explanation

To understand the context of the challenge, look at the description of the introductory challenge.

### Directive

John is French-English but lives in France, and his life is almost entirely available on the internet.

Find the city where he lives, the username of his current girlfriend, and the maiden name of his ex.

OSINT must remain passive, and any interaction is strongly prohibited.

There is no need to contact anyone to solve the challenge.

### Flag format

PWNME{lille_elonmusk_halliday} All in lowercase with "_" to separate the information.

## Solution

The first piece of information I found was the city where he lives.

### City

On Instagram, John Droper posted two posts. One where he talks about a 10 euro bill he used in Austria, and in the other post, John says it's good to have a barbecue with friends at his place after returning from vacation.

The post consists of 6 photos, one of which seems to contain interesting information:

![insta post](/images/pwnme2023/osint/droperkingjohn_insta_meet.png)

In this photo, we see an incomplete receipt because the top of the ticket is missing.

However, if we access the photo outside the post's pop-up by refreshing the page, we can see the photos in their actual size:

![insta post full](/images/pwnme2023/osint/droperkingjohn_insta_meet_full.png)

On the receipt, we see a mention of the city of `Lannemezan`.

We can deduce that he lives in this city since the post talks about a barbecue at his place, and he probably bought meat from the local butcher.

### Maiden name of his ex

Still on Instagram, `@dropkingjohn` posted several highlighted stories, one of which invites support for the Tarbes women's basketball club, the TGB.

![Instagram](/images/pwnme2023/osint/droperkingjohn_insta_tarbes.png)

On the side of the poster, we see the address where the match will take place, Quai de l'Adour in Tarbes.

Then follows another story that suggests his friends join him for a beer in "the best bar" after the match.

![Instagram](/images/pwnme2023/osint/droperkingjohn_insta_tarbes_biere.png)

On his Facebook profile, there was a post where John talked about his favorite bar:

![Facebook](/images/pwnme2023/osint/john_droper_facebook_bar.png)

In the Instagram story, John says he'll have to come disguised so as not to get caught.
Surprisingly, in the Facebook post, he says his ex is the owner of the bar.

By cross-referencing this information, we can deduce that his ex is the owner of the bar he will go to after the match.

In the Instagram story where he invites people to go to the bar after the match, we can see a large encrypted message:

```
3 444 777 33 222 8 444 666 66 0 66 666 777 3 0 555 33 0 555 666 66 4 0 3 33 0 555 2 0 777 444 888 444 33 777 33 0 7 777 33 6 444 33 777 0 22 2 777
```

The message looks like an encrypted message with a phone, so we can try to decrypt it by matching phone keys to letters.

There are probably online tools to do this, but I made this Python script to do it:

```python
telephone = {
    "2": "a",
    "22": "b",
    "222": "c",
    "3": "d",
    "33": "e",
    "333": "f",
    "4": "g",
    "44": "h",
    "444": "i",
    "5": "j",
    "55": "k",
    "555": "l",
    "6": "m",
    "66": "n",
    "666": "o",
    "7": "p",
    "77": "q",
    "777": "r",
    "7777": "s",
    "8": "t",
    "88": "u",
    "888": "v",
    "9": "w",
    "99": "x",
    "999": "y",
    "9999": "z",
    "0": " "
}

message_encode = "3 444 777 33 222 8 444 666 66 0 66 666 777 3 0 555 33 0 555 666 66 4 0 3 33 0 555 2 0 777 444 888 444 33 777 33 0 7 777 33 6 444 33 777 0 22 2 777"

chiffres = message_encode.split()

message_decode = ""
for chiffre in chiffres:
    message_decode += telephone[chiffre]

print(message_decode)
```

This gives us the result: `direction north along the river first bar`

Looking at the bar's location on Google Maps, we can see it is situated along the Adour River.

If we search for a bar in the area to the north of the bar, we find `Bar Le Landais`, which is located in `Tarbes`.

![Google Maps](/images/pwnme2023/osint/maps_ladour.png)

By searching the bar's name and the city's name, we find a link to the Yellow Pages, which gives us the owner's name: `P******* Caussade`.

![Yellow Pages](/images/pwnme2023/osint/pj_bat_tarbes.png)

We know we are looking for a maiden name, so we can try to find the bar owner by looking for her Facebook profile.

![Facebook](/images/pwnme2023/osint/fb_caussade_bar.png)

Only one profile catches my attention, that of `P******* Caussade Ghestem`, who is a `bar merchant` in `Tarbes`.

We can deduce that his ex's maiden name is `Ghestem`.

### Current girlfriend's username

Earlier, I had found his Twitter account but hadn't yet explored the lead.

Looking at his followers, we can see that he follows `@BlancheLoveJD`, whose bio says `I Love JD I always wanted to make him forget his ex gf`

![Twitter](/images/pwnme2023/osint/droperkingjohn_twitter_follows.png)

We are sure that she is his girlfriend.

### Flag

We now have all the information to find the flag:

- City: `Lannemezan`
- Current girlfriend's username: `BlancheLoveJD`
- Ex's maiden name: `Ghestem`

To recall, the flag format is PWNME{CITY_USERNAME_NAME}, all in lowercase with "_" separating the information.

So the flag is: `PWNME{lannemezan_blanchelovejd_ghestem}`

## Tips & Tricks

- Instagram crops images, so it's important to view the full images to find information. To do this, on desktop, simply refresh the page, and on mobile, tap on the image to enlarge it.
- Always cross-reference the gathered information.
- Don't hesitate to use Google Maps to search for locations.