---
title: "[Medium][PwnMe 2023][OSINT] Europe"
date: "2023-05-07"
categories: [ "CTF", "PwnMe 2023 CTF", "OSINT" ]
slug: "europe"
ShowToc: false
draft: false
weight: 4
---

## Introduction

This challenge is an OSINT challenge from the PWNME CTF.

### Context explanation

To understand the context of the challenge, look at the description of the introduction challenge.

### Directive

John loves adventure and travel. Can you give me the 3 cities he visited during his trip to Europe?

### Flag format

PWNME{city1_city2_city3} Cities in lowercase and in alphabetical order, separated by an "_".

## Solution

On John Droper's GitHub, `droperkingjohn`, on one of his commits, we can see that he removed part of his `index.html`.

![GitHub commit](/images/pwnme2023/osint/droperkingjohn_github_train.png)

```html
<body>
	<h1>John Droper's website</h1>
	<p>I will edit my curriculum later but this is some information about me:</p> <br>
	<p>- I could be pro with my team, but you know... (cruciate ligament rupture...) => https://www.tgb-basket.com/<br>- If you've already met me, you know that I love traveling; this is one of my new favorite forums (I'm not really active, but recently I shared information about my trip to Europe) =>  https://www.forum-train.com/forum/index.php</p>

	<hr style = "border:none;margin-bottom:20%">
	<p class="semiSecret">You can contact me :<br>johndroperdroperjohn@gmail.com<br>(links are not up for the moment, sorry...)</p>
	<div class="logosRS">
		<img src="./twitter.png" sizes="30px"/>
		<img src="./facebook.png"/>
		<img src="./instagram.png"/>
    <!-- No copyright but => https://github.com/droperkingjohn -->
	</div>
</body>
</html>
```

This commit is interesting because it gives us several pieces of information:

- An email address: `johndroperdroperjohn@gmail.com`
- A forum where he shares information: `https://www.forum-train.com/forum/index.php`
- The social networks he uses: `Twitter`, `Facebook`, and `Instagram`
- A basketball site: `https://www.tgb-basket.com/`

I searched for John Droper's two pseudonyms, `jdthetraveller` and `droperkingjohn`, on Twitter and Instagram, and I found these two profiles:

- Twitter: `https://twitter.com/droperkingjohn`

![Instagram](/images/pwnme2023/osint/droperkingjohn_twitter.png)

- Instagram: `https://www.instagram.com/droperkingjohn/`

![Instagram](/images/pwnme2023/osint/droperkingjohn_insta.png)

After finding the social media accounts mentioned on John's blog, I tried to find others by searching for his pseudonyms on Google and Bing.

### First city: Kaunas

![jeuxvideo jdthetraveller](/images/pwnme2023/osint/jdthetraveller_bing_jv.png)

We find a link to a thread on the `jeuxvideo.com` forum where John posted a message.

#### The jeuxvideo.com post

![jeuxvideo jdthetraveller](/images/pwnme2023/osint/jv_jd_thread.png)

~~~
Hi all,
I'm totally new to the site and a friend told me about the fact that you could help me,
I have to find another friend but he sent me 3 too weird messages the first was "investor",
the second "utopian" and the third "ligamentous".
I know it is mysterious but here I am completely lost.
I think losing that friend will be easier than solving the investigation.
~~~

If we search for the three words in French `investisseur`, `utopique`, and `ligamenteux` on https://what3words.com/, we find **`Kaunas`** in Lithuania.

#### The location with what3words

![what3words](/images/pwnme2023/osint/3words_osint.png)

So we have the first city visited by John Droper: **`Kaunas`**.

### Second city: Gols

On his Instagram posts, John posted a photo of a European banknote with the serial number visible.

![Banknote](/images/pwnme2023/osint/droperkingjohn_insta_bill.png)

The serial number is `U50441715662`.

From the description of the post, we can guess that John intends to do bill tracking to find his cherished banknote.

This lead takes us to the website https://www.eurobilltracker.com/ where we can track banknotes.

#### John on EuroBillTracker

By searching for his pseudonym `jdthetraveller` on the site, we find an account belonging to him, registered in `Gols`, Austria.

We are sure it's his account because he mentioned the URL of his blog in his description.

![eurobilltracker](/images/pwnme2023/osint/jdthetraveller_eurobilltracker.png)

![eurobilltracker](/images/pwnme2023/osint/jdthetraveller_eurobilltracker_location.png)

So we have the second city visited by John Droper: **`Gols`**.

### Third city:

Going back to the initial information obtained from John's GitHub, we know he has an account on the forum `https://www.forum-train.com/forum/index.php`.

#### John on forum-train

By searching with a Google dork `site:forum-train.com jdthetraveller`, we find a presentation post by John on the forum.

![forum-train](/images/pwnme2023/osint/google_forum-train.png)

This post tells us that John left Bratislava by train after traveling by car and that the train journey lasted exactly `10h58`.

![presentation forum-train](/images/pwnme2023/osint/forum-train_presentation.png)

```
Hello everyone,

Let me introduce myself, John D.,
I've been a fan of travel for a very long time!
I recently took a trip to Europe and part of that trip was by train.
After going to Central Europe I decided to go to Eastern Europe.
I left Bratislava and traveled for 10h58 it was really very long... 

If you have any tips for discovering places in the United States, I'll be there soon
```

Further down, in another message, John says he didn't have time to visit Bratislava, which allows us to confirm that Bratislava is not one of the cities visited by John.

![presentation forum-train](/images/pwnme2023/osint/forum-train_presentation2.png)

```
I forgot to mention that I didn't even have time to visit Bratislava unlike the other cities of my trip to Europe...

Otherwise, first of all thank you for the welcome!

I'm going to try to visit all the coolest corners of the United States so I'll take all the advice!
```

#### The train journey

With the information we have, we can search for journeys starting from `Bratislava` and lasting `10h58` on the website https://direkt.bahn.guru/.

If we set the departure point as the `Bratislava hl.st.` station, the map indicates that the city `Terespol` is `10h58` from `Bratislava` by train.

![train time map](/images/pwnme2023/osint/bahn_guru_train.png)

The city `Terespol` is indeed `10h58` from `Bratislava` by train, but most importantly, it is also in Eastern Europe, which corresponds to what he said in his presentation.

So we have the third city visited by John Droper: **`Terespol`**.

### The flag

The three cities visited by John Droper are:

- `Kaunas` in Lithuania
- `Gols` in Austria
- `Terespol` in Poland

As a reminder, the flag format is PWNME{city1_city2_city3} Cities in lowercase and in alphabetical order, separated by an "_".

The flag is therefore: `PWNME{gols_kaunas_terespol}`

## Tips & Tricks

- Google dork: `site:forum-train.com jdthetraveller`
- Try searching with other search engines like [Bing](https://www.bing.com/), [DuckDuckGo](https://duckduckgo.com/), [Qwant](https://www.qwant.com/), or [Yandex](https://yandex.com/).
- A location can be specified using GPS coordinates, address, postal code, city name, country name, etc. However, it is also possible to divide the globe into 3m² squares and specify a location using three words. **This is the concept of [what3words](https://what3words.com/).**
- To track Euro bills, you can use the website https://www.eurobilltracker.com/.
- All train departures/arrivals in Europe can be found on the website https://direkt.bahn.guru/.