---
title: "Ubuntu 20.04 basic server set-up (beginner tutorial)"
description: "In this blog we will set-up a basic Ubuntu 20.04 server using a Digital Ocean droplet."
date: 2020-09-27T12:24:15+02:00
draft: false
toc: false
images:
  - /images/blog/3/header.png
tags:
  - server
  - cloud
---

## Introduction :point_down:

In two previous blog posts, we explored the ease of deploying serverless functions. These are obviously a great way of removing the hassle of maintaining a server and most of the times also reducing the running costs.

However besides the whole **"serverless hype"** we are seeing right now in the tech world, a lot of projects still require a traditional cloud VM.

You might need more flexiblity, your app doesn't work well on a serverless set-up or it just doesn't fit your usecase. Luckily for us non-cool folks going _server_ instead of _serverless_ is just as easy! :smile:

So let's go over that in this blog post!

**Note:** To follow this tutorial I assume you have a terminal available. This can be on your Linux machine, Mac OS or Windows Linux Subsystem.

## 1. What are the options :question:

There are multiple cloud providers available that offer virtual cloud machines. Some of the most wide known:

- Digital Ocean
- AWS EC2
- Vultr
- Linode
- Kamatera

In this post we will use my personal favourite: Digital Ocean! :heart:

Besides being one of my favourite cloud providers, you also get $100 worth of credit for free if you sign up through [this referral link](https://m.do.co/c/52031fcadf3c). Can't get any better than that right?

Exactly, that's what I thought. So let's start! :fire:

## 2. Setting up SSH keys on your machine :key:

Secure Shell (better known as SSH) is a **cryptographic network protocol** which allows users to securely perform a number of network services over an unsecured network.

SSH keys provide a more secure way of logging into a server with SSH than using a password alone. While a password can eventually be cracked with a brute force attack, SSH keys are nearly impossible to decipher by brute force alone.

Generating a key pair provides you with two long string of characters: **a public and a private key**. You can place the public key on any server, and then unlock it by connecting to it with a client that already has the private key.

When the two match up, the system unlocks. :unlock:

#### I. Create RSA key pair

So the first step is creating a key pair on our development machine with the following command:

```
$ ssh-keygen -t rsa
```

The terminal prompt will ask you to set a password for the key. Make sure to **always set a password!** If you don't and someone could get unauthorized access to your machine, they could just copy your private key to their own machine and log in to all your servers. So make sure to password protect your key. :closed_lock_with_key:

The output will be similar to this:

```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/raf/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/raf/.ssh/id_rsa.
Your public key has been saved in /home/raf/.ssh/id_rsa.pub.
The key fingerprint is:
4a:dd:0a:c6:35:4e:3f:ed:27:38:8c:74:44:4d:93:67 raf@a
The key's randomart image is:
+--[ RSA 2048]----+
|          .oo.   |
|         .  o.E  |
|        + .  o   |
|     . = = .     |
|      = S = .    |
|     o + = +     |
|      . o + o .  |
|           . o   |
|                 |
+-----------------+
```

The public key is now located in `/home/raf/.ssh/id_rsa.pub`. The private key (identification) is now located in `/home/raf/.ssh/id_rsa`.

## 3. Adding the key to our Digital Ocean account & launching the droplet :file_folder:

Next up is adding our new key to our Digital Ocean account through their GUI. The benefit of this is that whenever you spin up a droplet, it will automatically copy the key to the new server! Saving you from doing that manually, which makes life easier.

The first thing we have to do is getting our public key. We can do that by running the following command:

```
$ cat ~/.ssh/id_rsa.pub
```

The output in your terminal should be a long string of characters. Similar to the one below here. Copy your public key.

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC/aOIJUaUUiZP0KA4vslD97oNVzVYZAiZTTG2Z3ozis3pnt3ZKL5k/XvdOCyfb449BjsShiQim7h1+/m/HEQNVggOqQpWigMf+Ky+zHbQm9mAy+/PpIYMfbWI29nuBnljA9yPcTAdKQM15KNobWeawz0vxoxrncX88sadfasl3j23SFKGUuFpVHXfd4P4njisXrR8vRh07EoU5xafjXrBPYAIsc1vHpK8SpTvwZFc8NwVq2VoQ2s0ig3Fw0gLTgj7roaDEfCW0KZ6SBkAZEmCsFxTnQF8NDiafupNo9ic09xLijvwwasdlkasdfasl2an3fK4twqRcZuZQiRFIppb3kMFQmVWCXW1Udi2ph79TyEXKvmtEnynY+s9SYcGdDbYnbMKqHCrfD4D2JKWXZmuxt7QkFFGTWJeDTPKynmhfuihqQ84LxIK2Gi8sPhRdcs15Kw5A5ctpz+aqd3WUJl78dyvakIcoTNHG06zxR2X7hpz+8U2DQmort4/Dtb5XO9myE5POevdasCroPHxEslzZShHw6dSztC48j7LJ80mSYhI46Vfk5jiH2gT8IHF3JhxytNxBYwTClBYq0c2+M7o62vCVKRCoKFMomCsfME1TmwnJWOMWfnFqMXwBmanUEqJFldEOSg66+egZ5Ym1gSIdww== rafrasenberg@Rafs-MacBook-Pro.local
```

Now [sign up for Digital Ocean](https://m.do.co/c/52031fcadf3c) and set-up your account.

After you have logged into the dashboard, head over to your **_Settings_** and click the tab **_Security_**. After that click on **_Add SSH key_**. This will open up a modal where you can paste the public key in. Make sure to save it.

![Digital Ocean dashboard](/images/blog/3/image1.png)

After that click on the button in the top right corner **_Create_** and from the dropdown choose **_Droplet_**.

DigitalOcean Droplets are Linux-based virtual machines (VMs) that run on top of virtualized hardware. Each Droplet you create is a new server you can use, either standalone or as part of a larger, cloud-based infrastructure.

When adding your droplet, make sure you choose Ubuntu 20.04 as the image and pick the standard $5 plan. When scrolling down make sure you select your SSH key as well, that you previously added in your account and give your server a cool hostname! :cool:

View the GIF below to see the full walkthrough.

![Creating Digital Ocean droplet](/images/blog/3/gif1.gif)

## 4. Logging in to our server :rocket:

Alright we have already come far haven't we? Now it is time for the real work!

If everything went well you should see a droplet instance in your Digital Ocean dashboard with the hostname you chose and the IP address that belongs to the droplet. Copy this IP address and fire up a terminal session and run the following command (replace the IP address with yours):

```
$ ssh root@167.23.10.37
```

It should ask for a password. This is the password that you set-up for your SSH key in step 2. Accept the warning about host authenticity if it appears. You should now be logged in as the root user on your very own cloud server. How cool is that! :tada:

**Note:** Root is the superuser account in Unix and Linux. It is a user account for administrative purposes, and typically has the highest access rights on the system.

## 5. Basic server set-up :computer:

When you first create a new Ubuntu 20.04 server, you should perform some important configuration steps as part of the basic setup. These steps will increase the security and usability of your server, and will give you a solid foundation for subsequent actions.

#### I. Setting up a basic firewall

Ubuntu 20.04 servers can use the UFW firewall to make sure only connections to certain services are allowed. We can set up a basic firewall very easily using this application.

Applications can register their profiles with UFW upon installation. These profiles allow UFW to manage these applications by name. OpenSSH, the service allowing us to connect to our server now, has a profile registered with UFW.

Before we enable the firewall, make sure that OpenSSH is added to our firewall configuration. Otherwise we won't be able to log back in. Run the following command:

```
$ ufw allow OpenSSH
```

Now enable our firewall on start-up:

```
$ ufw enable
```

If you check the status of our firewall now with the below command, it should be up and running.

```
$ ufw status
```

Currently the firewall is blocking all connections except for SSH, if you install and configure additional services, you will need to adjust the firewall settings to allow traffic in.

Small example: Let's say you want to run a webserver and you want to expose your website to the outside world through SSL. You would need to open up port 443 in order to serve the website, because currently the firewall is blocking all connections. That is as easy as running:

```
$ ufw allow 443
```

Make sure to always reload the firewall afterwards for the changes to apply. You can do this by running:

```
$ ufw reload
```

#### II. Changing the default SSH port

By default, SSH runs on port 22. To prevent automated bots and malicious users from brute-forcing to your server, we can change the port to something else.

An intelligent attacker would still scan your server to determine open ports and services running on them. However, changing the default SSH port will block thousands of automated attacks that don’t have time to rotate ports when targeting a Linux Server. :grin:

You can choose any available port you like, in this example we will use `Port 3467`. So the first thing we do is making sure our firewall allows this port. As you probably know by now, this can be done with:

```
$ ufw allow 3467
```

and then reload the firewall with:

```
$ ufw reload
```

Now we need to make an adjustment to our SSH config file and tell it to use our newly chosen port. You can do that by running the below command:

```
$ nano /etc/ssh/sshd_config
```

Search for the line that says `#Port 22`:

```
#Port 22
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::

#HostKey /etc/ssh/ssh_host_rsa_key
#HostKey /etc/ssh/ssh_host_ecdsa_key
#HostKey /etc/ssh/ssh_host_ed25519_key
```

Uncomment it and change it to `Port 3467`. Now save the file by using `ctrl + x` and then type `y` to save and exit.

The last thing that we have to do is make sure that we restart the SSH service to apply our new changes. You can do that with the following command:

```
$ systemctl restart sshd.service
```

> **IMPORTANT:**
> Whenever you make changes like this **ALWAYS** double check if you can get back into the server first, before you close your terminal session :exclamation:

So let's see if we can log back into our server through the new port. When you try the following:

```
$ ssh root@167.23.10.37
```

This will result in a host rejection. It's because the `ssh` command defaults to `Port 22`, however we obviously changed that and therefore it isn't in use anymore. What we have to do then is specifically set the port in our login command. You can do that like so:

```
$ ssh root@167.23.10.37 -p 3467
```

If everything went well, you just logged in through the new port. Now it is safe to close your other terminal session.

#### III. Disabling password authentication

A common practice after you have set-up the server for SSH key access, is disabling the password authentication option. This makes sure only connections with a pair of SSH keys are allowed.

Fortunately for us, Digital Ocean already took care of that because we specified our SSH key at the initalization of the droplet. However if you use a different cloud provider, always make sure to disable that. You can do that in the SSH config file.

#### IV. More security practices

There is a lot more to cover, but the steps we discussed in this blog is the bare minimum what I would recommend when setting up a server. The next obvious step is creating a regular user for daily use.

Security wise, you can take it further by improving your Firewall, setting up services like Fail2Ban, disabling direct root login, etc. I won't bore you with all of that here :zzz:, but feel free to check out the web for more information about this.

## Conclusion :zap:

Alright, we took a small dive into the world of server configuration.

In the next blog post we will go over the things that you need in order to actually serve a webapp on your server. We will also integrate a basic CI pipeline that connects with our droplet that will automatically deploy our changes after pushing it to Github. So stay tuned for that!

See you next time! :wave:
