---
title: Make Your Own Dark/Deep Web Empire
author: cotes
date: 2019-08-08 11:33:00 +0800
categories: [Blogging, Demo]
tags: [typography]
pin: true
---

<<<<<<< HEAD
> Warning: This writeup does not promote or encourage any illegal activity or harm to anyone, this write is only for Educational and Research Purposes
=======
This post is to show Markdown syntax rendering on [**Chirpy**](https://github.com/cotes2020/jekyll-theme-chirpy/fork), you can also use it as an example of writing. Now, let's start looking at text and typography.

## Headings

# H1 - heading
{: .mt-4 .mb-0 }

## H2 - heading
{: data-toc-skip='' .mt-4 .mb-0 }

### H3 - heading
{: data-toc-skip='' .mt-4 .mb-0 }

#### H4 - heading
{: data-toc-skip='' .mt-4 }

## Paragraph

Quisque egestas convallis ipsum, ut sollicitudin risus tincidunt a. Maecenas interdum malesuada egestas. Duis consectetur porta risus, sit amet vulputate urna facilisis ac. Phasellus semper dui non purus ultrices sodales. Aliquam ante lorem, ornare a feugiat ac, finibus nec mauris. Vivamus ut tristique nisi. Sed vel leo vulputate, efficitur risus non, posuere mi. Nullam tincidunt bibendum rutrum. Proin commodo ornare sapien. Vivamus interdum diam sed sapien blandit, sit amet aliquam risus mattis. Nullam arcu turpis, mollis quis laoreet at, placerat id nibh. Suspendisse venenatis eros eros.

## Lists

### Ordered list

1. Firstly
2. Secondly
3. Thirdly

### Unordered list

- Chapter
  + Section
    * Paragraph

### ToDo list

- [ ] Job
  + [x] Step 1
  + [x] Step 2
  + [ ] Step 3

### Description list

Sun
: the star around which the earth orbits

Moon
: the natural satellite of the earth, visible by reflected light from the sun

## Block Quote

> This line shows the _block quote_.

## Prompts

> An example showing the `tip` type prompt.
{: .prompt-tip }

> An example showing the `info` type prompt.
{: .prompt-info }

> An example showing the `warning` type prompt.
{: .prompt-warning }

> An example showing the `danger` type prompt.
>>>>>>> 717659d66ef1f68505417fbc8498b88bdbbb1f09
{: .prompt-danger }


Dark/Deep Web is the place where all badest things happens i would not mention anything here also there are some places on dark web where you can learn so many fasinating things you will be able to find Scintific Research, Classified Documents, Free EBooks, Advanced Hacking WriteUps and other illegel things

in this post i will share how we can create a Dark/Deep Web page and host it from our local server you can also buy a Hosting Provider but i gonna use a mine old laptop as a Server which the laptop has enogh space and ram to run this server with faster internet, so let's get started with technical stuff

Specifications of My Old Laptop:
- Intel i5-7200 Processor
- 20 Gigs Ram 
- 500 Gigs SSD
- 2 Gigs MX130 GPU
- Ubuntu 22.04 Os

## Prerequisits:
- Linux Os (Ubuntu 22.04) or any
- Tor Service & Tor Broswer Installed
- Python 3 Installed
- VSCode or any Editor/IDE
- HTML/CSS Knowledge

## Installation
first Download and install Python 3 by using `apt get install python-3` or if you're on mac then `brew install python-3` well i am using python http server to run our page locally but you can use anything Node, RubyOnRails, Django, Flask it's upto you

after python installed we need to download `Tor Service` and `Tor Browser` on our machine, to download it simply type `sudo apt install tor tor-browser`

after both of them installed we gonna write some html code for our page this one also upto you means you can write any code according to your will, i just gonna create simple html with image and heading

the code should look like this

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <b>Welcome To My Depp/Dark Web Empire</b>
    <i>You Are Being Hunted By FBI</i>
</body>
</html>
```

Now Open terminal And Type `python -m http.server 8000` and make sure the to start the server where index.html is located

Now Open brower and type your local ip address like this `127.0.0.1:8000/index.html`

We Can See That we got what we want

```
Welcome To My Deep/Dark Web Empire
You Are Now On Depp/Dark Web
```


Creating and hosting your own .onion service (a Tor hidden service) using Python involves several steps. It's important to note that this guide will provide a high-level overview, and you should ensure you're using this technology responsibly and legally. Here's a simplified tutorial to help you get started:

**Step 1: Install Required Software**

Before you start, you'll need to have a few things installed:

- **Python**: You should have Python installed on your server. Python 3 is recommended.

- **Tor**: Install the Tor service on your server. On Linux, you can use your package manager. For example, on Debian-based systems:

  ```
  sudo apt-get install tor
  ```

**Step 2: Configure Tor**

1. Open the Tor configuration file for editing:

   ```
   sudo nano /etc/tor/torrc
   ```

2. Add the following lines to the configuration file:

   ```
   HiddenServiceDir /var/lib/tor/myonionsite/
   HiddenServicePort 80 127.0.0.1:80
   ```

   This will configure your .onion service to listen on port 80 and route requests to your local web server (localhost:80).

3. Save the configuration file and restart Tor:

   ```
   sudo service tor restart
   ```

4. After the service restarts, you can find your .onion address in the `/var/lib/tor/myonionsite/hostname` file.

**Step 3: Create a Simple Python Web Server**

You can create a basic Python web server using the `http.server` module. Here's a simple script to serve content:

```python
import http.server
import socketserver

PORT = 80

Handler = http.server.SimpleHTTPRequestHandler

with socketserver.TCPServer(("", PORT), Handler) as httpd:
    print("Serving at port", PORT)
    httpd.serve_forever()
```

Save this script, for example, as `web_server.py`.

**Step 4: Start Your Web Server**

Run your Python web server script:

```
python web_server.py
```

This will start a web server that listens on port 80.

**Step 5: Access Your .onion Service**

Visit your .onion address (found in the `hostname` file) using the Tor Browser. You should be able to access the content served by your Python web server.

Remember that this is a very basic setup, and for a production .onion service, you would need to consider security, performance, and the nature of the content you're hosting. Always be aware of legal and ethical considerations when using Tor and hosting .onion services. This tutorial is for educational purposes and should be used responsibly.


