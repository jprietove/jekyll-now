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

## Giving the server access to the proxy through SSH

Now I redirect the proxy port to the server through SSH:

```
user@laptop:~$ ssh -f -N -n -R8888:127.0.0.1:8888 serveruser@192.168.1.99
```

Options explained:

* -f: ssh goes to the background

* -N: Do not execute a remote command.  This is useful for just forwarding ports.

* -n:  Redirects stdin from /dev/null (actually, prevents reading from stdin).
 This must be used when ssh is run in the background.

* -R8888:127.0.0.1:8888 (port:host:hostport): redirects server port 8888 to
laptop (host) hostport 8888

* serveruser@192.168.1.99: server user name and IP Address

## Accessing the server and configuring APT for use the proxy

Let's access the server:

```
user@laptop:~$ ssh serveruser@192.168.1.99
```

And create the following file:

```
serveruser@server:~$ sudo nano /etc/apt/apt.conf.d/12proxy
```

With the following content:

```
Acquire::http::Proxy "http://localhost:8888";
```

## Update the server

If everything is OK, now we can update the server:

```
serveruser@server:~$ sudo apt-get update
...
serveruser@server:~$ sudo apt-get upgrade
```

## When finished...

When we finish, just kill the SSH process that is redirecting the 8888 port:

```
user@laptop:~$ ps -aux | grep 8888
user      5307  ... ssh -f -N -n -R8888:127.0.0.1:8888 serveruser@192.168.1.99
user@laptop:~$ kill -9 5307
```
