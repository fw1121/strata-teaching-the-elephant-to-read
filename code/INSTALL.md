# Welcome to Teaching the Elephant to Read #

We are very excited to be teaching the Teaching the Elephant to Read tutorial at Strata on Monday, the 28th of October. We will be covering numerous software packages that can be time consuming to install on various operating sytems and laptops.

The instructions below will walk you through the creation and configuration of a virtual machine running in Virtual Box with a lot of different software packages. The instructions have been tested and debugged so you shouldn't have too many problems. However, if you do have any issues, please email me at sayhitosean [AT] gmail.com. 

Please note that you will need a **64-bit machine and operating system** for this tutorial. The virtual machine/image that we will be building and using has been tested on Mac OS X (up through Mavericks) and 64-bit Windows. 


# Installation #

In order to get ready execute the code in this tutorial, you will need a development environment set up and ready to go. These instructions will help you get it set up.

Please note that this process could take an hour or longer depending on the bandwidth of your connection as you will need to download approximately 1 GB of software. DO NOT WAIT UNTIL THE BEGINNING OF THE TUTORIAL to follow these instructions.

## Install and Configure Virtual Machine ##

First, you will need to install Virtual Box, free software from Oracle. Go here to download the 64-bit version appropriate for your machine:

<a href="https://www.virtualbox.org/wiki/Downloads" target="_blank">Download Virtual Box</a>

Once Virtual Box is installed, you will need to grab a Ubuntu x64 Server 12.04 LTS image and you can do that directly from Ubuntu here.

<a href="http://www.ubuntu.com/download/desktop/thank-you?release=lts&bits=64&distro=desktop&status=zeroc" target="_blank">Ubuntu Image</a>

## Linux Setup ##

First, let's create a user account with admin privileges with username "hadoop" and the very creative password "password."

    username: hadoop
    password: password

Honestly, you don't have to do this. If you have a user account that can already sudo, you are good to go and can skip to the install some software.  But if not, use the following commands.

	~$ sudo adduser hadoop
	~$ sudo usermod -a -G sudo hadoop
	~$ sudo adduser hadoop sudo

Log out and log back in as "hadoop." 

Now you need to install some software.

    ~$ sudo apt-get update && sudo apt-get upgrade
    ~$ sudo apt-get install build-essential ssh avahi-daemon
    ~$ sudo apt-get install vim lzop git
    ~$ sudo apt-get install python-dev python-setuptools libyaml-dev
    ~$ sudo easy_install pip

The above installs may take some time. 

At this point you should probably generate some ssh keys (for hadoop and so you can ssh in and get out of the VM terminal.)

    ~$ ssh-keygen
    Generating public/private rsa key pair.
    Enter file in which to save the key (/home/hadoop/.ssh/id_rsa):
    Created directory '/home/hadoop/.ssh'.
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:
    Your identification has been saved in /home/hadoop/.ssh/id_rsa.
    Your public key has been saved in /home/hadoop/.ssh/id_rsa.pub.
    [… snip …]

Make sure that you leave the password as blank, hadoop will need the keys if you're setting up a cluster for more than one user. Also note that it is good practice to keep the administrator seperate from the hadoop user- but since this is a development cluster, we're just taking a shortcut and leaving them the same.

One final step, copy allow that key to be authorized for ssh.

    ~$ cp .ssh/id_rsa.pub .ssh/authorized_keys

You can download this key and use it to ssh into your virtual environment if needed.

## Installing Hadoop ##

Hadoop requires Java - and since we're using Ubuntu, we're going to use OpenJDK rather than Sun because Ubuntu doesn't provide a .deb package for Oracle Java. Hadoop supports OpenJDK with a few minor caveats: [java versions on hadoop][hadoop_java]. If you'd like to install a different version, see [installing java on hadoop][installing_java].

    ~$ sudo apt-get install openjdk-7-jdk

Do a quick check to make sure you have the right version of Java installed:

    ~$ java -version
    java version "1.7.0_25"
    OpenJDK Runtime Environment (IcedTea 2.3.10) (7u25-2.3.10-1ubuntu0.12.04.2)
    OpenJDK 64-Bit Server VM (build 23.7-b01, mixed mode)

