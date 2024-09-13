# docker-saltstack

# 🚀 Docker Compose setup to spin up a salt master and minions for easy testing, learning, and prototyping. 🚀

https://github.com/coding-to-music/docker-saltstack

From / By https://github.com/cyface/docker-saltstack

https://timlwhite.medium.com/the-simplest-way-to-learn-saltstack-cd9f5edbc967

## Environment variables:

```java

```

## GitHub

```java
git init
git add .
git remote remove origin
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:coding-to-music/docker-saltstack.git
git push -u origin main
```

# docker-saltstack

Docker Compose setup to spin up a salt master and minions.

You can read a full article describing how to use this setup [here](https://medium.com/@timlwhite/the-simplest-way-to-learn-saltstack-cd9f5edbc967).

You will need a system with Docker and Docker Compose installed to use this project.

Just run:

`docker-compose up`

from a checkout of this directory, and the master and minion will start up with debug logging to the console.

Then you can run (in a separate shell window):

`docker-compose exec salt-master bash`

and it will log you into the command line of the salt-master server.

From that command line you can run something like:

`salt '*' test.ping`

and in the window where you started docker compose, you will see the log output of both the master sending the command and the minion receiving the command and replying.

[The Salt Remote Execution Tutorial](https://docs.saltstack.com/en/latest/topics/tutorials/modules.html) has some quick examples of the comamnds you can run from the master.

Note: you will see log messages like : "Could not determine init system from command line" - those are just because salt is running in the foreground and not from an auto-startup.

The salt-master is set up to accept all minions that try to connect. Since the network that the salt-master sees is only the docker-compose network, this means that only minions within this docker-compose service network will be able to connect (and not random other minions external to docker).

#### Running multiple minions:

`docker-compose up --scale salt-minion=2`

This will start up two minions instead of just one.

#### Host Names

The **hostnames** match the names of the containers - so the master is `salt-master` and the minion is `salt-minion`.

If you are running more than one minion with `--scale=2`, you will need to use `docker-saltstack_salt-minion_1` and `docker-saltstack_salt-minion_2` for the minions if you want to target them individually.

### Medium article:

![{{line_content}}](/images/simplist-way-to-learn-saltstack.png)

# The Simplest Way to Learn SaltStack

By Tim White

https://timlwhite.medium.com/?source=post_page-----cd9f5edbc967--------------------------------

8 min read

Feb 21, 2018

180 Likes

### intro

tl;dr: Use Docker Compose to instantly spin up a SaltStack Master and Minions, shell into the master and remotely run commands on the minions all without installing SaltStack on your system, paying for cloud instances, or wrangling ssh keys. The impatient can jump right to the quickstart.

When I first was diving into the world of fabric/chef/puppet/ansible/salt, I gravitated to SaltStack because it was 1) Python, and 2) Focused on performance running commands across a LOT of minions. But along the way I tried all of them, and tore out a lot of hair out installing / uninstalling each tool and its dependencies.

The challenge with any of these tools is that you need a suite of servers linked together to really see how it all works. Many of the tutorials focused on spinning up a bunch of cloud servers on Digital Ocean or AWS and then installing and configuring these tools on them one by one.

I gave the cloud cluster approach a try, but I found that I was so focused on trying to do things as quickly as possible (so I could delete the cloud instances that charge per hour) that I couldn’t focus on learning and evaluating. In addition, the extra complexities of roles and security, passing around ssh keys, and network connectivity with a cloud cluster muddied the waters.

## Enter Docker Compose

I have been a HUGE fan of docker-compose for the last few years, and I finally sat down to figure out how to build a docker-compose environment to test these cluster-based tools in a calm, relaxed, repeatable way that let me focus on the tool and not the cloud.

What I created is a single GitHub repo that contains the Docker and Docker Compose configurations to spin up multiple virtual machines right on your desktop or dev server, log into them, issue salt commands, and watch what happens.

Even once you are using salt in production, a setup like this is essential to test out new salt configurations without risk to your real environments.

The configurations of all the components are extremely minimal so you can figure out what is going on as quickly as possible, and extend things as you need to.

## How SaltStack Works

SaltStack can be configured a lot of ways, but the default way is to have a master to which many minions connect. The connection is permanent between them, and secured with SSH keys. By default, that connection uses ZeroMQ for ultra-fast data transfer, and executes commands on all targeted minions in parallel, which means you can affect thousands of minions simultaneously.

Once the master and minions are connected, you can run commands on the master that are executed on any or all of the minions. You can use information about each minion, such as its name (e.g. webserver_1) or details about the server (which SaltStack calls grains) to filter which minions execute given commands. This is the Remote Execution piece.

You can also define states which can cause configuration to be applied and validated on minions. Configuration meaning users, directories, files, firewall, services, and so forth. This is the “Configuration Management” piece. One slightly confusing concept here the ‘highstate’, which is all the states combined into a comprehensive configuration for the whole shebang.

SaltStack also has an event-based Reactor that can monitor for events and then run commands and apply states based on those events.

Finally, there is an API Daemon that can be run to listen for commands from sources other than the shell.

This article doesn’t dive into Reactor and the API, but this docker-compose setup absolutely would support going there.

## Project Layout

The project consists of Dockerfiles, which define the base operating system containers and installation of SaltStack on those containers, and the docker-compose.yml, which defines a ‘stack’ of those containers that are linked together with a private virtual network.

Both the Dockerfiles start with Ubuntu, and use the Ununtu package manager to update the system, add the SaltStack package repository, and install either the salt master or minion. Then there is a one-line find/replace to tell the master to accept all minions, and the tell the minions what the DNS name of the master is. It then starts up the master or minion when the container starts. It also installs git on the minion, since many of the things you want to do need to have git installed. The virt-what package is also installed to give SaltStack a bit more information about the virtual machine.

