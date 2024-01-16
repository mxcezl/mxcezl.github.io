---
title: "[Medium][PwnMe 2023][Web] Beat me!"
date: "2023-05-07"
categories: [ "CTF", "PwnMe 2023 CTF", "Web" ]
ShowToc: false
draft: false
weight: 3
---

## Introduction

This challenge is a WEB challenge from the PWNME 2023 CTF.

### Context explanation

A pro player challenges you in a new game. They have spent a lot of time on it and achieved an extremely high score.

### Directive

Your goal is to beat them by any means necessary.

## Solution

The challenge is a game where you must move a ship to avoid shots and fire at enemies.

![Beat me!](/images/pwnme2023/web/beat_me_game.png)

The player you must beat is Eteck, the challenge creator, who has a score of 1337420.

Of course, we won't tryhard to beat this score; instead, we'll try to find a flaw in the game.

When playing, at the end of the game, we can see that two requests are made:

- **POST /scores**: to send the score obtained at the end of the game.
- **GET /scores**: to retrieve the scoreboard.

![Beat me!](/images/pwnme2023/web/beat_me_game_end.png)

The payload sent to post the score is the following:

```json
{
    "name": "mxcezl",
    "score": 12,
    "signature": -1166920824
}
```

If we try to replay this request by modifying our score, we get an error indicating that the signature is not correct:

![Beat me!](/images/pwnme2023/web/beat_me_post_score_error.png)

Let's analyze the source code of the page to see how the signature is calculated.

On the home page there is nothing special, except for some imports of JavaScript scripts, including `/js/main.0acc8a51228c77e4a908.bundle.js`.

This JavaScript file is obfuscated, making it unreadable.

Before wasting time trying to deobfuscate it, we can try to debug the script in the browser to see where the signature calculation is done.

Searching for `signature` in the code, we come across this function:

![signature](/images/pwnme2023/web/beat_me_script.png)

We can see here that the signature value is calculated using the `_0x3f306f` function with `_0x5a84cd`.

We can imagine that `_0x3f306f` is the hash function, and `_0x5a84cd` is the score.

This is confirmed by playing a bit and analyzing the values during debugging:

![signature](/images/pwnme2023/web/beat_me_score_debug.png)

We can see in the top left of the screenshot that my score is 12.

Likewise, we can see that the variable `_0x5a84cd` is equal to 12 in the debug section on the right side of the screenshot.

Since we're debugging, we can modify the value of `_0x5a84cd` to see what happens.

![signature](/images/pwnme2023/web/beat_me_cheated_score.png)

Then, if we continue debugging, we can see that the score has been taken into account, and the signature is valid since the flag appears.

![signature](/images/pwnme2023/web/beat_me_flag.png)

Flag: `PWNME{ChE4t_oN_cLI3N7_G4m3_Is_Not_3aS1}`

## Tips & Tricks

- Debug the JavaScript code to see how data is processed.
- Use the debugger to modify values.
- Don't overlook obfuscated or minified JavaScript files; often, they contain valuable information.