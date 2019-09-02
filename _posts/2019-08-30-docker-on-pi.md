---
layout: post
title:  "Docker Swarm on a Raspberry Pi Cluster"
date:   2019-09-02 09:00:00 -0700
categories: tech
tags: rpi raspbian docker swarm
---
### Docker Swarm Services on a Raspberry Pi 4 Cluster
For sake of ease, this post makes the following assumptions:
- _At least_ 2x Raspberry Pi 4's (3 is better, 5 is ideal)
- You've a macOS device for initial configuration of the SD cards
- Familiarity with a termianl


#### Step 1: Getting Raspbian
First, you will need to procure an operating system image for your Raspberry Pi and a utility called [balenaEtcher](https://www.balena.io/etcher/) to flash that image onto your SD card.

There are lots of choices out there for RPi-compatible operating systems, but for this post we'll be using Raspbian. Raspbian is a derivative of the Debian operating system which has been tuned specifically for use with Raspberry Pi hardware, and is the Pi community's most common OS choice for projects like these. As of writing this, Raspbian is built atop the Debian Buster release.

To procure a copy of the Raspbian image, mosey on over to the [Raspberry Pi's downloads page](https://www.raspberrypi.org/downloads/raspbian/) and you will be presented three different images of the Raspbian operating system:
1. **Raspbian Lite** => A minimal, no-frills version of Raspbian with only bare necessities
2. **Raspbian** => Raspbian Lite + a desktop environment
3. **Raspbian Full** => Raspbian + recommended software

The choice of which variant to use is up to you. In most cases, I normally opt for the most lightweight image and install additional packages as I need then, but for this post, I chose going with Raspbian Full. Given that we're going to be configuring at least 2 separate Docker nodes (1x manager, 1x worker), it will be beneficial to use the same image variant across both of your RPi4's. Now, while your Raspbian image is downloading, you will want to download and install the latest release of [balenaEtcher](https://github.com/balena-io/etcher/releases/latest).

_Do note:_ If you're running the macOS 10.15 Catalina beta (like I am) you will need to launch balenaEtcher via the binary (embedded in its `.app`) with `sudo` in order for the SD card flashing process to succeed. To launch it with `sudo`, open a terminal and enter the following:
`$ sudo /Applications/balenaEtcher.app/Contents/MacOS/balenaEtcher`

With balenaEtcher's GUI open, simply drag the Raspbian image's `.zip` file into that window, connect your SD card, and hit "Flash!". The flashing process will take about 5 minutes to finish. Once it has completed, the SD card might have been automatically ejected, if that's the case then you will need to disconnect then reconnect the SD card to your machine.

**Now open up a terminal window and let's get configuring!**

#### Step 2: Network Setup
Now that the Raspbian image has been laid down on the SD card, we need to configure it to work on our wireless network and allow us to remotely login to the device via SSH (because connecting peripherals is for the birds).

In order for our Raspberry Pi to enable SSH connections upon its initial network connection (which we will setup next), we need to make sure there is an empty file called `ssh` at the root of the RPi's boot volume. The existence of the `ssh` file _at this location_ will tell Raspbian that we acknowledge the security concern with allowing SSH connections (primarily due to the fact that the device will have only default credentials at that time) but we want to do it anyway. That said, we can easily achieve creating this empty file using `touch`:
`$ touch /Volumes/boot/ssh`

In order for your RPi to know what wifi network is _your_ wifi network, we need to create a new configuration file on your SD card which dictates these settings. Let's use `nano` to create this new file:
`$ nano /Volumes/boot/wpa_supplicant.conf`

With `nano` open, we can now begin writing the configuration for connecting to our wireless network, which will look something like this:
```
country=NL
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="MySSID"
    psk="MyPassphrase"
}
```
There are three key-values in this file that you will need to change:
1. `country` => Replace this value with your [2-character ISO 3166 country code](https://en.wikipedia.org/wiki/ISO_3166-1)
2. `ssid` => Swap out `MySSID` for whatever you've named your wifi network
3. `psk` => Here you need to enter your SSID's preshared key (a.k.a your wifi password)

Once your config file has been updated, hit `ctrl + x` (to initiate exiting `nano`) followed by `y` (to save your changes) and then simply hit `Enter` (to confirm saving changes to `/Volumes/boot/wpa_supplicant.conf`). Now simply eject your first SD card, put it aside, and repeat steps 1 and 2 for however many cards you have.

We are going to do the next step one device at a time (so don't power up all your Pi's yet!) to avoid any sort of network issues relating to duplicate hostnames on these devices.

**With your first Raspberry Pi powered on, proceed to step 3.**

#### Step 3: Changing system defaults
##### Login via SSH
As mentioned earlier, because of the existence of the `/boot/ssh` file, we are able to login to these devices using the default credentials (_Username:_ `pi` / _Password:_ `raspberry`) baked into the Raspbian image:
`$ ssh pi@raspberrypi.local`
When prompted for the password, enter `raspberry`. You're likely to encounter a `ECDSA host key` warning the first time you login, just type `yes` to accept the warning and it should then spawn your secure shell session's prompt:
```
pi@raspberrypi:~ $
```

##### Rotating the default password
The absolute first thing you should do is to change the default password to a strong-but-memorable passphrase. We can initiate a password reset for our `pi` user with the `passwd` command like so:
```
pi@raspberrypi:~ $ passwd
Changing password for pi.
(current) UNIX password:
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
pi@raspberrypi:~ $
```

##### Change hostname and hosts file
Changing the hostname of each device is important for two reasons: First so you can easily distinguish each device. Second, to avoid hostname conflicts. When a conflict (i.e. when two or more devices join the same network with the same hostname) _does_ occur, every device that joins the network after the first one will end up with an empty (or, in some cases, altered) hostname in your network's routing table. Ultimately, this results in difficulty isolating (and connecting to) a network device. Suffice it to say that it is worth a little extra effort on the front-end to not wind-up in this situation.

(As an aside, you may also find it helpful to physically label each device with its hostname as to not end up confusing which device corresponds to which hostname.)

To modify our RPi's hostname we must modify two separate files: `/etc/hostname` and `/etc/hosts`. We'll first start with modifying the easier of the two files by running:
`pi@raspberrypi:~ $ sudo nano /etc/hostname`

When `nano` opens this file, you will notice that it is quite plain; just a text file with nothing but the hostname of the device. This file's purpose is likely exactly what you'd assume it to be... It tells the device its hostname. All you need to do is delete `raspberrypi` and then enter whatever you'd like to call your first RPi4 node (I've named mine `pi1`). After modifying the contents of the file to reflect the new hostname you want to use, save and close it, and we can move on to modifying `/etc/hosts`.

While the `/etc/hostname` file tells the device its own hostname, the `/etc/hosts` file does something similar but slightly different: it tells a device the hostnames of its own and _other devices_ on the network and correlates those hostnames to IP addresses. Think of it as a local DNS server that trumps all other DNS providers within a network. Again, let's fire up our old friend `nano` and get to it:
`pi@raspberrypi:~ $ sudo nano /etc/hosts`

When the file opens, you'll see something that looks like the following:
```
127.0.0.1	raspberrypi
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters
```
For the first RPi device in your cluster, you'll only want to change the hostname which is mapped to the device's loopback interface: `127.0.0.1`. Simply replace `raspberrypi` with `pi1` (or whatever you decided to call it), then close and save the file. _Easy peasy._

Now if this is the 2nd/3rd/Nth device you're configuring, you'll want to also append a new line which maps the other device's internal IP's to their respective hostnames.

You might have noticed that after changing the `/etc/hostname` file, your shell prompt still reflects `pi@raspberrypi:~ $`. _What gives?_ Don't worry, after rebooting the device our change will take effect.

##### Reboot
Before you reboot the device, we want to jot down its IP address, as it will need to be populated in the `/etc/hosts` file on each of the devices you configure after the first one. Also, don't forget to cycle back through all devices you're configuring to ensure that all of the Pi's IP/hostname mappings exist. Let's assume you have 5x Pi's, your `hosts` file on `pi1` might look something like this when its all said and done:
```
127.0.0.1	pi1
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters

192.168.0.112	pi2
192.168.0.113	pi3
192.168.0.114	pi4
192.168.0.115	pi5
```

Once you've modified the system defaults for your first device, we need to reboot the device, with `sudo reboot`, and then wash-rinse-and-repeat step 3 for each subsequent device until you've got all your RPi4's reachable via hostname on your local wireless network.

#### Step 4: Docker Installation
##### Updating with `apt`
At this point, all of our Raspberry Pi's have had their initial configuration set, but there's still a bit of groundwork needing to be laid down before we can get Docker running on these Bad Larry's. Specifically, we need to make sure that each of them are fully patched, that the `apt` package manager's cache of available packages is up-to-date, and that they all have the necessary packages (and their dependencies) installed on them.

First, let's get these devices up-to-date by invoking this chain of commands:
`pi@pi1:~ $ sudo apt update && sudo apt upgrade && sudo reboot`

The first command in the chain, `apt update`, will update the list of available packages and versions. The second command, `apt upgrade`, actually performs the pending updates for packages that are already installed. Then, once the device comes back online after it reboots, be sure to perform this same command on each device (or "node" as we will soon be calling them).

##### Docker Engine
Now we're starting to actually get into the guts of Docker! In order for Docker (and its various components like `docker-compose`) to function properly, there are a few dependent packages that we need to ensure are installed:
`pi@pi1:~ $ sudo apt install -y python python-pip libffi-dev python-backports.ssl-match-hostname`
I won't explain what each of these packages do, but I encourage you to Google them if you're curious. Moving on...

Since we've satisfied all necessary dependencies for Docker, we're ready to install it. Thanks to the kind folks over at Camp Docker, as they've provided us a quick'n'easy shell script to handle the installation; all we have to do is:
`pi@pi1:~ $ curl -sSL https://get.docker.com |sh`

Woohoo! Our Raspberry Pi is now Docker-laden. But in order to allow the `pi` user to use Docker, we need to add the user account to the `docker` group.

_"Wait... wut? What's the `docker` group you speak of?"_

Ah, yes... The wonders of running arbitrary shell scripts from the internet. The shell script that we ran in the previous command not only installed the package for Docker, but it also made some slight modifications to the system itself. More specifically, the installation script we ran also created a new UNIX group called `docker`, and then granted privileges to run Docker to all members of that group. (If you're still curious, you can actually take a quick peek at all groups that exist on the system by running `cat /etc/group`.)

Anywho, onward! To add our `pi` user to the `docker` group, we run:
`pi@pi1:~ $ sudo usermod -aG docker pi`

Then we can confirm that Docker was properly installed, and that the service is running, by invoking `docker info`, and Docker will return a bunch of information relating to the Docker service:
```
pi@pi1:~ $ docker info
Client:
 Debug Mode: false

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 19.03.1
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 894b81a4b802e4eb2a91d1ce216b8817763c29fb
 runc version: 425e105d5a03fabd737a126ad93d62a9eeede87f
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 4.19.66-v7l+
 Operating System: Raspbian GNU/Linux 10 (buster)
 OSType: linux
 Architecture: armv7l
 CPUs: 4
 Total Memory: 3.814GiB
 Name: pi2
 ID: UIT2:MZR2:3ZQL:4R3X:7VB4:MCCK:LOBM:5PFE:QHYY:NRGM:3ZGY:UF5K
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false

WARNING: No swap limit support
WARNING: No cpu cfs quota support
WARNING: No cpu cfs period support
```

_Fantastic!_ We now have a functioning Docker environment running on our Raspberry Pi. Now to lay down the final piece (albeit optional since compose is not required for Docker) of the Docker ecosystem: `docker-compose`
##### Docker-Compose
Before we install compose, let's take a step back and get a brief overview of what it is. Docker provides us the ability to build containerized environments, and compose works atop that by allowing us to use the YAML syntax to define an application which spans across multiple containers. So if we look at a Docker container as a blueprint for the process of building a bike, we could look at compose as a blueprint for building a factory that builds bikes. Probably not the best analogy, but it will suffice...

To install `docker-compose`, we're going to leverage one of the dependencies we installed a short while ago; a Python package manager called `pip`:
`pi@pi1:~ $ sudo pip install docker-compose`

Once `pip` wraps up the installation of compose, we can confirm the installation succeeded by asking `docker-compose` for its version:
```
pi@pi1:~ $ docker-compose --version
docker-compose version 1.24.1, build 4667896
```

_Joy!_ We're going to give our Raspberry Pi its final reboot before we start rolling out services (and their underlying containers). Just like before, run the following to reboot:
`pi@pi1:~ $ sudo reboot`

And as with step 3, you will now want to repeat step 4 across all nodes which will be a part of our Docker Swarm cluster.

#### Step 5: Docker Swarm
We have now approached the last required bits to get our RPi cluster running a service with Docker Swarm. The final step is to initialize our Docker Swarm and add our first RPi as the swarm's manager node. We can make this swarm initialization happen by running the following command from our first node in the cluster (or whichever node you would like to make the manager):
`docker swarm init --advertise-addr <IP_Address_of_node>`

After running this command, you will see that the swarm was successfully initialized:
```
pi@pi1:~ $ docker swarm init --advertise-addr 192.168.0.111
Swarm initialized: current node (92zz1235w20zdhafk6zmzuxnz) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1aibc07zzza12asdfhjmxaxabc76mbhaabcza0a0sx0abcc2zp-00s6xzdt27y2gk2kpm0cgo6y1 \
    192.168.0.111:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Additionally, after successful invocation of the swarm, `docker swarm init` even provides you the command needed for adding worker nodes to this swarm. Let's go ahead and SSH into another node, and run the provided command to get our swarm complete with a worker:
```
pi@pi2:~ $ docker swarm join --token SWMTKN-1-1aibc07zzza12asdfhjmxaxabc76mbhaabcza0a0sx0abcc2zp-00s6xzdt27y2gk2kpm0cgo6y1 192.168.0.111:2377
This node joined a swarm as a worker.
```

Great! We now have a single manager node, as well as a worker node. Let's go ahead and run this command again on all of our subsequent nodes:
```
pi@pi3:~ $ docker swarm join --token SWMTKN-1-1aibc07zzza12asdfhjmxaxabc76mbhaabcza0a0sx0abcc2zp-00s6xzdt27y2gk2kpm0cgo6y1 192.168.0.111:2377
This node joined a swarm as a worker.
```

Now that all of our worker nodes have been added to the swarm, we can jump back into our manager node and confirm the status of all ndoes in the swarm:
```
pi@pi1:~ $ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
94hf9145w20odh4fk5zmwuhns *   pi1                 Ready               Active              Leader              19.03.1
pzzfxdghpzyxso74u4mlsx1sp     pi2                 Ready               Active                                  19.03.1
45w20opzzfxd94hf9145w20op     pi3                 Ready               Active                                  19.03.1
```

And there we have it: a 3 node cluster of Raspberry Pi 4's all running Docker in Swarm mode and ready to begin rolling out services!
