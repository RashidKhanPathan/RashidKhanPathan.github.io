---
title: Make Your Own Dark/Deep Web Site
author: RashidKhanPathan
date: 2019-08-08 11:33:00 +0800
categories: [WriteUp, Tutorial]
tags: [writing, writeup]
pin: true
toc: true
---

> This writeup is only for Educational Purposes, it is not encourgae or promote any illgel activity, if using this writeup anyone did something wrong then the author will not be held responsible for it even it about just setup
{: .prompt-danger }

Dark/Deep Web is the place where all badest things happens i would not mention anything here also there are some places on dark web where you can learn so many fasinating things you will be able to find Scintific Research, Classified Documents, Free EBooks, Advanced Hacking WriteUps and other illegel things

in this post i will share how we can create a Dark/Deep Web page and host it from our local server you can also buy a Hosting Provider but i gonna use a mine old laptop as a Server which the laptop has enogh space and ram to run this server with faster internet, so let's get started with technical stuff

## Specifications of My Old Laptop:
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

