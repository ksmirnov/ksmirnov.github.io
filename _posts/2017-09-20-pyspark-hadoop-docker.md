---
layout: post
title:  PySpark... Hadoop... Docker?
date:   2017-10-17 11:41:00 -0800
description: A guide for getting Docker cluster ready for PySpark jobs
img: elephanship.png
tags: [Blog, IT, Development, Spark, Hadoop, Docker]
author: # Add name author (optional)
---
# Introduction

PySpark... Hadoop... Docker... Looking at the title some may ask what a game have you been up to? Well, some time ago I 
was pulled in a project with pretty exciting stack of technologies - Google Tensorflow, Apache Spark, Cassandra etc. And
though my role in this project wasn’t mission critical, I had a chance to dive in and play with all those things. The 
project’s outcome is a visual search engine, that seeks for similar images in a catalog. It leverages multiple 
applications for different tasks. One of these parts is responsible for images vectorization and preparing data for 
Cassandra with a help of Tensorflow and PySpark. I can’t say I took a significant part in this application’s 
development, but I had a necessity to run it, optimize cluster’s parameters, check output data and etc. That was the 
moment when I came up with the idea of setting my local Hadoop cluster on Docker. Mostly for convenience. And that was
the moment when I actually stuck. It didn’t turn out to be as easy as it looked at first glance and I became almost
obsessed with the idea of getting everything done in a convenient way. Now, when I managed to set it up, I’d like to
share my experience in case if anyone has a similar task.

