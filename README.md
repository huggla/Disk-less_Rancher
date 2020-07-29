# How to create disk-less Rancher hosts.
I believe that the seed to containerization came from the media live boot, adopted by virtually every Linux distribution for testing and installation. These, read only, ISO files has much in common with container images. But, while many server system administrators has gone all in, using containers for virtually every server task, they are still installing and maintaining at least one Linux server for the sheer purpose of running Docker. I plan to show, in a series of articles, how to set up a Rancher 1.6 environment where both Rancher Server and Rancher Agents are run on machines without hard drive, where installation has been replaced by a single, read only, ISO image. In this first article will I focus on how to setup disk-less Rancher hosts. With the help of this guide you will be able to create flexible server clusters in no time, using already existing, low spec hardware.

To be able to follow this guide you’ll need Internet access and at least 8 GB of free ram. If you choose to boot the live image from a physical device (not in a virtual machine) then you will also need a second computer with an Internet browser.


## The basics

I will start out with the most basic stuff that most of you probably already know. If you are an experienced Linux/Rancher user, feel free to skip to the next chapter.


#### Getting a live ISO image

While most Linux distributions provide ISO images, only some of those contain the Docker tools by default. In this guide we will be using RancherOS which is build for the sheer purpose of running Docker. The latest generic RancherOS live image can be downloaded from here (https://releases.rancher.com/os/latest/rancheros.iso).


#### Putting it on media

The ISO image file can easily be burnt to an empty cd-r/rw or dvd+-r/rw using any disc burning software. Don’t simply drop the ISO file onto the disc, choose “burn image”, or you’ll end up with a disc that won’t boot. While optical disc still is quite common, most of us prefer to boot from flash based storage. To “burn” the live image to an empty USB drive I recommend Etcher (https://etcher.io/). For testing out this guide you could also choose to mount the ISO as boot media in a virtual machine.


#### Booting

Now, lets boot that RancherOS image. Note! Some computers don’t boot discs or flash devices “out of the box”. In that case Google is your friend.

Once booted you’ll end up at the RancherOS command prompt. If anything goes wrong while working in the live environment you can always reboot the ISO and restart from the beginning. The ISO file is read only and nothing is saved between boot ups.


#### RancherOS

All you need to know about RancherOS can be read on https://rancher.com/rancher-os/. There you’ll also find video tutorials and a forum where you can ask questions. One thing some of you will notice is that RancherOS uses English keyboard layout. Sadly, there is no easy way to change keyboard layout in the RancherOS live environment, so you’ll have to learn where to find some keys by trial and error. You can choose to access RancherOS through ssh though, then you’ll be able to use any keyboard layout, but this guide won’t cover how to do that.


#### Running a basic disk-less Rancher setup

Start Rancher Server by typing the following command at the RancherOS command prompt:
sudo docker run -d --restart=unless-stopped -p 8080:8080 rancher/server:stable

While you wait for Rancher Server to start up, find out your IP address by running ifconfig. Then open a browser on an other computer, or, if you have booted RancherOS in a virtual machine, just switch window. Go to http://<SERVER_IP>:8080. On the Rancher Server start page click “Add a host” and then press Save. On the next page you’ll get the command for starting Rancher Agent and adding the host to your Rancher setup. Go back to the RancherOS prompt and run that command.
Now in the Rancher Server web interface navigate to INFRASTRUCTURE>Hosts and confirm that the host has been successfully added.

Now you have created a very basic, disk-less Rancher setup, ready for container deployment.


## A more useful setup

When you’ve spent some time with the basic disk-less Rancher setup, you’ll notice a bunch of problems that makes the basic setup only useful for simple testing. In this chapter I will go into more detail and show how to iron out some of those problems, making disk-less hosts a great choice under certain circumstances. 


#### A note on persistent state

One thing that stands out when trying the basic setup is the lack of persistence. Every command written, setting changed or container created are instantly and irrevocably deleted once RancherOS is restarted. The RancherOS documentation mentions one way to store all that information in one place, that is to add a “persistent state partition”. Problems with that approach are:
1. The setup wouldn’t be disk-less anymore.
2. You end up with something that, in practice, differs little from an ordinary RancherOS installation.

If we look what’s actually stored on the persistent state partition we’ll find Rancher Agent state files, RancherOS configuration files, and (the largest part) files that belong to docker, like images, volumes, containers etc. Among those files you’ll also find the Rancher Server database. The Rancher Agent state files are not essential, but good to have (as we will see later). They are mostly static and can, once configured, be moved to read only media. The RancherOS configuration isn’t strictly essential either but, most would probably consider it a must have. It’s static too, and can be read from cloud-config files residing in the cloud, on a local network share, or on read only media. Most of the docker files are only needed if you lack access to a reliable docker image repository (like Dockerhub), they can be pulled or recreated during RancherOS start up.


### The lossless Rancher host

In this example we will use the rancheros.iso image file as the read only media where to store Rancher Agent state files as well as cloud-config. Lets start with the cloud-config.


#### Rancher host cloud-config

If you are totally new to cloud-configs you should now take a moment to read about it in the RancherOS documentation. Here is a short list of things we want to put in the cloud-config:

¤ A unique hostname.
¤ Network settings.
¤ Docker command to pull and run Rancher Agent after boot.

Open a text editor and create your cloud-config. Save as plain text and name the file user_data. Below is cloud-config template you can use as a starting point. The host registration url is part of the host registration command you got from the Rancher Server web GUI earlier (if the server is still running).

>#cloud-config<br>
hostname: \<A unique hostname\><br>
rancher:<br>
&emsp;&emsp;network:<br>
&emsp;&emsp;&emsp;&emsp;dns:<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;nameservers:<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;- \<Add nameserver ip!\><br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;search:<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;- \<Add search address!\><br>
&emsp;&emsp;&emsp;&emsp;interfaces:<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;eth0:<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;address: \<Add host ip or remove!\><br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;gateway: \<Add gateway or remove!\><br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;dhcp: \<Set false or true!\><br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;mtu: \<Set mtu or remove!\><br>
&emsp;&emsp;services:<br>
&emsp;&emsp;&emsp;&emsp;rancher-agent1:<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;command: \<Host registration url\><br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;image: rancher/agent<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;environment:<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;- CATTLE_HOST_LABELS=\<Add label key!\>=\<Add label value!\><br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;privileged: true<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;volumes:<br>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;- /var/run/docker.sock:/var/run/docker.sock


##### Editing the iso

To be able to edit the otherwise read only ISO file we need to install a tool. I use ISO Master which is available for both Linux and Windows. 

Start ISO Master and open rancheros.iso. 

Now you are able to browse and edit the contents of the ISO. You’ll notice that the root of the RancherOS ISO contains two directories: boot and rancheros. The boot directory contains everything needed for booting and running the live system. The rancheros directory contains a RancherOS Docker image and an archive file that is needed if you wish to install RancherOS to a hard drive. You can choose to delete the rancheros directory if you wish, but there is no harm in leaving it untouched.

Create the following directory layout, where openstack is a directory located in the root of the ISO file and latest is a subdirectory inside openstack.
/openstack/latest

Now add the cloud-config file, named user_data, to the “latest” directory. 

Lastly you need to change the label of the ISO so that RancherOS recognizes it as a config drive.
Open the File menu in ISO Master and choose Properties. Change the volume name from RancherOS to config-2.

Save the edited ISO with a new filename.

Lets try out your new, disk-less, rancher host image by burning it to a new boot media or boot it in a new virtual machine.


## Problems with non persistent Rancher Agent state

The disk-less Rancher we have at this point is quite good but there are a few annoyances. The biggest one is caused by the lack persistent Rancher Agent state. You’ll notice this when you reboot a host. When the host reconnects with Rancher Server after boot, Rancher Server will fail to identify it as one of the already registered hosts, and it will be register as a brand new one. This results in lost host settings, a cluttered hosts overview, and hostname confusion.


### Moving Rancher Agent state to the iso

With the Rancher Agent state files stored in ram, you will have a new set created at each boot. We don’t want that, so lets move them to the ISO. In RancherOS you’ll find the files under /var/lib/rancher/state.

Change directory by running the following command at the RancherOS command prompt:
cd /var/lib/rancher/state

List all the contents of the folder (including hidden files):
ls -la

The agent state is located in three hidden files: .docker_uuid, .physical_host_uuid and .registration_token. In the state directory you’ll also find a directory named cni, ignore it. 

Print the contents of the files:
cat <file name> && echo

Recreate the three files in a text editor on the machine where ISO Master is installed. Start ISO Master and select View hidden files in the View menu. Create a new directory in the root of the ISO and name it state and then add the copies of .docker_uuid, .physical_host_uuid and .registration_token.

(Question to Rancher: Can the contents of .docker_uuid, .physical_host_uuid and .registration_token be found in Rancher Server web GUI? Where?)

Next, add the following lines to the end of user_data and save the new ISO image:

runcmd:
- mkdir -p /media/config-2 /var/lib/rancher/state
- mount /dev/disk/by-label/config-2 /media/config-2
- cp /media/config-2/state/.* /var/lib/rancher/state

Thats it. Boot the new image. Rancher Server will now recognize the host and not add a new one.


## Optimizing Docker images for disk-less hosts

You should note the size of your Docker images when you run purely in ram. The larger the image, the less memory will be available as working memory for running containers. You can also save much memory if all images running on a host share the same base image, for example the official Alpine image. Even though a base image is used by many running containers, it is still only downloaded once. Another thing you need to look at is how the services inside your Docker images are configured. Many images requires you to mount a volume if you want something else than the default configuration. To avoid adding volumes, make sure that important settings can be set with runtime environment variables.

If you need inspiration on how to optimize your Docker images for diskless hosts, or if you just want a image to try out, then take a look at my images: https://hub.docker.com/u/huggla/


## Creating a cheep cluster of Rancher hosts

Imagine that you need a server but have no money to buy one. 

In a first scenario you do have access to a trove of no longer used, somewhat old, workstations, of which most lacks a hard drive. 

In a second scenario you are working in a small office with a handful coworkers, all with a computer on their desk.

So, what should you do?

Well, we haven’t covered how to run Rancher Server disk-lessly (in a useful way) yet, so we still need to install it on one computer. That’s needed in both scenarios. In scenario 1, you could then, with minor preparations, boot the rest of the workstations from USB sticks and use them as Rancher hosts. In scenario 2, you could install Virtualbox on your coworkers computers and have it automatically start a virtual machine in the background when the computer starts. Each virtual machine would boot an ISO image and serve as a Rancher host. Computer resources would then be shared between server tasks and desktop tasks performed by the coworker.
