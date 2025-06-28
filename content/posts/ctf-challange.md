---
title:  'CTF Challenge'
date:  2024-03-22T21:10:44+01:00
draft:  false
tags:
  - ctf
  - security
---
## Intro
Recently while browsing the internet I noticed a CTF challenge that was form of preselection to job interview. Generally I was not very interested with job in security field, but I wanted to try if I can solve any of the tasks without any preparation and generally I like challenges. I did some CTF's in the past but it was a couple years ago when I was at the university and I think I wasn't good at it. I solved only a couple from web app category, but other categories were usually to hard for me.
## Challenge
The challenge consists of 7 (or 8) tasks. I am not sure if 8th task is valid or not because it was commented out on the site with CTF and it the challenge description was mentioned that you need to send them 7 flags.
### 1 - Xcross-this
There was not much on the page. Only the input filed and some numbers. I noticed that I can right click on the numbers and it showed that page is using MathJax library. I started to look for known XSS vulnerabilities is this library and found that there is one with `\unicode{}` payload. I was trying different payloads, but I wasn't sure what I want to achieve. Then I went to the browser console and noticed following error `Uncaught ReferenceError: xss is not defined`, so I went to the source page and noticed some script:
```javascript
function $_GET(name) {
  const url = new URL(location.href);
  return url.searchParams.get(name) || '';
}

const query = $_GET('xss') || '\\frac 1 2';

// MathJax automatically renders text from within '$$' delimiters
latex.textContent = '$$' + query + '$$';
xss.value = query;

// Obsługuje kliknięcie w link
const flagLink = document.getElementById('flagLink');
flagLink.addEventListener('click', function(e) {
  e.preventDefault();
  fetch(this.getAttribute('href'))
    .then(response => response.json())
    .then(data => {
      if (data.flag) {
        alert(data.flag);
      } else {
        alert('Nie udało się pobrać flagi.');
      }
    })
    .catch(error => {
      console.error(error);
      alert('Wystąpił błąd.');
    });
});
```
So I set breakpoint to be able to define `xss` object before the exception. The script went on, but it was not able to find element with id `flagLink`. I thought that I can create one, but the problem is next the script tries to fetch URL form that element. So what URL should I set? The only clue was `<a href="?LogOut=1" id="LogOut"></a>` in page source code, so I changed id of that element, but I got the message that getting the flag failed. I looked at the response `{"error":"VVRGU1IxZ3pkRlJrU0VsM1ltMWtTbUpyTVdoa1IyZzJWMWM1TVZGWVNteG1VVzg5Q2c9PQo=\n"}` and I thought that is wrong way, so I started to look for different solution. After some time I went back to that idea and tried decode the response with base64, but it did not help. Without any hope I tried decode value again and again and it was it. The flag was encoded three times with base64.
### 2 - Cross-site
This task was commented out. I've been trying to solve this one but unfortunately I failed.
The purpose of this task was to get flag from file `get_flag.php` using a search engine that was vulnerable to XSS. On the page code I found the script that was fetching content of `get_flag.php` and inserting the value in hidden input filed. I was not able to figure out how to modify the request to get the flag. I also did not want to spend too much time on task while I was not sure if this task is valid or not.
This is the script from the task.
```javascript
document.addEventListener('DOMContentLoaded', () => {
  // Wykonaj zapytanie AJAX do skryptu PHP
  fetch('get_flag.php')
    .then(response => response.text())
    .then(flagValue => {
      // Tworzenie ukrytego elementu input i wstrzyknięcie flagi
      var secretCodeInput = document.createElement('input');
      secretCodeInput.type = 'hidden';
      secretCodeInput.className = 'secretcode';
      secretCodeInput.value = flagValue;

      // Dodanie elementu do ciała dokumentu
      document.body.appendChild(secretCodeInput);
    });
});
```
### 3 - Database
The task was to use SQL injection to get hidden job offer. I was just trying different SQL injection payloads and keep my fingers crossed. I noticed that I have to use `'` at the end of the payload or I get SQL syntax error. The syntax error was from SQLite, so I stared to look for vulnerabilities for that database, but I did not found anything useful. I also tried use comment sign `--` to affect the query, but it always gave me syntax error. After a while I found this payload in the internet at it worked; `?query=')+or+true--`. The hidden job offer appeared with flag.
### 4 - External Entity
I think this task was the easiest. At the top of the page there was following code snipped `$flag = @file_get_contents("/tmp/flag"); ` and below that there was text area with following XML
```xml
<creds>
    <user>admin</user>
    <pass>admin</pass>
</creds>
```
When you sent that XML, you got a message that you are successfully logged in and your username.
At that point I was not aware of any XML vulnerabilities, but I just google some XML exploits and found XML external entity injection which can be used to retrieve file. Thus it looked perfect. I just create payload with provided file path and username as defined entity.
```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///tmp/flag"> ]>
<creds>
    <user>&xxe;</user>
    <pass>admin</pass>
</creds>
```
And now instead of username we got the content of `/tmp/flag`
### 5 - deardir
The task was about to provide path to file as a parameter `file`. The page source code also contained the comment with path `/tmp/flag`. It looked very simple path traversal vulnerability. When I tried to provide following parameter `?file=/tmp/flag` server returned error message `include(/var/www/html//tmp/flag): failed to open stream: No such file or directory`, so it means that we are in `/var/www/html/` dir. Thus we have to move three dirs up and then we can get to the `/tmp/flag/` using following query `?file=../../../tmp/flag`.
### 6 - I'm brOken'
In this task was a login page with testing account. After the logging in, a session cookie was created with JWT, so the idea was to find vulnerability for JWT. The simplest exploit was to change algorithm type to `none` and then we could manipulate the cookie. But when I changed that I received message from server that this algorithm is not allowed, so I was looking for another vulnerability. I was searching for a some time, but without luck and then I found another article describing the `none` algorithm vulnerability. In this article I noticed that they are creating payload with multiple cases of `none` e.g. `None`, `nOnE` etc. So I tried this one and it worked. I was able to change username to admin, but flag did not appear, so I was trying different usernames, but every time the page was empty. Then I checked the page source code and noticed commented link to js file. In that file was the flag base64 encoded. After solving this, name of this task became obvious. :)
### 7 - Welcome, I, you
In this task there was information to use endpoint `/greeting/<name>`. I noticed that in HTTP reply there was `Server` header with following value: `Werkzeug/3.0.1 Python/3.8.18` and the only vulnerability for Python I knew was Jinja Template injection. I tried to send some simple template and see if it is evaluated by server or not. I sent `{{ 2 + 2}}` and the response was `Welcome 4!`. Thus I started to looking for payload that gives me the flag and I found this one `/greeting/{{request.application.__globals__.__builtins__.__import__('os').popen('cat flag.txt').read()}}`
### 8 - Why s0 deserious?
I liked this task the most. On the page there was information that this page is about deserialization and there is object of class `FileRead`. They also mentioned that the flag is in file `flag.txt` and you have to provide date to parameter `data` in PHP. At first I have no idea what should I do, but I started to looking for how to deserialize in PHP and I found that I have to create a string that represents an object. I used provided information and prepared following payload: `?data=O:8:"FileRead":1:{s:4:"file";s:17:"/var/www/flag.txt";}`. In response I got content of file with the flag.
## Conclusion
I think the tasks were quite easy for someone familiar with CTF's and for me they were just right. I writing this a few days after I solved these tasks and I think I should describe these tasks just after I solved it or even during solving, because I don't remember my exact thought process and what exactly I was trying to do, but it could be also interesting and informative.