Now we need to disable IPv6 on Ubuntu- there is one issue when hadoop binds on `0.0.0.0` that it also binds to the IPv6 address. This isn't too hard: simply edit (with the editor of your choice, I prefer `vim`) the `/etc/sysctl.conf` file using the following command

	sudo vim /etc/sysctl.conf

and add the following lines to the end of the file:



    # disable ipv6
    net.ipv6.conf.all.disable_ipv6 = 1
    net.ipv6.conf.default.disable_ipv6 = 1
    net.ipv6.conf.lo.disable_ipv6 = 1

Unfortunately you'll have to reboot your machine for this change to take affect. You can then check the status with the following command (0 is enabled, 1 is disabled):

    ~$ cat /proc/sys/net/ipv6/conf/all/disable_ipv6

And now we're ready to download Hadoop from the [Apache Download Mirrors][download_hadoop]. Hadoop versions are a bit goofy: [an update on Apache Hadoop 1.0][guide_to_hadoop_versions] however, as of October 15, 2013 [release 2.2.0 is available][hadoop_update]. However, the stable version is still listed as version 1.2.1.

Go ahead and unpack in a location of your choice. We've debated internally what directory to place Hadoop and other distributed services like Cassandra or Titan in- but we've landed on `/srv` thanks to [this post][opt_or_srv]. Unpack the file, change the permissions to the hadoop user and then create a symlink from the version to a local hadoop link. This will allow you to set any version to the latest hadoop without worrying about losing versioning.

    /srv$ sudo tar -xzf hadoop-1.2.1.tar.gz
    /srv$ sudo chown -R hadoop:hadoop hadoop-1.2.1
    /srv$ sudo ln -s $(pwd)/hadoop-1.2.1 $(pwd)/hadoop

Now we have to configure some environment variables so that everything executes correctly, while we're at it will create a couple aliases in our bash profile to make our lives a bit easier. Edit the `~/.profile` file in your home directory and add the following to the end:

    # Set the Hadoop Related Environment variables
    export HADOOP_PREFIX=/srv/hadoop

    # Set the JAVA_HOME
    export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64

    # Add Hadoop bin/ directory to PATH
    export PATH=$PATH:$HADOOP_PREFIX/bin

    # Some helpful aliases

    unalias fs &> /dev/null
    alias fs="hadoop fs"
    unalias hls &> /dev/null
    alias hls="fs -ls"
    alias ..="cd .."
    alias ...="cd ../.."

    lzohead() {
        hadoop fs -cat $1 | lzop -dc | head -1000 | less
    }

We'll continue configuring the Hadoop environment. Edit the following files in `/srv/hadoop/conf/`:

**hadoop-env.sh**

    # The java implementation to use. Required.
    export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64

**core-site.xml**

    <configuration>
        <property>
            <name>fs.default.name</name>
            <value>hdfs://localhost:9000</value>
        </property>
        <property>
            <name>hadoop.tmp.dir</name>
            <value>/app/hadoop/tmp</value>
        </property>
    </configuration>

**hdfs-site.xml**

    <configuration>
        <property>
            <name>dfs.replication</name>
            <value>1</value>
        </property>
    </configuration>

**mapred-site.xml**

    <configuration>
        <property>
            <name>mapred.job.tracker</name>
            <value>localhost:9001</value>
        </property>
    </configuration>

That's it configuration over! But before we get going we have to format the distributed filesystem in order to use it. We'll store our file system in the `/app/hadoop/tmp` directory as per [Michael Noll][noll_tutorial] and as we set in the `core-site.xml` configuration. We'll have to set up this directory and then format the name node.

    /srv$ sudo mkdir -p /app/hadoop/tmp
    /srv$ sudo chown -R hadoop:hadoop /app/hadoop
    /srv$ sudo chmod -R 750 /app/hadoop
    /srv$ hadoop namenode -format
    [… snip …]