One thing to note, the SaltStack Master is set up to accept all minions that try and connect to it. While this would be a terrible thing to do in an open network, the private network that docker-compose sets up means that only our minions will be able to get to the master, and it saves a lot of headaches with the ssh keys.

### docker-compose.yml

The heart of the setup is the docker-compose.yml file:

```bash
version: '3.7'
services:
  salt-master:
    build:
      context: .
      dockerfile: Dockerfile.master
  salt-minion:
    build:
      context: .
      dockerfile: Dockerfile.minion
    depends_on:
      - salt-master
```

It’s as minimal as can be — we define two services, salt-master and salt-minion. Both of them are defined with a build which means that they will create new containers when the stack first starts up. The context sets up the working directory for the build, and dockerfile tells it which file to use. The depends_on ensures that the minion doesn’t start until the master has started. It doesn’t check that the master is ready, it just controls the startup order.

Compose uses this minimal amount of information to work a lot of magic for us. When we start up the stack, it will create a virtual network between the services, add DNS entries to each container for all the services, set the hostname of the container to the docker container id (i.e. salt-master and salt-minion), and save the state of our containers so that when they are started up again they will remember their state — at least until you intentionally rebuild them.

Fire it up using the Quickstart instructions below, and we’ll dive into what more we can do with this after that.

## Quickstart

Install Docker and Docker-Compose (If you are on Win10 or Mac OS, the Docker install includes Compose). Note that if you don’t have git, you can download a zip file of this setup instead.

Open command line to your favorite working directory and:

```bash
git clone https://github.com/cyface/docker-saltstack.git
cd docker-saltstack
docker-compose up
```

Open new command line window in the docker-saltstack directory and:

```bash
docker-compose exec salt-master bash
salt '*' test.ping
```

Watch the first window for debugging output from the salt master and minions, issue commands from your salt-master bash session.

The exec command tells docker-compose to run a command on an already-running container, in this case to run the bash shell on the salt-master, which logs you into that container. The salt command is then issued from inside the container.

## The Log Output

The window where you started up the stack is now full of DEBUG logging from both the salt-master and the salt-minions. The output is prefixed with the name of the service, so you can tell what is coming from where. This is a lot of output, but it’s very helpful in seeing what is going on.

Note that you may see “Could not determine init system from command line” in the output; that is because the Ubuntu container we are using isn’t running an init system (since Docker assumes you will be running one main process per container in the foreground). Since we are running Salt-Master and Salt-Minion in the foreground on these containers, their output is what you will see in the Compose output.

The logs can fly by pretty fast if you issue commands that do a lot, so make sure your terminal has a lot of lines of scrollback set up.

## Running Commands on the Minions

Now you can run more commands on the minions. You already ran one command in the Quickstart — the test.ping command:

```bash
salt '*' test.ping
```

The salt command is the ‘run stuff’ command. The first argument indicates which minions to run the command on — ‘\*’ targets all the minions. The next argument is the command to run, followed any arguments.

You can query the grains on the minions to find out more about them:

```bash
salt '*' grains.get os
```

This command reports back the operating system of each minion.

From this point, you can start going through the official salt tutorials while shelled into the master, and watch what happens to your minions!

If you want to shell into a minion to see how it was affected, make sure you are on your local system (not inside a container) and run:

```bash
docker-compose exec --index=1 salt-minion bash
```

The index command is how you specify which of the minions to log into — helpful if you are running multiple minions (see below on how to do that).

## Running Multiple Minions

```bash
docker-compose up --scale salt-minion=2
```

This will start up two minions instead of just one, still all networked together!

## Minion Names

The hostnames of the minions in this setup are their docker container IDs: salt-master and salt-minion.

If you start up with a a scale setting of more than one, the actual names of the minions will be `docker-saltstack_salt-minion_1` and `docker-saltstack_salt-minion_2`. You can experiment with with the grains setting in the `/etc/salt/minion` file to set up useful metadata about each minion.

For example, if you set up the roles grain, so that one of the minions has webserver in its roles list:

```bash
salt -G 'roles:webserver' grains.get roles
```

Will return the roles grain from just minions who have the webserver in their roles grain.

## Starting/Stopping/Restarting Things

You can `ctrl-c` to stop the docker stack in your log monitoring window, that will shut down all the containers.

From another window, you can run things like:

```bash
docker-compose restart salt-minion
```

To restart all the minions (useful if you edit the `/etc/salt/minion` file for example).

If you want to run the cluster in the background without watching the logs:

```bash
docker-compose up -d
```

To stop the cluster:

```bash
docker-compose down
```

To adjust the scale of the minions (which you can do while its running):

```bash
docker-compose up -d --scale salt-minion=3
```

Setting a scale less than the current scale will shut down containers.

## Things to try

Looking for a few things to try before you dive into the full tutorials?

### Copy a file to all the minions

```bash
touch myfile.txt
salt-cp '*' myfile.txt /myfile.txt
```

### Create a state the installs ngnix on all the minions

```bash
mkdir /srv/salt
echo -e "nginx:\n  pkg.installed" > /srv/salt/nginx_installed.sls
salt '*' state.apply nginx_installed
```

### Conclusion

I hope this setup is helpful to you. As a quick note — if you really want to run SaltStack in Docker containers (underneath some primary process) then it’s quite a bit more involved. There is a new option for targeting Docker containers with SaltStack without running the Minion, which you might find helpful.

https://docs.saltstack.com/en/latest/topics/tutorials/docker_sls.html
