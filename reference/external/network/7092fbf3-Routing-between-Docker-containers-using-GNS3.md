---
title: Routing between Docker containers using GNS3.
source: https://cciethebeginning.wordpress.com/2015/03/04/routing-between-docker-containers-using-gns3/
kind: external
domain: network
original_date: 2015-03-04
fetched_at: 2026-05-16
bookmark_title: Routing between Docker containers using GNS3. | CCIE, the beginning!
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[cciethebeginning.wordpress.com](https://cciethebeginning.wordpress.com/2015/03/04/routing-between-docker-containers-using-gns3/)
> 原始日期：2015-03-04
> 抓取日期：2026-05-16

# Routing between Docker containers using GNS3.

# Routing between Docker containers using GNS3.

March 4, 2015 3 Comments

The idea is to route (IPv4 and IPv6) between Dockers containers using GNS3 and use them as end-hosts instead of Virtual Machines.

Containers use only the resources necessary for the application they run. They use an image of the host file system and can share the same environment (binaries and libraries).

In the other hand, virtual machines require entire OS’s, with reserved RAM and disk space.


If you are not familiar with Docker, I urge you to take a look at the below excellent short introduction and some additional explanation from Docker site. :




As for now, Docker has limited networking functionalities. This is where pipework comes to the rescue. Pipework allows more advanced networking settings like adding new interfaces, IP’s from a different subnets and set gateways and many more…

To be able to route between the containers using your own GNS3 topology (the sky the limit!), pipework allows to create a new interface inside a running container, connect it to a host bridge interface, give it an IP/mask in any subnet you want and set a default gateway pointing to a device in GNS3. Consequently all egress traffic from the container is routed to your GNS3 topology.



### Lab requirements:

**Docker:**

https://docs.docker.com/installation/ubuntulinux/#docker-maintained-package-installation

**Pipework:**

sudo bash -c "curl https://raw.githubusercontent.com/jpetazzo/pipework/master/pipework\ > /usr/local/bin/pipework"

For each container, we will generate docker image, run a container with an interactive terminal and set networking parameters (IP and default gateway).

To demonstrate docker flexibility, we will use 4 docker containers with 4 different subnets:


- Two docker container instances based upon a special minimal ubuntu image modified for docker (https://github.com/phusion/baseimage-docker). On top of which (as separate image) we install some linux tools like wget, iperf, inetutils-traceroute, iputils-tracepath, mtr, dnsutils, sipp, pjsua. (https://github.com/AJNOURI/Docker-files/blob/master/phusion-dockerbase)
- A docker container instance based upon the first built image (1), on top of which (as separate image) we install apache (https://github.com/AJNOURI/Docker-files/blob/master/apache-docker)
- A docker container instance based upon a simple ubuntu, on top of which (as separate image) we install firefox (https://github.com/AJNOURI/Docker-files/blob/master/firefox-docker)
|


This is how containers are built for this lab:


.

.

Here is the general workflow for each container.

1- build image from Dockerfile (https://docs.docker.com/reference/builder/):
An image is readonly.
Spawn and run a writable container with interactive console. The parameters of this command may differ slightly for each GUI containers.
Create host bridge interface and link to a new interface inside the container, assign to it an IP and a new default gateway.
|


To avoid manipulating image id’s and container id’s for each of the images and the containers, I use a bash script to build and run all containers automatically:

https://github.com/AJNOURI/Docker-files/blob/master/gns3-docker.sh


#!/bin/bash IMGLIST="$(sudo docker images | grep mybimage | awk '{ print $1; }')" [[ $IMGLIST =~ "mybimage" ]] && sudo docker build -t mybimage -f phusion-dockerbase . [[ $IMGLIST =~ "myapache" ]] && sudo docker build -t myapache -f apache-docker . [[ $IMGLIST =~ "myfirefox" ]] && sudo docker build -t myfirefox -f firefox-docker . BASE_I1="$(sudo docker images | grep mybimage | awk '{ print $3; }')" lxterminal -e "sudo docker run -t -i --name baseimage1 $BASE_I1 /bin/bash" sleep 2 BASE_C1="$(sudo docker ps | grep baseimage1 | awk '{ print $1; }')" sudo pipework br4 -i eth1 $BASE_C1 192.168.44.1/24@192.168.44.100 BASE_I2="$(sudo docker images | grep mybimage | awk '{ print $3; }')" lxterminal -e "sudo docker run -t -i --name baseimage2 $BASE_I2 /bin/bash" sleep 2 BASE_C2="$(sudo docker ps | grep baseimage2 | awk '{ print $1; }')" sudo pipework br5 -i eth1 $BASE_C2 192.168.55.1/24@192.168.55.100 APACHE_I1="$(sudo docker images | grep myapache | awk '{ print $3; }')" lxterminal -t "Base apache" -e "sudo docker run -t -i --name apache1 $APACHE_I1 /bin/bash" sleep 2 APACHE_C1="$(sudo docker ps | grep apache1 | awk '{ print $1; }')" sudo pipework br6 -i eth1 $APACHE_C1 192.168.66.1/24@192.168.66.100 lxterminal -t "Firefox" -e "sudo docker run -ti --name firefox1 --rm -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix myfirefox" sleep 2 FIREFOX_C1="$(sudo docker ps | grep firefox1 | awk '{ print $1; }')" sudo pipework br7 -i eth1 $FIREFOX_C1 192.168.77.1/24@192.168.77.100


And we end up with the following conainers:


**GNS3**

All you have to do is to bind a separate cloud to each bridge interface (br4,br5,br6 and br7) created by pipework, and then connect them to the appropriate segment in your topology.


Note that GNS3 topology is already configured for IPv6, so as soon as you start the routers, Docker containers will be assigned IPv6 addresses from the routers through SLAAC (Stateles Auto Configuration) which makes them reachable through IPv6.


Here is a video on how to launch the lab:



### Cleaning up

To clean your host from all containers and images use the following bash script:

*https://github.com/AJNOURI/Docker-files/blob/master/clean_docker.sh *which uses the below docker commands:

Stop running containers:
*sudo***docker****stop**<container id’s from `**sudo docker ps**`>
*sudo***docker****rm**<container id’s from `**sudo docker ps -a**`>
*sudo***docker****rmi**<image id’s from `**sudo docker images**`>
|

sudo ./clean_docker.sh Stopping all running containers... bf3d37220391 f8ad6f5c354f Removing all stopped containers... bf3d37220391 f8ad6f5c354f Erasing all images... Make sure you are generating image from a Dockerfile or have pushed your images to DockerHub. *** Do you want to continue? No

I answered “No”, because I still need those images to spawn containers, you can answer “Yes” to the question if you don’t need the images anymore or if you need to change the images.


**References:**

Docker:

- http://youtu.be/aLipr7tTuA4
- https://docs.docker.com/introduction/understanding-docker/
- https://docs.docker.com/reference/builder/

pipework for advanced Docker networking:

Running firefox inside Docker container:

- http://fabiorehm.com/blog/2014/09/11/running-gui-apps-with-docker/
- https://blog.jessfraz.com/posts/docker-containers-on-the-desktop.html

Baseimage-Docker:

3D model shipping container:

- Humberto Ferreira at http://www.blender-models.com/model-downloads/objects/id/shipping-container/