As a shortcut, here's a repository with a Dockerfile and a helper script:<br/>
[https://github.com/ksmirnov/ambari_docker]<br/>
And for those, who don't play "Im feeling lucky", I'm going to put some explanation below, starting from Dockerfile
definition and finishing with Ambari cluster configuration.

# Dockerfile

As an initial point, I needed a script or a manager to install a particular stack on a cluster. In my case these 
elements were:
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

The first problem I faced was a necessity to have [systemd] installed to run such
services as *sshd* and *ntpd*. By default CentOS 7 doesn't have systemd and requires some customization, but luckily for
me there's a systemd enabled image already available at Docker's repository:
{% highlight bash %}
FROM centos/systemd
{% endhighlight %}

Next, I needed to install some essentials to get everything work:
{% highlight bash %}
RUN yum install -y --quiet epel-release \
 && yum -y --quiet update \
 && yum install -y --quiet \
    bzip2 \
    initscripts \
    ntp \
    openssh-clients \
    openssh-server \
    sudo \
    wget
{% endhighlight %}

Now, when all the essentials are installed, it's time to work on SSH. Ambari needs a key-less SSH access, so let's 
generate a key, authorize it and get a container ready for an intervention:
{% highlight bash %}
RUN echo 'root:hortonworks' | chpasswd \
  && ssh-keygen -t rsa -f ~/.ssh/id_rsa -P '' \
  && cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys \
  && sed -i '/pam_loginuid.so/c session    optional     pam_loginuid.so'  /etc/pam.d/sshd \
  && echo -e "Host *\n StrictHostKeyChecking no" >> /etc/ssh/ssh_config
{% endhighlight %}

Following section will install Anaconda and PySpark. Most of the dependencies of my job were delivered in a ZIP file
with Python eggs inside, but I needed to have some of them installed locally to be able to submit PySpark driver:
{% highlight bash %}
# Installing Miniconda & PySpark
RUN wget -nv http://repo.continuum.io/miniconda/Miniconda-latest-Linux-x86_64.sh -O ~/miniconda.sh \
 && bash ~/miniconda.sh -b -p $HOME/miniconda \
 && rm -f ~/miniconda.sh

# Setting environment variables
ENV PATH "$HOME/miniconda/bin:$PATH"
ENV PYTHON "$HOME/miniconda/bin/python"
ENV PYTHONPATH "$PYTHON"

RUN ~/miniconda/bin/conda update -y conda \
 && ~/miniconda/bin/conda install -y pyspark
{% endhighlight %}

And the last few steps in a Docker file to install Ambari on a container, expose some ports and run systemd:
{% highlight bash %}
RUN wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.5.2.0/ambari.repo \
    -O /etc/yum.repos.d/ambari.repo \
 && yum repolist \
 && yum install -y ambari-server

EXPOSE 22 8080 8081 8082 8083 8084 8085 8086 8087 8088

CMD ["/usr/sbin/init"]
{% endhighlight %}

# Setup script

OK, when [Dockerfile] is ready, let's run everything. I'll provide a part of [install script] and put some comments
afterwards:
{% highlight bash %}

docker build -t centos-ambari .
docker network create ambarinet 2> /dev/null

# Launch containers
master_id=$(docker run --privileged -d --net ambarinet -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
    -p ${PORT}:8080 -p 50070:50070 -p 19888:19888 -p 8088:8088 --name ${NODE_NAME_PREFIX}-0 centos-ambari)

# Starting SSHD and NTPD services on a master node
docker exec $NODE_NAME_PREFIX-0 systemctl start sshd
docker exec $NODE_NAME_PREFIX-0 systemctl start ntpd

echo ${master_id:0:12} > hosts
for i in $(seq $((N-1)));
do
    container_id=$(docker run --privileged -d --net ambarinet -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
        --name $NODE_NAME_PREFIX-$i centos-ambari)

    # Starting SSHD and NTPD services on a worker node
    docker exec $NODE_NAME_PREFIX-$i systemctl start sshd
    docker exec $NODE_NAME_PREFIX-$i systemctl start ntpd

    echo ${container_id:0:12} >> hosts
done

# Copy the workers file to the master container
docker cp hosts $master_id:/root

# Print the private key
echo "Copying back the private key..."
docker cp $master_id:/root/.ssh/id_rsa .

# Setup and start the Ambari server
docker exec $NODE_NAME_PREFIX-0 ambari-server setup -s
docker exec $NODE_NAME_PREFIX-0 ambari-server start

# Print the hostnames
echo "Using the following hostnames:"
echo "------------------------------"
cat hosts
echo "------------------------------"

{% endhighlight %}
What does this bash script do:
1. Builds a Docker image
2. Establishes a network for th cluster
3. Launches master node with following settings:
   * *--privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro* - required to run systemd
   * *-p ${PORT}:8080 -p 50070:50070 -p 19888:19888 -p 8088:8088* - exposing some important ports
   * *--net ambarinet* - connected to *ambarinet* network
4. Starts SSHD and NTPD on master node
5. Repeats steps 3 and 4 for every other node in the cluster
6. Exposes SSH key to provide SSH-less access
7. Starts Ambari server setup on master node
8. Prints Docker hosts

# Ambari configuration
As soon as you see a message like this,
````
Using the following hostnames:
------------------------------
410fb41cc857
ed5ff538cd2c
255d4301cd5d
------------------------------
````
it's time put a terminal aside and go to [Ambari UI].<br/>

First of all, let's define user's credentials for Ambari. Once you fill in Username and Password, you'll automatically
create a new user.
![Login page]({{"/assets/img/ambari/login.png"|absolute_url}}){: .framed}

Next - welcome page. Here you just need to click "Launch install wizard" to proceed further.
![Welcome page]({{"/assets/img/ambari/welcome.png"|absolute_url}}){: .framed}

Let's define a nme for the cluster.
![Get started page]({{"/assets/img/ambari/cluster_name.png"|absolute_url}}){: .framed}

Ambari will ask to choose HDP version, which is 2.6.2 in this case. There's a possibility to use local repository, but I
didn't need any custom builds, so I went with public one.
![Select version page]({{"/assets/img/ambari/hdp.png"|absolute_url}}){: .framed}

Almost there. Now Ambari asks for cluster's hosts. Remember the output message from install script? Now it's time to 
copy and paste these names like shown on the picture below. The same should be done for an SSH key in order to provide
Ambari an access to cluster's nodes.
![Install options page]({{"/assets/img/ambari/hosts.png"|absolute_url}}){: .framed}

![Confirm hosts page]({{"/assets/img/ambari/confirm.png"|absolute_url}}){: .framed}

And... voila! Ambari registered hosts and now it's ready to install services. I could continue with this guide, but
that's a completely different story and it really depends on user's needs. For example, I needed HDFS, ResourceManager 
and Spark only so I'll just leave a link to the official 
Ambari documentation [HERE] and wish you having fun with your local Ambari cluster ;)

[//]: # (Link references)

[systemd]: https://en.wikipedia.org/wiki/Systemd "Wikipedia: systemd"
[https://github.com/ksmirnov/ambari_docker]: https://github.com/ksmirnov/ambari_docker "Ambari on Docker"
[Dockerfile]: https://github.com/ksmirnov/ambari_docker/blob/master/Dockerfile "Dockerfile"
[install script]: https://github.com/ksmirnov/ambari_docker/blob/master/deploy_cluster.sh "Install script"
[HERE]: https://ambari.apache.org/1.2.1/installing-hadoop-using-ambari/content/ch03s05.html "Ambari docs"
[Ambari UI]: http://localhost:8080/
