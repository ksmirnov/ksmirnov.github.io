---
layout: post
title:  PySpark... Hadoop... Docker?
date:   2017-10-17 11:41:00 -0800
description: A guide for getting Docker cluster ready for PySpark jobs
img: terminal.jpg
tags: [Blog, Unix, IT, Development, Terminal]
author: # Add name author (optional)
---
PySpark... Hadoop... Docker... Looking at the title some may ask what a game have you been up to? Well, some time ago I 
was pulled in a project with pretty exciting stack of technologies - Google Tensorflow, Apache Spark, Cassandra etc. And
though my role in this project wasn’t mission critical, I had a chance to dive in and play with all those things. The 
project’s outcome is a visual search engine, that seeks for similar images in a catalog. It leverages multiple 
applications for different tasks. One of these parts is responsible for images vectorization and preparing data for 
Cassandra with a help of Tensorflow and PySpark. I can’t say I took a significant part in this application’s 
development, but I had a necessity to run it, optimize cluster’s parameters, check output data and etc. That was the 
moment when I came to the idea of setting my local Hadoop cluster. Mostly for convenience. And that was the moment when 
I actually stuck. It didn’t turn out to be as easy as it looked at first glance and I became almost obsessed with the 
idea of getting everything done in a convenient way. Now, when I managed to set it up, I’d like to share my experience 
in case if anyone has a similar task.

First of all, I needed a script or a manager to install a particular stack on a cluster. In my case these elements were:
* Apache Hadoop HDFS with YARN resource manager
* Apache Spark
* Python 2.7
* Any UI over Hadoop

There are multiple solutions available at the market, but I chose Apache Ambari, because I didn’t have enough time to do
a comprehensive analysis and comparison of all Hadoop distributives and their managers. Also Ambari was more or less 
familiar to me, so I decided to give it a try. Next, operating system. It didn’t take much time to choose and I settled 
down on CentOS. There are many reasons for CentOS, I don’t want to emphasize it, but the most meaningful one is that 
this OS is quite popular in corporate IT and thus it’s better in terms of support. The only additional restriction about
OS was its version - I had to use CentOS 7, because I needed Python 2.7. CentOS 6 supports and relies to Python 2.6 and
installing custom Python would be another issue for me.

The first problem I faced was a necessity to have [systemd](https://en.wikipedia.org/wiki/Systemd) installed to run such
services as *sshd* and *ntpd*. By default CentOS 7 doesn't have systemd and requires some customization, but luckily for
me there's a systemd enabled image already available at Docker's repository:
>FROM centos/systemd

Next, I needed to install some essentials to get everything work:
>RUN yum install -y --quiet epel-release \
 && yum -y --quiet update \
 && yum install -y --quiet \
    bzip2 \
    initscripts \
    ntp \
    openssh-clients \
    openssh-server \
    sudo \
    wget

Now, when all the essentials are installed, it's time to work on SSH. Ambari needs a key-less SSH access, so let's 
generate a key, authorize it and get a container ready for an intervention:
>RUN echo 'root:hortonworks' | chpasswd \
  && ssh-keygen -t rsa -f ~/.ssh/id_rsa -P '' \
  && cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys \
  && sed -i '/pam_loginuid.so/c session    optional     pam_loginuid.so'  /etc/pam.d/sshd \
  && echo -e "Host *\n StrictHostKeyChecking no" >> /etc/ssh/ssh_config

And the last few steps in a Docker file to install Ambari on a container, expose some ports and run systemd:
>RUN wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.5.2.0/ambari.repo \
>    -O /etc/yum.repos.d/ambari.repo \
> && yum repolist \
> && yum install -y ambari-server
>
>EXPOSE 22 8080 8081 8082 8083 8084 8085 8086 8087 8088
>
>CMD ["/usr/sbin/init"]