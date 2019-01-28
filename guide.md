# DRAFT - OpenBSD Router Guide

**WARNING**: This is a rough draft - do NOT use it

**License:** [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/)

This guide will help you build your own OpenBSD router and firewall for your home network.

Why would you want to do this? 

Commercial routers are dangerous because they are full of security flaws. Using OpenBSD on a do-it-yourself router is much more secure.

I decided to use a barebones computer with two network interface cards as my router. I chose the [OEM Production 2550L2D-MxPC Intel NM10 Booksize Barebones System](https://www.newegg.com/Product/Product.aspx?Item=N82E16856205007) from Newegg. I put this computer into service as a router in August 2013 and it has been chugging along ever since.

## Outline

1. Prerequisites
1. Download OpenBSD
1. Install OpenBSD on the Router
1. Configure router
1. Install operating system
1. Configure the router
1. Configure wireless networking
1. Configure VPN
1. Perform Server Security Tweaks
1. Perform Firewall Security Tweaks
1. Last steps

## Prerequisites

### Skills, Knowledge & Attitudes

While this guide explains a lot of things, there are some things that are out of scope.

Overall, you should be ok to embark on this journey if you already have the following skills, knowledge and attitudes:

* welcome a challenge
* harbor no fear of the command line
* have a willingness to learn
* edit files from the command line using [`vi`](https://en.wikipedia.org/wiki/Vi)
* install an internal hard drive
* install RAM
* use `scp` to securely copy files from one computer to another
* create SSH keys
* append SSH public keys to an `authorized_keys` file

### Tasks

Before installing, do the following:

* choose a machine name for this router - perhaps `myrouter`
* choose a password for the root user (administrator) of this router
* get a USB key to use as an OpenBSD installer (2 GB is plenty)
	* some computers cannot boot off a USB 3.0 drive - you should be good-to-go if you use a USB 2.0 drive
* choose the hardware you will use for the router
	* 2 &#215; RJ-45 ethernet jacks
		* one to connect to cable modem
		* one to connect to your internal home network
	* compatible with OpenBSD
	* hard drive big enough - a small ssd (16 GB) should be more than enough
	* install RAM
    * figure out which key to press during the startup to get to your BIOS boot settings - you might need this to get your router to boot off the USB key
* draw a network map - trust me, this will help you visualize everything
	* show lines of demarcation
	* label interfaces by OpenBSD designations
	* label the subnets (ex: 192.168.1.0, 192.168.X.0)
	* use colored pencils to mark the logical paths to test
		* cable modem <--> OpenBSD external interface
		* OpenBSD internal interface to a host
		* host to OpenBSD internal interface
		* host to OpenBSD external interface
		* host to a random interweb site (cnn.com)
* figure out what the subnet(s) will be
* physically connect the external interface to the lan

## Download the OpenBSD Operating System

On your non-router computer you can download OpenBSD directly from [OpenBSD.org](https://www.openbsd.org).

Click on the Download link on the main page.

Locate the **`installXX.fs`** disk image.

* choose
	* **`amd64`** - this is probably the file you want if
		* your computer has a 64-bit Intel or 64-bit AMD cpu
	* **`i386`** - you'll want this one if your computer if
		* your computer has an Intel Atom cpu
		* your computer is > 10 years old

This file is around 350 MB and it should download fairly quickly.

Now that you have the installer file you need to burn it onto the USB drive.

### Burn the `installXX.fs` disk image onto the USB installer drive

Note: Simply copying it from your Downloads folder is not enough. You need to do a bit-for-bit copy from the installer file to the USB drive.

* insert your USB drive into the computer on which you downloaded the installer file
* determine the node of the USB drive
	* if you're using GNU/Linux it will be `/dev/sd**X**` <-- replace the X with the letter of the USB drive
	* if you're using Windows, check your file explorer - it will be D: or E: or something like that
* 'Burn' the installer file onto the USB drive


#### GNU/Linux

**WARNING**: Doing a bit-for-bit copy to the USB drive will delete the contents of the drive.

In the following `dd` command, `if` represents the input file and `of` represents the device node of the USB drive.

	$ sudo dd if=/enter/path/to/install*.fs of=/dev/sdX bs=1M

#### Windows

Search Google for instructions on how to install the installer file on Windows.

There is a good [guide](https://hammondmason.wordpress.com/2015/02/09/getting-an-openbsd-image-onto-a-usb-stick-using-windows/) on Hammond Mason's blog.

## Install OpenBSD on the Router

Now that you have the installer file burned onto your USB drive, you are ready to install OpenBSD on your router

### Initial Power-on

You have your USB key with the OpenBSD installer and you have your router ready to go.

* plug your USB key with the OpenBSD installer into your router
* plug an ethernet cable in between your cable modem and your router
* plug a monitor into your router (it will normally run headless (without a monitor) but for now only you'll want to see what it's doing)
* plug in a keyboard
* start up your router and hit the necessary key to enter the BIOS setup
* configure the clock to Coordinated Universal Time [(UTC)](https://www.timeanddate.com/worldclock/timezone/utc)

### Installation Walkthrough

Here's a listing of what you need to do at each step along the way of a normal installation.

`Welcome to the OpenBSD installation program.` **`(I)nstall`**

`Choose your keyboard layout` **`[default]`**

`System hostname?`  **`routername`**

`Available network interfaces are: em0 em1.`  **`choose the one that you plugged in (see Prerequisites section)`**

`IPv4 address for em0?`  **`[dhcp]`**  

Note: **`dhcp`** is a good choice here since eventually, this port will be connected straight into the cable modem. Using this setting, the ip address, name servers, and search domain for this external-facing interface will be set by the DHCP server (in this case: your ISP). You'll set up an inward-facing DHCP server that will provide dynamic ip addresses to the computers inside your network later in this guide. 

Write down these interface names - you'll need them later in the setup

`IPv6 address for em0?`  **`[none]`**

`Which network interface do you wish to configure?` **`[done]`**

Note: So far we've only configured the ethernet port connecting your router to your cable modem. We'll configure the other ethernet port later on in this guide.

`DNS domain name` **`[enter your domain name (example.com)]`**

Don't worry if you don't have a domain name or you don't want to assign a domain name - you can leave this blank

`Password for root` --  **`enter it!`**

`Start sshd by default?` **`[yes]`**

Enabling the SSH daemon (`sshd`) to start by default will enable to you connect securely to this computer after you finish the install.

`Do you expect to run the X Window System `  **`[yes]`** ...not really, but what does it really hurt?

`Do you want X to start by xenodm?` **`[no]`**

`Setup user?`  **`[no]`**

Although you really _do_ want to add another user, you don't want to do it right now because you can't change the UID here.  It's better to use the `adduser` command once the system is up and running. You'll see how to do this later in this guide.

`Allow root ssh login?` **`[no]`**

We'll set up SSH later in this guide.

**WARNING**: This next step will erase any data on the hard drive that you choose. 

This next step assumes that you are installing OpenBSD on your router hardware and that the hard disk in your router is blank or anything you will be overwriting has already been backed up.

`Which disk is the root disk?` **`[default]`**

This question is asking you to specify which disk you'd like to install OpenBSD. The default is correct if you are installing on your router and the hard drive is blank.

`Use the (W)hole disk`  **`[whole]`**

`Use (A)uto layout`   **`[a]`**

The installer whill format the hard drive - this only takes a few seconds.

`Location of sets? `  **`[cd0]`** Note: 'cd0' is not the default.

`Set name(s)` **`-g*`** to prevent downloading unneeded games

`What timezone are you in` **`[enter your country code]`** type ? if you don't know your country code

`What sub-timezone are you in` **`[enter your sub-timezone name]`** type ? if you don't know

`Directory does not contain SHA256.sig. Continue without verification?` **`[yes]`** Note: 'yes' is not the default.

Wait for everything to install. This should only take a moment. You will be surprised at how fast installing the OpenBSD operating system goes.

`Time appears wrong. Set time to [verify this timestamp is accurate]` **`[yes]`**

Unplug the cable connecting the router to the cable modem.

There is no need to be connected at this point. Reduce the threat cross-section for now by disconnecting your external-facing LAN cable.

Reboot your router and remove the USB installer drive.

## Configure the Router

In this section some basic configuration of the computer takes place.  Next some networking tasks are accomplished.  Finally, the router is brought up to speed and tested.

Overview of the system configuration:

* Configure `doas`
* Add a non-root user
* Apply patches
* Configure OpenSSH
* Set up the network interfaces
* Configure the DHCP server for the internal network
* Configure the `pf` firewall
* Update the DNS server

The following configuration files will be edited on the command line:

* `/etc/hostname.[internal interface]`
* `/etc/sysctl.conf` to enable routing
* `/etc/rc.conf.local` to allow DHCP to run at startup
* `/etc/dhcpd.conf` to provide DNS and ip address on the internal LAN
* `/var/unbound/etc/unbound.conf` to allow DNS caching

### Configuration - Operating System

Log in as root using the password you entered in the installation process.

First things first...

#### Configure `doas`

Note: You will need to know how to edit files with the `vi` text editor to complete the following steps.

The `doas` program is a program that will let you execute commands as another user. In our case, we'll use `doas` to execute programs as the root user.

* create the `/etc/doas.conf` file

	`# touch doas.conf`

* add the following to the `doas.conf` file to allow any user in the `wheel` group to execute commands as any user while keeping the environment variables `PS1` (the user prompt and `SSH_AUTH_SOCK` and unsetting the `ENV` environment variable:
	
	`permit setenv { -ENV PS1=$DOAS_PS1 SSH_AUTH_SOCK} :wheel`

#### Add users with `adduser`

* add any admin users to the `wheel` group

	`# adduser`

You'll be prompted to create a default adduser file - click enter to accept the defaults.

Eventually you'll get to the part where you can actually add a user:

	Ok, let's go.
	Don't worry about mistakes. There will be a chance later to correct any input.
	Enter username:
	Enter full name:
	Enter shell: [default]
	Uid: [default]
	Login group: [defaults to be same as username]
	Invide user into any other groups: **wheel**

#### Patch the OpenBSD Router Installation

The installer only contains OpenBSD as it was on release day. There have been security patches since then so let's apply the patches now.

	$ doas syspatch
	
This will apply the patches that are included on the Patches link on this [page]((https://www.openbsd.org/)).

#### SSH

Since your router will run headless, you'll need a way to connect to it. We will use OpenSSH (SSH) to make an encrypted connection from your non-router computer to your router. This guide assumes that you'll be connecting to your router from within your home LAN. You _can_configure your router so that you can connect to it from the scary internet, but that's beyond the scope of this guide.

Copy your **public** ssh keys that you will be using to access this router from your daily-driver laptop or desktop computer to your new user account on this router. Append your client public (has 'pub' in the filename) SSH keys to this file on the router `$HOME/.ssh/authorized_keys`

##### Configure `sshd_config`

Make the following changes in the `/etc/ssh/sshd_config` file:

	PubkeyAuthentication yes 
	PasswordAuthentication no
	ChallengeResponseAuthentication no

Test your keys by trying to log in from a local-LAN machine. You should be prompted for a passphrase instead of a password.

If you were able to successfully SSH into the router from a machine on the local LAN using your SSH key and not a password, reboot the router.

Plug in an ethernet cable between your cable modem and the router.

Note: If you are interested in learning more about SSH, I **highly** recommend [SSH Mastery by Michael W Lucas](https://www.tiltedwindmillpress.com/product/ssh-mastery-2nd-edition/).

### Configuration - Network Interfaces

Network interfaces

* One interface will be external-facing: connected to your DSL or cable modem.
* The other interface will be internal-facing: it will connect to the other computers on your local (home) network.

In this example, 

* `em0` is external-facing and
* `em1` is internal-facing

Luckily, we already configured `em0` during the install process. Now we need to configure the internal network interface `em1`.

1. Configure the OpenBSD _internal_ interface (`em1`)

* Add the following to `/etc/hostname.em1`:
* **`inet 192.168.1.1 255.255.255.0`**

2. Kick start the networking service: 

* **`# /bin/sh /etc/netstart`**

3. Enable forwarding of LAN traffic (this makes it a router):

NOTE: DNS (unbound) needs to be configured before anything will work

* Create the file `/etc/sysctl.conf` if it does not already exist and add the following: 

	**`net.inet.ip.forwarding=1`**

4. Kickstart the networking again.

	`# /bin/sh /etc/netstart`

5. Test pinging from a local LAN client to the OpenBSD internal and external interfaces

### Configuration - DHCP Server ###

This section sets up the DHCP server that will assign ip addresses to your local computers. The DHCP server will run on the internal-facing interface, `em1`.

1. Configure DHCP to run at startup.  

* Add the following to `/etc/rc.conf.local` (create this file if it does not already exist):

	**`dhcpd_flags="em1"`**

2. Configure DHCP server details:

* Create the file `/etc/dhcpd.conf` if it doesn't already exist
* Add the following to that file:

	option domain-name-servers 192.168.1.1; 
	
	subnet 192.168.1.0 netmask 255.255.255.0 
	{ 
	option domain-name-servers 192.168.1.1;
	option routers 192.168.1.1; 
	range 192.168.1.50 192.168.1.75;
	}

Notes:

* `option domain-name-servers` is what the clients of this computer will use, so, in this case it will be the ip address of this computer, 192.168.1.1
* `option routers` is the ip address of the gateway that the clients will use to get out to the Internet
* `range` is the low and high boundaries for dynamically-assigned ip addresses - in this case I set the low to 50 and the high to 75 - you can change these


1. Reboot and test DHCP on an a client that is attached to the OpenBSD internal interface.  You should be able to ping both interfaces on the OpenBSD router.  However, you will not be able to get to the outside world just yet.

### Configuration - NAT (firewall) ###

** NEED TO UPDATE **

Test the ruleset by issuing the following (will give you the line number of any syntax errors):

	# pfctl -nf /etc/pf.conf

If you get nothing in response, then you have no syntax errors. Load the pf ruleset by issuing:

	# pfctl -ef /etc/pf.conf

You might also need to kickstart networking:

	# sh /etc/netstart

Next reboot and try accessing any website on a client connected to the OpenBSD internal interface.

### Configure DNS Server

This will provide DNS to the apartment LAN hosts.

Edit `/var/unbound/etc/unbound.conf` so it includes the following:

To Do

Once the `/var/unbound/etc/unbound.conf` file is written, check the file to make sure there are no errors:

	$ unbound-checkconf

Fix any errors that `unbound-checkconf` identifies.

Restart networking:

	$ doas sh /etc/netstart
		
### Change to a Non-Default Name Server

As configured, the external-facing interface, `hostname.em0` is using the DNS servers that the external DHCP (your ISP) is specifying. Alternatively you can point this interface to use whichever DNS server you choose - for example: a DigitalOcean droplet, OpenDNS, or even Google.

Add this to the `/etc/dhclient.conf` file:

	interface "em0" {
		supersede domain-name-servers 208.67.220.220,208.67.222.222;
		}

Please note the spelling of `supersede` above has no letter `c`.

Rester networking:

	$ doas sh /etc/netstart

Check to see if the newly-specified DNS servers are bing used:

	$ cat /etc/resolv.conf

You should see the ip addresses of the new DNS servers you specified in `/etc/dhclient.conf` listed in the output.



## Configure Wireless Network ##

Set up wireless networking for the house.

## Configure VPN ##

## Perform Security Tweaks ##

## Last Steps

Sign up for the OpenBSD "announce" mailing list to be informed about any security vulnerabilities.

Go to the [OpenBSD Mailing List Server](https://lists.openbsd.org/cgi-bin/mj_wwwusr?user=&passw=&func=lists-long-full&extra=announce) to sign up.

## References ##

1. http://networkfilter.blogspot.com/2014/05/defend-your-network-and-privacy-vpn.html
2. http://www.bsdnow.tv/tutorials/openbsd-router
3. https://www.ietf.org/rfc/rfc1042.txt