You should now be able to run Hadoop's `start-all.sh` command to start all the relevant daemons:

    /srv$ hadoop-1.2.1/bin/start-all.sh
    starting namenode, logging to /srv/hadoop-1.2.1/libexec/../logs/hadoop-hadoop-namenode-ubuntu.out
    localhost: starting datanode, logging to /srv/hadoop-1.2.1/libexec/../logs/hadoop-hadoop-datanode-ubuntu.out
    localhost: starting secondarynamenode, logging to /srv/hadoop-1.2.1/libexec/../logs/hadoop-hadoop-secondarynamenode-ubuntu.out
    starting jobtracker, logging to /srv/hadoop-1.2.1/libexec/../logs/hadoop-hadoop-jobtracker-ubuntu.out
    localhost: starting tasktracker, logging to /srv/hadoop-1.2.1/libexec/../logs/hadoop-hadoop-tasktracker-ubuntu.out

And you can use the `jps` command to see what's running:

    /srv$ jps
    1321 NameNode
    1443 DataNode
    1898 Jps
    1660 JobTracker
    1784 TaskTracker
    1573 SecondaryNameNode

Furthermore, you can access the various hadoop web interfaces as follows:

* [http://localhost:50070/](http://localhost:50070/) – web UI of the NameNode daemon
* [http://localhost:50030/](http://localhost:50030/) – web UI of the JobTracker daemon
* [http://localhost:50060/](http://localhost:50060/) – web UI of the TaskTracker daemon

To stop Hadoop simply run the `stop-all.sh` command.

## Python Packages ##

To run the code in this section, you'll need to install some Python packages as dependencies, and in particular the NLTK library. The simplest way to install these packages is with the `requirements.txt` file that comes with the code library in our repository. We'll clone it into a repository called `tutorial`.

    ~$ git clone https://github.com/bbengfort/strata-teaching-the-elephant-to-read.git tutorial
    ~$ cd tutorial/code
    ~$ sudo pip install -U -r requirements.txt
    [… snip …]

However, if you simply want to install the dependencies yourself, here are the contents of the `requirements.txt` file:

    # requirements.txt
    PyYAML==3.10
    dumbo==0.21.36
    language-selector==0.1
    nltk==2.0.4
    numpy==1.7.1
    typedbytes==0.3.8
    ufw==0.31.1-1

You'll also have to download the NLTK data packages which will install to `/usr/share/nltk_data` unless you set an environment variable called `NLTK_DATA`. The best way to install all this data is as follows:

    ~$ sudo python -m nltk.downloader -d /usr/share/nltk_data all

At this point the steps that are left are loading data into Hadoop.

### References ###

1. [Hadoop/Java Versions][hadoop_java]
2. [Installing Java on Ubuntu][installing_java]
3. [An Update on Apache Hadoop 1.0][guide_to_hadoop_versions]
4. [Running Hadoop on Ubuntu Linux Single Node Cluster][noll_tutorial]
5. [Apache: Single Node Setup][single_node_setup]

<!-- References -->
[hadoop_java]: http://wiki.apache.org/hadoop/HadoopJavaVersions "Hadoop/Java Versions"
[installing_java]: https://help.ubuntu.com/community/Java "Installing Java on Ubuntu"
[download_hadoop]: http://www.apache.org/dyn/closer.cgi/hadoop/core "Hadoop Mirrors"
[guide_to_hadoop_versions]: http://blog.cloudera.com/blog/2012/01/an-update-on-apache-hadoop-1-0/ "Apache Hadoop version 1.0"
[hadoop_update]: http://hadoop.apache.org/releases.html#15+October%2C+2013%3A+Release+2.2.0+available "October 15, 2013: Release 2.2.0 available"
[opt_or_srv]: http://serverfault.com/questions/96416/should-i-install-linux-applications-in-var-or-opt "Stack Overflow"
[noll_tutorial]: http://www.michael-noll.com/tutorials/running-hadoop-on-ubuntu-linux-single-node-cluster/#sun-java-6 "Michael Noll"
[single_node_setup]: http://hadoop.apache.org/docs/stable/single_node_setup.html "Single Node Setup"
