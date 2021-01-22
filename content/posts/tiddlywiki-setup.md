---
title: Setup of Personal Non-linear Notebook TiddlyWiki Server
date: '2019-08-13'
---

# TiddlyWiki

[TiddlyWiki](https://tiddlywiki.com/) is a note-taking app for managing complex information. It has two features that I really like:

## Philosophy of Tiddlers

I think people have two purposes for taking notes. One is to deepen the understanding of knowledge by means of the output process of taking notes, and the other is to facilitate reuse of this knowledge in the future. The former is a temporary role, but the notes for this purpose need not be saved, while the latter is very important, and a relatively complete form is needed to organize the knowledge for later reference. TiddlyWiki uses Tiddler to record meaningful and indivisible information, and to express deeper information through the connection between them, so as to record and retrieve knowledge at any level. As its official website says:

> The purpose of recording and organising information is so that it can be used again. The value of recorded information is directly proportional to the ease with which it can be re-used.
>
> The philosophy of tiddlers is that we maximise the possibilities for re-use by slicing information up into the smallest semantically meaningful units with rich modelling of relationships between them. Then we use aggregation and composition to weave the fragments together to present narrative stories.
>
> TiddlyWiki aspires to provide an algebra for tiddlers, a concise way of expressing and exploring the relationships between items of information.

## Self-contained

TiddlyWiki is a complete application with two main manifestations, one is a separate single html file, and the other is a directory containing many Tiddler files and meta information. Single html document includes all the logic and data, program directory mode requires node tiddlywikias a server. The program itself is open source, and it gives all the data management rights to the user, you only need a browser to access the data. Unlike other online services, even after a few decades --- as long as Web technology is still there, you can continue to use the entire application.

# Derver Deployment

TiddlyWiki can be used in a variety of ways, as a single html, combined with TiddlyDesktop for saving, and using the pages service on the public network, or using a server program to manage a directory. In the actual use process, the single html experience is not good, because html does not have permission to overwrite the data when saving the data, can only be saved by downloading new files or using other applications, and will also be downloaded when accessing on the network. The file is too big a problem. The following mainly describes how to deploy a directory through the server serve.

## NodeJS Environment

TiddlyWiki is written by js. The required runtime environment is node. Install node under ubuntu:

```bash
# install package manager
apt install npm
# instal node version manager
npm install -g n
# update nodejs to stable version
n stable
# change the registry
npm config set registry http://registry.npm.taobao.org/
```

## TiddlyWiki Server

After installing node, use the following command to install the TiddlyWiki server program:

```bash
sudo npm install -g tiddlywiki
```

### Create a New TiddlyWiki Folder

```bash
tiddlywiki mywiki --init server
```

A new directory called mywiki will be created in the current directory to store data for all wikis. use

```bash
tiddlywiki mywiki --listen
```

The server side of TiddlyWiki will be launched and accessible in the browser.

For a detailed explanation of the instructions, refer to the [official website](https://tiddlywiki.com/#WebServer) . There is also a more detailed explanation in the official [Github repository](https://github.com/Jermolene/TiddlyWiki5#installing-tiddlywiki-on-nodejs) .

### Verification

One problem with using the server and putting it on the public network is that anyone can modify your wiki. We hope that users who can log in with their username and password can modify it, but users who are not logged in can only view it. [This feature](https://tiddlywiki.com/#WebServer%20Authorization) was added in [version 5.1.18](https://tiddlywiki.com/#Release%205.1.18) . Create a new file, users.csv, containing the username and password of all users, in the form:

```csv
username,password
jane,do3
andy,sm1th
roger,m00re
```

When the server is started, add `--listen credentials="/path/to/users.csv" "readers=(anon)" writers=jane,andy,roger` can achieve the goal of anonymous view, and user modify.

### Nginx, HTTPS

If you want to deploy TiddlyWiki in a subdirectory (that is, not listening to port 80 by TiddlyWiki), you need to use Nginx as a reverse proxy to forward the request to the TiddlyWiki server. Let's take my service as an example. My homepage is ssine.cc. If I want TiddlyWiki to run on the wiki.ssine.cc domain name, I need to forward the wiki.ssine.cc request from Nginx to the port where TiddlyWiki is located.

Modified `/etc/nginx/sites-available/wiki.ssine.cc.conf` ：

```conf
server {
    listen 80;
    listen [::]:80;

    server_name wiki.ssine.cc;
    root /var/www/ssine.cc;

    index index.html;
    location / {
            try_files $uri $uri/ =404;
    }
}
```

Then restart Nginx and activate this route:

```bash
sudo ln -s /etc/nginx/sites-available/wiki.ssine.cc.conf /etc/nginx/sites-enabled/wiki.ssine.cc.conf
sudo nginx -t
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

Obtain an HTTPS certificate:

```bash
sudo certbot --rsa-key-size 4096 --nginx
```

Then modify the Nginx settings to enable the reverse proxy:

```conf
server {
    server_name wiki.ssine.cc;
    root /var/www/ssine.cc;

    index index.html;

    location / {
        proxy_pass http://localhost:1004/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
        client_max_body_size 0;

        access_log /var/log/nginx/wiki.access.log;
        error_log /var/log/nginx/wiki.error.log;
    }

    listen [::]:443 ssl http2; # managed by Certbot
    listen 443 ssl http2; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/nextcloud.dennisnotes.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/nextcloud.dennisnotes.com/privkey.pem; # managed by Certbot
    ssl_trusted_certificate /etc/letsencrypt/live/nextcloud.dennisnotes.com/chain.pem;
    include /etc/nginx/snippets/ssl.conf;
}
```

Then repeat the above steps, restart Nginx, you can forward the traffic of wiki.ssine.cc to the local 1004 port.

Finally, start TiddlyWiki Server on port 1004, the directory is tw :

```bash
tiddlywiki tw --listen host=127.0.0.1 port=1004 credentials="/path/to/users.csv" writers=sine,jill
```

You can also use the nohup directive to have the service run in the background:

```bash
nohup tiddlywiki tw --listen host=127.0.0.1 port=1004 credentials="/path/to/users.csv" writers=sine,jill > log.txt 2>&1 &
```

# Settings

Here are some of the adjustments to TiddlyWiki in everyday use.

## Add CSS

The way to add your own CSS is simple. Create a new Tiddler, set Type to text/css, and add a tag of $:/tags/Stylesheet to take effect.

### Centering Story River

A list of all Tiddlers is displayed, called Story River, and a column showing the title is called Side Bar. There are two layout modes: the fixed story river and the fixed sidebar, but the widescreen display will be very biased if the story river is fixed. If the sidebar is fixed, the story river is too wide to read a line. So manually modify the CSS so that the story river width is fixed and centered, and the sidebar is on the right side of the story river:

```css
@media (min-width: {{$:/themes/tiddlywiki/vanilla/metrics/sidebarbreakpoint}}) {

  html .tc-page-container {
    text-align: center;
  }

  html .tc-story-river {
    position: relative;
    width: 712px;
    /*width:1300px;*/
    padding-right: 0px;
    padding-left: 0px;
    margin: 0 auto;
    text-align: left;
  }

  html .tc-sidebar-scrollable {
    text-align: left;
    left: 50%;
    right: 0;
    margin-left: 350px;
  }

  html .tc-tiddler-frame {
    width: 100%;
  }
}
```

### Background Color of Selected Text

Change the background of the selected text to black, nothing else but feeling cooler 😎 .

```css
::selection {
    background:#333;
    color:#fff;
}

::-moz-selection {
    background:#333;
    color:#fff;
}

::-webkit-selection {
    background:#333;
    color:#fff;
}
```

### Tiddler Content Styling

Modify the indentation of the list, the default indentation is too much, looking uncomfortable:

```css
ul{
  padding-left:0px;
  margin-left:25px;
}

ol{
  padding-left:0px;
  margin-left:25px;
}
```

Text spacing fine-tuning, heading style, horizontal line style:

```css
.tc-story-river {
  word-spacing: 0.1em;
}

.tc-story-river h1 {
  text-align: center;
}

.tc-story-river h2 {
  text-align: center;
}

.tc-story-river h3 {
  text-decoration: underline;
}

hr {
  margin:0 auto;
  border: 0;
  height: 1px;
  background: #333;
  background-image: linear-gradient(to right, #333, #A349A4, #333);
}
```

### New Font

Many good-looking fonts are not safe, which means that there may not be on the client. In order to ensure the display effect, font-face is used to add fonts. When there is no font on the client, it will be downloaded at the address declared by font-face.

```css
@font-face
{
font-family: Constantia;
src: url('https://source-bed.oss-cn-beijing.aliyuncs.com/fonts/constan.ttf');
}

@font-face
{
font-family: Constantia;
src: url('https://source-bed.oss-cn-beijing.aliyuncs.com/fonts/constanb.ttf');
font-weight:bold;
}

@font-face
{
font-family: Constantia;
src: url('https://source-bed.oss-cn-beijing.aliyuncs.com/fonts/constani.ttf');
font-style:italic;
}
```

This will allow you to use Constantia fonts in your settings.

## Plugins

The main function of the plugin is to handle the rendering of Tiddler text, such as rendering the latex math formula, or converting the flow of the text description into svg. The way to install the plugin is very simple, drag its link to the TiddlyWiki interface and import it.

### Latex

Although the latex parser commonly used on the Web side is Mathjax, the more complete plugin on TiddlyWiki is only KaTeX. I also spent a lot of effort to port the Mathjax plugin that only supports single page mode to the server. Later, I think I still use KaTeX. It’s too much of a hassle to maintain a plugin yourself.

### Mermaid

I want to draw a flowchart here. There is already a plugin called Mermaid.js but the layout is very ugly. It also took a while. Later, when I was writing a formal language automaton note, I wanted to be able to display Tiddler's links on nodes and edges, and I did a lot of Ugly Hack. Now that I have forgotten the grammar of that time, the code I wrote is a bit too lazy to look at. It is also a trade-off, but it has a lot of functions, but it needs to remember more grammar to maintain some things, or it can be simple in function but simple in people. I plan to port a viz.js later, because the Graphviz layout is great, but the js file is a bit large and should be carefully considered. However, the main factor in whether or not to use this build tool is whether it needs to be modified in the future, otherwise it will be fine to upload the image locally and upload it.

## Macro

TiddlyWiki also supports macro definitions, which can be used to expand a macro (similar to a function) into a piece of text. The way to create a macro is to create a new Tiddler, set the Type to application/javascript, and add filed: module-type, value: macro at the bottom Add a new field.

Use when calling in other Tiddlers use `<<macro-name param1 param2 ...>>` .

### Embedding SVG

Embedding svg into html is cool, because the text in the image can even be selected, because it is often used as a macro:

```javascript
(function(){

/*jslint node: true, browser: true */
/*global $tw: false */
"use strict";

/*
Information about this macro
*/

exports.name = "svg";

exports.params = [
  {name: "src"},
    {name: "width"}
];

/*
Run the macro
*/
exports.run = function(src, width) {
  if(width[width.length-1] == "%") return '<center><embed src="' + src + '" style="width:' + width + ';" type="image/svg+xml" /></center>';
    else
  return '<center><embed src="' + src + '" style="width:' + width + 'px;" type="image/svg+xml" /></center>';
};

})();
```

The argument is the url and width of svg, which is used like this:

```tid
<<svg "https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/8086general-regs.svg" 450>>
```

### Prevent image caching

Some external images will be cached for a long time. If you want to re-request the image every time you open the page, you can use the following macro:

```javascript
(function(){

/*jslint node: true, browser: true */
/*global $tw: false */
"use strict";

/*
Information about this macro
*/

exports.name = "nc-img";

exports.params = [
  {name: "url"}
];

/*
Run the macro
*/
exports.run = function(url) {
  return url + "?" + new Date().getTime();
};

})();
```

The principle is to add `? + random number` to the image address, so the browser will consider another image to re-request.

How to use (the only place to use is to update the Project Euler's badge XD):

```tid
<img src=<<nc-img url:"https://projecteuler.net/profile/sineLiu.png">>/>
```
