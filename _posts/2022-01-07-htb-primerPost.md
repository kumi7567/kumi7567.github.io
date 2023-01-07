---
layout: single
title: Ready - Hack The Box
excerpt: "Ready was a pretty straighforward box to get an initial shell on: We identify that's it running a vulnerable instance of Gitlab and we use an exploit against version 11.4.7 to land a shell. Once inside, we quickly figure out we're in a container and by looking at the docker compose file we can see the container is running in privileged mode. We then mount the host filesystem within the container then we can access the flag or add our SSH keys to the host root user home directory."
date: 2021-05-15
classes: wide
header:
  teaser: /assets/images/htb-writeup-ready/ready_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - infosec
tags:
  - linux
  - gitlab
  - cve
  - docker
  - privileged container
---

![](/assets/images/htb-writeup-ready/ready_logo.png)

Esto es una prueba para ver como se ve

````
echo "Que tal estas"

````
