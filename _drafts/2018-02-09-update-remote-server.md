---
layout: post
title: Providing temporary Internet Access to a server for updating
categories: ssh networking ubuntu proxy

---

I need to update a remote server, with private IP address, without Internet
Access. I can ssh to it through a VPN. How can I give the server temporary
Internet Access for updating it?

## Install tinyproxy

I will use my laptop as a HTTP proxy server. I will use tinyproxy:

```
laptop$ sudo apt install tinyproxy
```

When installed, tinyproxy automatically starts and listen to 8888 port.

## SSH to the server

Now I get SSH access to the server,
```
laptop$ ssh -f -N -n -R8888:127.0.0.1:8888 serveruser@192.168.1.99
```

Options explained:

* -f: ssh goes to the background

* -N: Do not execute a remote command.  This is useful for just forwarding ports.

* -n:  Redirects stdin from /dev/null (actually, prevents reading from stdin).
 This must be used when ssh is run in the background.

* -R8888:127.0.0.1:8888 (port:host:hostport): redirects server port 8888 to
laptop (host) hostport 8888

* serveruser@192.168.1.99: server user name and IP Address
