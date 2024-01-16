---
title: "[Easy][HeroCTF v5][Web] Referrrrer"
date: "2023-14-13"
categories: [ "CTF", "HeroCTF v5", "Web" ]
tags: [ "Configuration", "Referer", "NGINX", "Express" ]
slug: "referrrrer"
ShowToc: false
draft: false
weight: 2
---

## Introduction

Bypass the security of a website that implements [Referer](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referer)-based authentication.

The challenge sources are available on my GitHub: [Referer Sources](https://github.com/mxcezl/HeroCTF-v5-Sources/tree/main/Web/Referrrrer/Referrrrer%20Sources)

## Solution

By looking at the challenge sources, we find two folders: `app` and `nginx`.

In the `nginx` folder, we find an `nginx.conf` file that contains the server's configuration.

```nginx
worker_processes auto;

events {
    worker_connections 128;
}

http {
    charset utf-8;

    access_log /dev/stdout;
    error_log /dev/stdout;

    upstream express_app {
        server app:3000;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://express_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location /admin {
            if ($http_referer !~* "^https://admin\.internal\.com") {
                return 403;
            }

            proxy_pass http://express_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

We can see that there is a `/admin` location that checks if the `Referer` header starts with `https://admin.internal.com`. If it doesn't, a 403 error is returned.

On the `app` side, there is an `index.js` file that contains the server's code.

```javascript
const express = require("express")
const app = express()


app.get("/", (req, res) => {
    res.send("Hello World!");
})

app.get("/admin", (req, res) => {
    if (req.header("referer") === "YOU_SHOUD_NOT_PASS!") {
        return res.send(process.env.FLAG);
    }

    res.send("Wrong header!");
})

app.listen(3000, () => {
    console.log("App listening on port 3000");
})
```

These are the only files that are useful to solve the challenge.

We understand that the goal of the challenge is to access the `/admin` endpoint to retrieve the flag.

There are two checks to pass:

- In `nginx.conf`: the `$http_referer` variable must start with `https://admin.internal.com`.
- In `index.js`: the `req.header("referer")` variable must be equal to `YOU_SHOUD_NOT_PASS!`.

By taking a closer look at the `Referer` header, we find [this page](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referer) that explains that this header is sent by the browser when clicking on a link.

We also learn that "referer" is a misspelling of "referrer," which is why it is written with a single "r."

![Referer Header Documentation](/images/heroctfv5/web/referer_mozilla_misspelling.png)

In nginx, `$http_referer` is a variable that contains the value of the `Referer` header sent by the browser.

However, for `index.js`, `req.header("referer")` is a function that does not work like the other getters.

On the [official documentation of `req.get(...)` in Express v4.x](http://expressjs.com/en/4x/api.html#req.get), we learn that this function returns the value of the **`Referer` or `Referrer`** header sent by the browser.


![Explication Referer header](/images/heroctfv5/web/referrrrer_express_req_get_explain.png)

![Documentation of req.get(...) in Express v4.x](/images/heroctfv5/web/referrrrer_express_req_get.png)

This is where the vulnerability of the challenge lies.

We can craft a request that contains the `Referrer` header with the value `YOU_SHOUD_NOT_PASS!` and the `Referer` header with the value `https://admin.internal.com`.

```bash
curl 'http://static-01.heroctf.fr:7000/admin' -H 'Referer: https://admin.internal.com/' -H 'Referrer: YOU_SHOUD_NOT_PASS!'
Hero{ba7b97ae00a760b44cc8c761e6d4535b}
```

Flag: `Hero{ba7b97ae00a760b44cc8c761e6d4535b}`

## Alternative Solution

It is possible to bypass the nginx check by exploiting the fact that Express routes are case-insensitive while nginx routes are case-sensitive.

Accessing the `/Admin` route will not trigger the nginx check on `$http_referer` since the nginx route is `/admin`.

Express, on the other hand, does not differentiate between `/admin` and `/Admin` and triggers the check on `req.header("referer")`.

The Express check passes if `req.header("referer")` is equal to `YOU_SHOUD_NOT_PASS!`.

```bash
curl -i -H 'Referer: YOU_SHOUD_NOT_PASS!' http://static-01.heroctf.fr:7000/Admin

HTTP/1.1 200 OK
Server: nginx/1.24.0
Date: Sun, 14 May 2023 13:23:11 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 38
Connection: keep-alive
X-Powered-By: Express
ETag: W/"26-Cj1P1GdO8Vke/DfJFC3B2cH95nw"

Hero{ba7b97ae00a760b44cc8c761e6d4535b}
```

## Tips & Tricks

- The header `Referer` is a misspelling of `Referrer` ([source](https://en.wikipedia.org/wiki/HTTP_referer)).
- `$http_referer` is an nginx variable that contains the value of the `Referer` header sent by the browser.
- In Express v4.x, calling the function `req.get("referer")` returns the value of the **`Referer` or `Referrer`** header sent by the browser.
- Route names exposed with Express v4.x are case-insensitive; `/Admin` and `/admin` are equivalent ([source](http://expressjs.com/en/api.html)).
  - `CaseSensitive: Disabled by default, treating “/Foo” and “/foo” as the same.`
- Route names in the nginx `location` configuration are case-sensitive; `/Admin` and `/admin` are not equivalent.