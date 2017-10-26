---
layout: post
title:  Prepare macOS for development from scratch
date:   2017-09-18 11:41:00 -0800
description: Unix commands cheat sheet
img: terminal.jpg
tags: [Blog, Unix, IT, Development, Terminal]
author: # Add name author (optional)
---
Have you ever happened to prepare a new laptop or desktop for development? I don't know about you, but it's a nightmare
for me! Everything looks and feels so unfamiliar and inconvenient. Recently my belongings in the office were complemented
with a Mac Pro and initially I wanted to use it as a cluster host only, but then I decided to connect it to a monitor,
buy a keyboard and a mouse for convenience, install some useful command line tools... and that's how it all started!
I'm really glad it's all behind right now. I turned it into a workstation and I'm writing this blog entry using this
black canister actually. But what surprised me when I was half way here, that I wasted so much time trying to remember
proper bash command (terminal history was so empty!), searching for download links and etc. So, I decided to create my 
own macOS cheat sheet featuring Homebrew, Git, Maven, Vim, IDEs and many more, because maybe somebody is looking for the
same steps right now?

**Note: the list is subject to change at any time as my memories come :D** 

## Step 1: Homebrew and bash completion along with some essential tools
{% highlight bash %}
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

brew upgrade

brew install vim git maven
brew install bash-completion maven-completion pip-completion docker-completion

brew tap homebrew/completions

echo "if [ -f $(brew --prefix)/etc/bash_completion ]; then
  . $(brew --prefix)/etc/bash_completion
fi" >> ~/.bash_profile

brew cleanup

{% endhighlight %}

## Step 2: Configuring SSH and Git
{% highlight bash %}

git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global alias.co checkout
git config --global apply.whitespace nowarn

ssh-keygen

{% endhighlight %}

## Step 3: Getting other tools installed
Here's more links to useful tools.
* Anaconda: [https://www.anaconda.com/download/#macos]
* IntelliJ IDEA: [https://www.jetbrains.com/idea/download/#section=mac]
* PyCharm: [https://www.jetbrains.com/pycharm/download/#section=mac]
* Docker: [https://www.docker.com/docker-mac]
* DBeaver: [https://dbeaver.jkiss.org/download/]
* MuCommander: [http://www.mucommander.com/]

## Commands: Common
{% highlight bash %}
# Getting OS name and version
cat /etc/*-release

# Determining kernel architecture
uname -a

# Printing out N last command lines
history N

# Getting an output of a file copied to clipboard
cat ~/.ssh/id_rsa.pub | pbcopy

# Finding a file by regex pattern
find . -regex '.*md'

# Find a file containing a pattern
grep -e '.*png' *

# Find a file by name and content
find . -regex '.*md' -exec grep -l -e 'Tensorflow' {} \;

# Copy from/to remote server with SHH
scp user@from-host:/path/to/source.file user@to-host:/path/to/destination.file

{% endhighlight %}

## Commands: Java
{% highlight bash %}
# Getting a byte code a class
javac ClassName.java

# Running a class in a classpath
java -cp lib/mypackage.jar Main arg1 arg2

# Running a JAR
java -jar lib/mypackage.jar arg1 arg2
{% endhighlight %}

## Commands: Docker
{% highlight bash %}
# Getting all containers:
docker ps -a

# Getting all images
docker images

# Stopping all containers
docker stop $(docker ps -a -q)

# Removing all stopped containers
# Where -a shows all containers, -q defines a representation with IDs only and -f defines filter status=exited
docker rm $(docker ps -a -q -f status=exited)

# Getting into Docker containers
docker exec -it <mycontainer> bash
{% endhighlight %}

[//]: # (Link references)

[https://www.anaconda.com/download/#macos]: https://www.anaconda.com/download/#macos "Anaconda"
[https://www.jetbrains.com/idea/download/#section=mac]: https://www.jetbrains.com/idea/download/#section=mac "IntelliJ IDEA"
[https://www.jetbrains.com/pycharm/download/#section=mac]: https://www.jetbrains.com/pycharm/download/#section=mac "PyCharm"
[https://www.docker.com/docker-mac]: https://www.docker.com/docker-mac "Docker"
[https://dbeaver.jkiss.org/download/]: https://dbeaver.jkiss.org/download/ "DBeaver"
[http://www.mucommander.com/]: http://www.mucommander.com/ "MuCommander"
