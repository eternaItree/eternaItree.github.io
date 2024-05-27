---
layout: single
title: "SSH Tunnel"
date: 2024-01-24
classes: wide
header:
  teaser: /assets/images/avatar.jpg
tags:
  - ssh
  - tunneling
---

## Introduction 
Introduces various ssh tunnels, covering 3 main configurations: local port forwarding, remote port forwarding, and dynamic port forwarding (SOCKS proxy)

## local port forwarding (ssh tunnel)

### local-side configuration

```jsx
ssh -f -N -L [local_addr:]local_port:remote_addr:remote_port [user@]sshd_addr

-f means run background
-N means no more commands to be executed on the remote server when establishing a tunnel

e.g. ssh -f -N -L 9001:remote_addr:9002 root@remote_addr
```

### server-side configuration

```jsx
echo "AllowTcpForwarding yes" | sudo tee -a /etc/ssh/sshd_config
sudo systemctl restart sshd
```

The **`ssh -L`** command establishes an SSH tunnel that forwards network traffic from a local address and port (local_addr:local_port) to a remote address and port (remote_addr:remote_port). With this tunnel in place, accessing the local address and port is equivalent to accessing the remote address and port. The network traffic flows from the local machine (localhost) to the remote machine, enabling seamless communication as if the services on the remote machine were running locally.

![Untitled](/assets/images/SSH%20Tunnel%20cdd86f5ce4d84bc1ac3b9ec664ddd460/Untitled.png)

Ref: [https://iximiuz.com/en/posts/ssh-tunnels/](https://iximiuz.com/en/posts/ssh-tunnels/#:~:text=Local%20port%20forwarding%20)

## remote port forwarding (ssh reverse tunnel)

### local-side configuration

```jsx
ssh -f -N -R [remote_addr:]remote_port:local_addr:local_port [user@]gateway_addr

ssh -f -N -R \*:8111:localhost:8008 root@remote_addr
```

### server-side configuration

```jsx
echo "AllowTcpForwarding yes" | sudo tee -a /etc/ssh/sshd_config
echo "GatewayPorts yes" | sudo tee -a /etc/ssh/sshd_config // or GatewayPorts clientspecified
sudo systemctl restart sshd

// GatewayPorts clientspecified: This allows remote port forwarding to bind to any address specified by the client. It enables you to specify the address when establishing the SSH connection for remote port forwarding.
// GatewayPorts yes: This allows remote port forwarding to bind to any address on the SSH server, making forwarded ports accessible from anywhere.
```

This is the reverse of local port forwarding, remote port forwarding is used when we want to expose the service from the local to outside world. Thus the network traffic flows from the remote_addr:port → local_addr:port. 

Note that the SSH server needs to be configured with the [`GatewayPorts yes`](https://linux.die.net/man/5/sshd_config#GatewayPorts) setting.

![Untitled](/assets/images/SSH%20Tunnel%20cdd86f5ce4d84bc1ac3b9ec664ddd460/Untitled%201.png)

Ref: [https://iximiuz.com/en/posts/ssh-tunnels/](https://iximiuz.com/en/posts/ssh-tunnels/#:~:text=Local%20port%20forwarding%20)

### dynamic port forwarding (socks proxy)

Unlike the local port forwarding or remote port forwarding, which forward the specified port, the dynamic port forwarding can forward any type of traffic that specified by users. 

It uses `SOCKS` protocol within the tunnel, within the tunnel, there is a socks proxy server that built on the local machine to route network traffic through the tunnel. 

```jsx
ssh -D 1234 christine@$ti    // start built-in socks server listening on port 1234

echo socks5 127.0.0.1 1234 | sudo tee -a /etc/proxychains4.conf // specify a socks server to proxy traffic when using proxychains

proxychains psql -U chrinstine -h localhost -p 5432 // proxychains will proxy this traffic through the socks tunnel to the ssh server and execute the command, which feels like executing on the ssh server.
```

![Untitled](/assets/images/SSH%20Tunnel%20cdd86f5ce4d84bc1ac3b9ec664ddd460/Untitled%202.png)

Ref: HTB Challenge Funnel 

## Appendix

### server proxy in vite.config.ts

this proxy configuration is to stipulate the 

### socks proxy

A SOCKS5 server is capable of **proxying any type of network traffic**. The reason for this is that SOCKS (**Socket Secure**) is a protocol that operates at the **transport layer of the TCP/IP model** and provides a flexible framework for proxying various types of network connections.Here are a few key reasons why SOCKS5 servers can handle different types of traffic, including HTTP, HTTPS, FTP, SMTP, POP3, SH, and many others.