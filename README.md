# CoreOS on Vagrant Running an Ubuntu Docker Container with Apache

## Introduction

This is an example of how to run the Apache web server in a Docker container on CoreOS using Vagrant and VirtualBox. Let's break that down a bit...

### VirtualBox

VirtualBox allows you to run virtual machines on a real host machine. For example, you can run a virtual Windows machine on an Ubuntu host, or a virtual Ubuntu server on an OSX host. It's primarily a GUI application, with an interface for installing and configuring your machines.

[Get VirtualBox][]

### Vagrant

Vagrant is a tool to make it easy to create, provision, start stop and generally look after your VirtualBox machines. It's a command line application. Instead of using the VirtualBox GUI the configuration and settings from a machine are read from a `Vagrantfile`.

[Get Vagrant][]

### CoreOS

CoreOS is a minimal Linux distribution designed for distributed platforms. It's also handy for development environments too. It comes with Docker built-in.

[CoreOS website][]

### Docker

Docker is used to create lightweight, portable *containers* for your application. These are a little bit like virtual machines in that they have their own operating system, file system, processes etc. but unlike a VM they share resources with the host system.

[Docker website][]

## Installation

**Sorry, this example won't work on Windows**

Assuming you've got VirtualBox and Vagrant installed, from the root of this repo run:

    $ vagrant up

Assuming everything worked as expected you should now be able to visit http://localhost:8080 and see a very interesting web page :)

**WARNING** On first provisioning it could be quite slow to download CoreOS and the Ubuntu image. The pulling of the Ubuntu image can appear like it's hanging as doesn't give any progress. Try to be patient :) If not I've found it's sometimes quicker to stop the provisioning and manually pull the container. To do that press `Ctrl+C` twice to stop the provisioning. Then reload the machine without provisioning using `vagrant reload --no-provision`. Then SSH into the machine with `vagrant ssh` and run `docker pull ubuntu`. When that's done `exit` out of the machine and run `vagrant provision` again.

## About the Vagrantfile

This contains the tasks needed to install, configure and run CoreOS on a VM, which roughly break down as:

1. Download a CoreOS operating system image
1. Install CoreOS to a new virtual machine
1. Sort out some networking stuff
1. Set up a shared folder between your host machine and the guest CoreOS machine (this is what doesn't work on Windows)
1. Forward port 8080 on your host machine to port 8080 on guest CoreOS machine
1. Run a *provisioner* shell script on the guest CoreOS machine

The provisioner takes over to do the following steps:

1. Build a Docker container based on the `Dockerfile`
1. Add the systemd service to run the container whenever the machine starts
1. Start the container on initial provisioning

The provisioner is only run once when the machine is started for the first time. These steps don't need to happen on subsequent restarts of the machine.

## About the Dockerfile

The Dockerfile contains a set of instructions for building your container. There's a [Dockerfile tutorial][] which explains more about how this works in general. For this specific example the Dockerfile does the following:

1. Creates a new container based on the Ubuntu container
1. Sets up `aptitude` and does and `apt-get update` to update all the Ubuntu packages in the normal way
1. Installs apache and php5
1. Sets up apache environment
1. Tells the container to expose port 80
1. Sets an entry point to run apache on start up
1. Issues a command to run it in the foreground - this is important as normally a container exits after a command is completed, by running it in the foreground it won't exit

The building is done in the provisioner in the Vagrantfile with the line:

    docker build -t test-apache - < /home/core/share/Dockerfile

To break it down:

* `docker build` is the command to create a new container

* `-t test-apache` means tag the resulting container with the name "test-apache" (more about tagging below)

* ` - < /home/core/share/Dockerfile` means read the build instructions from this Dockerfile (which is read via STDIN) which is in the shared directory set up in the Vagrantfile

To explain about the tagging step, Docker works in a similar way to git, in that changes to a container are committed and can be pushed and pulled from repositories. There is a public repository of pre-built containers you can use to base your own containers on. It's also possible to have private repositories too, although this is all very alpha at the moment.

For this example the base container is the default Ubuntu one provided by the CoreOS team. This is an approved image, meaning it should be in a reliable working state.

In the Dockerfile each step effectively results in a commit. Each commit is cached so if you later add new steps to your container it will only have to run the steps after the last commit.

At the end we tag the resulting container with a name so we can refer to it later, but it isn't actually being pushed to any repository. To learn more about commit, push and pull try the [Docker tutorial][].

While this builds the container it doesn't actually run anything. This is done by creating a service file which is run by `systemd`.

## About the systemd File (apache.service)

This contains the information needed to run the container. You could manually run the container by SSHing into the server and running:

    docker run -v /home/core/share/app:/var/www -p 8080:80 -d test-apache

The `apache.service` file automates this to make sure the container runs when CoreOS starts up without you having to manually log-in and start it. It also contains other instructions, like identifying the service and setting how long it should wait before restarting after a crash. 

Let's break that command down:

* `docker run` is the command that actually runs the container.

* The `-v` flag mounts a directory on CoreOS as a *volume* that is then available to the container. For development this is a good way to share the actual runnable code for your application. In this case we mount the `app` directory at `/var/www`, the default website root for Apache that contains the website HTML. Remember this has already been shared with CoreOS in the Vagrantfile, so this directory is actually on your physical host machine.

* The `-p` flag forwards port 80 of the container to 8080 of CoreOS. Again this is then shared with your physical host machine by the port forwarding in the Vagrantfile.

* The `-d` flag means run as a daemon

* `test-apache` is the name of the container we created and tagged when using `docker build` with the Dockerfile.

During provisioning the `apache.service` file is copied to `/media/state/units` which is where all service files should go. Then systemd is restarted to ensure it works the first time the machine is provisioned. Once this is in place nothing else needs to be done, the service will start whenever the machine is restarted.

[Get VirtualBox]:http://virtualbox.org
[Get Vagrant]:http://vagrantup.com
[CoreOS website]:http://coreos.com
[Docker website]:http://docker.io
[Dockerfile tutorial]:http://www.docker.io/learn/dockerfile/
[Docker tutorial]:http://www.docker.io/gettingstarted
