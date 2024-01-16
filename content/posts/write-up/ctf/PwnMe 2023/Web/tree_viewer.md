---
title: "[Intro][PwnMe 2023][Web] Tree Viewer"
date: "2023-05-07"
categories: [ "CTF", "PwnMe 2023 CTF", "Web" ]
ShowToc: false
draft: false
weight: 1
---

## Introduction

This is a WEB challenge from the PWNME CTF.

### Context Explanation

Here, you can check the content present on the server.

### Prompt

Find a way to abuse this functionality and read the contents of the flag.txt file.

### Solution

![Tree viewer](/images/pwnme2023/web/tree_viewer_home.png)

This is the challenge homepage.

What I noticed immediately is the `<?= shell_exec('ls '.$parsed); ?>` present in the source code.

If we can control the `$parsed` variable, we can execute commands.

Thanks to the source code provided, we can see that the input is passed into the `preg_match_all` function, filtering out the characters `;` and `|`.

```php
<?php
$parsed = isset($_POST['input']) ? $_POST['input'] : "/home/";
if($illegals){
    echo "Illegals chars found";
    $parsed = "/home/";
}
```

Therefore, we can use the `&&` characters to chain commands after the `ls` command and thus read the contents of the `flag.txt` file.

![Tree viewer flag](/images/pwnme2023/web/tree_viewer_flag.png)

Payload: `&& cat /home/flag.txt`

Flag: `PWNME{US3R_1nPUT2_1n_ShELL_Y3S_6x8c}`

## Tips & Tricks

- `&&` can be used to chain commands, similar to `;` or `|`.
- Check which characters are filtered.
- Review the page source code.