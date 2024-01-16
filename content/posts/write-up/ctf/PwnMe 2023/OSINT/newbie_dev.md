---
title: "[Easy][PwnMe 2023][OSINT] Newbie Dev"
date: "2023-05-07"
categories: [ "CTF", "PwnMe 2023 CTF", "OSINT" ]
slug: "newbie-dev"
ShowToc: false
draft: false
weight: 2
---

## Introduction

This challenge is an OSINT challenge from PWNME CTF.

### Context Explanation

To understand the context of the challenge, refer to the introduction challenge description.

### Task

As a budding developer, find information about this undeveloped passion.

## Solution

On John Droper's Facebook profile, we can see that he has a username: `jdthetraveller`.

We keep this in mind for later.

![Facebook profile](/images/pwnme2023/osint/john_droper_facebook_details.png)

On his feed, we have information about his username. He says that he chose it on [AFNIC](https://www.afnic.fr/), which is a domain name registry solution.

![Facebook profile](/images/pwnme2023/osint/john_droper_facebook_profile.png)

Trying to access https://jdthetraveller.fr/ leads to a construction blog likely owned by John Droper.

At this point, we do not have much information, so we will try to find information on the server with dirsearch.

![Dirsearch](/images/pwnme2023/osint/jdthetraveller_dirsearch.png)

We find a `/.git` directory that contains a `config` file with information about the git repository.

We can retrieve this file with `wget http://jdthetraveller.fr/.git/config`.

```bash
┌──(root㉿kali)-[~]
└─# wget https://jdthetraveller.fr/.git/config
--2023-05-07 23:46:11--  https://jdthetraveller.fr/.git/config
Resolving jdthetraveller.fr (jdthetraveller.fr)... 13.48.131.55
Connecting to jdthetraveller.fr (jdthetraveller.fr)|13.48.131.55|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 271 [application/octet-stream]
Saving to: ‘config’

config                          100%[=======================================================>]     271  --.-KB/s    in 0s

2023-05-07 23:46:11 (3.60 MB/s) - ‘config’ saved [271/271]


┌──(root㉿kali)-[~]
└─# cat config
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = https://github.com/droperkingjohn/myOwnWebsite.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main
```

First, there is a new nickname belonging to John Droper: `droperkingjohn`. We keep it in mind for the future.

We also find a link to a git repository: https://github.com/droperkingjohn/myOwnWebsite

![Github](/images/pwnme2023/osint/droperkingjohn_github.png)

If we look through the commits, we find this one:

![Github](/images/pwnme2023/osint/droperkingjohn_github_flag.png)

Flag: `PWNME{W0w_th15_l00k_l1ke_4n_e4sY_Fl4G}`

## Tips & Tricks

- Use **dirsearch** to find hidden directories.
- In a **.git** directory, you can retrieve information about the Git repository using the **config** file.
- Use **git log** to view the commits in a Git repository.
- Use **git show** to view the contents of a commit.
- Examine the different commits and their comments to find information.