# Setup an Secure Nginx Web Server on Ubuntu and Debian
(Updated August 15, 2018)

## Table of Contents  
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Setup and Initial Server](#setup-and-initial-server)
- [Update and Upgrade](#update-and-upgrade)
- [Install Common Packages](#install-common-packages)
  * [1 Build essentials](#1-build-essentials)
  * [2 Extra Packages](#2-extra-packages)
  * [3 OpenSSL (with ALPN support)](#3-openssl)
  * [4 Nginx](#4-nginx)

## Table of Contents
-[Introduction](#introduction)
-[Prerequisites](#prerequisites)
-[Setup and Initial Server](#setup-and-initial-server)
-[Update and Upgrade](#update and upgrade)
-[Install Packages](#install-packages)
    * [1. Build essentials](#1-build-essentials)
    * [2. Extra Packages](#2-extra-packages)
    * [3. OpenSSL](#3-openssl)
    * [4. Nginx](#4-nginx)


## Introduction

The new Ubuntu and Debian (and other Debian based distros) has to be customized before it can be put into use as a production system. In this article, I will you to increase the security and usability of your server with the latest stable packages and will give you a solid foundation for subsequent actions.


## Prerequisites

A newly activated Ubuntu server, preferably setup with SSH keys. Log into the server as root with your server-ip-address.

```
ssh root@SERVER_IP_ADDRESS
```
Complete the login process by accepting the warning about host authenticity, if it appears, then providing you root authentication (password or private key). If it is your first time logging into the server, with a password, you will also be prompted to change the root password.

The root user is the administrative user in a Linux environment that has very broad privileges. Because of the heightened priviledges of the account, you are discouraged from using it on a regular basis. This is because part of the power inherent with the root aaccount is the ability to make very destructive changes, even by accident.

The next step is to set up an alternative user account with a reduced scope of influence for day-to-day work. You can use the `sudo` command to gain increased privileges during the times when you need them.


## Setup and Initial Server

### Step 1 _ Create a New User

For security reasons, it is advisable to be performing daily computing tasks using the root account. Instead, it is recommended to create a standard user that will be using `sudo` to gain administrative privileges. For this tutorial, assume that we are creating a user named joe. To create the user account type:

```
add user joe
```
Set a password for the new user. You'll be prompted to input and confirm a password.

```
passwd joe
```
Add the new user to the wheel group so that it can assume root privileges using sudo.

```
gpasswd -a joe wheel
```

Finally, open another terminal on your local machine and use the following command to add your SSH key to the new user's home directory on the remote server. You will be prompted to authenticate before the SSH key is installed.

```
ssh-copy-id joe@server-ip-address
```
After the key has been installed, log into the server using the new user account.

```
ssh -l joe server-ip-address
```
OR 

```
ssh joe@server-ip-address
```

### Step 2 _ Disallow Root Login and Password Authentication

Since you can now login as standard user using SSH keys, a good security practice is to configure SSH so that the root login and password authentication are both disallowd. Both settings have to configured in the SSH daemon's configuation file. So, open it using vim.

```
sudo vim /etc/ssh/sshd_config
```
Look for the PermitRootLogin line, uncomment it and set the value to no.

```
PermitRootLogin     no
```
Do the same for the PasswordAuthentication line, which should be uncommented already:

```
PasswordAuthentication      no
```

Save and close the file. To apply the new settings, reload SSH.

```
sudo systemctl reload sshd
```

### Step 3 _ Configure the Timezones and Network Time Protocol Synchronization

The next step is to adjust the localization settings for your server and configure the Network Time Protocol (NTP) synchronization.

The first step will ensure that your server is operating under the corrent time zone. The second step will configure your system to synchronize its system clock to the standard time maintained by the global network of NTP servers. This will help prevent inconsistent behavior that can arise from out-of-sync clocks.

#### 3.1 Configure Timezones

Our first step is to set our server's timezone. This is a very simple procedure that can accomplished using the timedatectl command:

First, take a look at the available timezones by typing:

```
sudo timedatectl list-timezones
```

Grep possible Asian timezones

```
sudo timedatectl list-timezones | grep Asia
```

This will give you a list of the timezones available for your server. When you find the region/timezone setting that is correct for your server, set it by typing:

```
sudo timedatectl set-timezone region/timezone
```

For instance, to set it to Indian Standard Time, you can type:

```
sudo timedatectl set-timezone Asia/Kolkata
```

Your system will be updated to use the selected timezone. You can confirm this by typing:

```
sudo timedatectl
```

### 3.2 Configure NTP Synchronization

Now that you have your timezone set, we should configure NTP. This will allow you computer to stay in sync with other servers, leading to more predictability in operations that rely on having the correct time.

Ubuntu installs timesyncd for the purpose of ntp synchronization. Though this is fine for most purposes, some applications that are sensitive to even the slightest perturbations in time may be better served by ntpd, as it uses more sophisticated techniques to constantly and gradually keep the system time on track.

Before installing ntpd, we should turn off timesyncd:

```
sudo timedatectl set-ntp off
```

Verify that timesyncd is off using:

```
timedatectl
```

Look for `systemd-timesyncd.service active: no` in the output. This means `timesyncd` has been stopped.

We can now install the ntp package with apt-get

```
sudo apt-get install ntp
```

`ntpd` will be started automatically after install.

To enable the service so that it is automatically started each time the server boots:

```
sudo systemctl enable ntp
```
Your server will now automatically correct its system clock to align with the global servers.


### Step 4 _ Setup FirewallD (Optional)

After setting up the bare minimum configuration for a new server. there are additional steps that are highly recommended in most cases.

#### Prerequisites

In this guide, we will be focusing on configuring some optional but recommended components. This will involve setting your system up with a firewall and a swap file.

#### Configuring a basic firewall 

Firewalls provide a basic level of security for your server. These applications are responsible for denying traffic to every port on your server with exceptions for ports/services you have approved. FirewallD like ufw is a frontend to the iptables protocol in linux. FirewallD provides a tool called firewall-cmd that can be used to configure your firewall policies. Our basic strategy will be to lock down everything that we do not have a good reason to keep open.

The firewalld service has the ability to make modifications without dropping current connections, so we can turn it on before creating our exceptions:

```
sudo systemctl start firewalld
```

Now that the service is up and running, we can use the firewall-cmd utility to get and set policy information for the firewall. The firewalld application uses the concept of "zones" to labed the trustworthiness of the other hosts on a network. This labelling gives us the ability to assign different rules depending on how much we trust a network.

In this guide, we will only be adjusting the policies for the default zone. When we reload our firewall, this will be the zone applied to our interfaces. We should start by adding exceptions to our firewall for approved services. The most essential of these is SSH, since we need to retain remote administrative access to the server.

You can enable the service by name (eg ssh-daemon) (MUST ENABLE) by typing:

```
sudo firewall-cmd --permanent --add-service=ssh
```

or disable it by typing:

```
sudo firewall-cmd --permanent --remove-service=ssh
```

or enable custom port by typing:

```
sudo firewall-cmd --permanent --add-port=4200/tcp
```

This is the bare minimum needed to retain administrative access to ther server. If you plan on running additional services, you need to open firewall for those as well.

If you plan to run web server with SSL/TLS enabled, you should allow traffic for https as well:

```
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
```

If you need SMTP email enabled, you can type:

```
sudo firewall-cmd --permanent --add-service=smtp
```

This is the bare minimum needed to retain administrative access to ther server. If you plan on running additional services, you need to open firewall for those as well.

If you plan to run web server with SSL/TLS enabled, you should allow traffic for https as well:

```
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
```

If you need SMTP email enabled, you can type:

```
sudo firewall-cmd --permanent --add-service=smtp
```

To see any additional services that you can enable by name, type:

```
sudo firewall-cmd --get-services
```

When you are finished, you can see the list of exceptions that will be implemented by typing:

```
sudo firewall-cmd --permanent --list-all
```

When you are ready to implement the changes, reload the firewall:

```
sudo firewall-cmd --reload
```

If, after testing, everything works as expected, you should make sure the firewall will be started at boot:

```
sudo systemctl enable firewalld
```

Remember that you will have to explicitly open the firewall (with services or ports) for any additional services that you may configure later.


### Step 5 _ Enable the IPTables Firewall (Optional for advanced)

By default, the active firewall application on a newly activated Ubuntu server is IPTables. Though FirewallD is good replacement for IPTables, many security applications still do not have support for it. So if you'll be using any of those applications, like OSSED HIDS, it's best to disable/uninstall FirewallD.

#### Allow Traffic Through thr Firewall

Since you'll most likely be going to use your new server to host some websites at some point, you'll have to add new rules to the firewall to allow SSH, HTTP and HTTPS traffic.

To accomplish that, open the IPTables file:

```
sudo vim /etc/sysconfig/iptables
```

Add the rules for SSH (port 22), HTTP (port 80) and HTTPS (port 443) traffic, so that portion of the file appears as shown in the code block below.

```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
```

Save and close the file, then reload IPTables.

```
sudo systemctl reload iptables
```

With the above step completed, your server should now be reasonably secure and be ready for use in production.


### Step 6 _ Optimized system configuration for maximum concurrency

In default Ubuntu is not fully optimized to make full use of available hardware.
This means it might fail under high load. So we need to config the sysctl.conf file for optimization.

#### 1. Open sysctl.conf file

```
sudo vim /etc/sysctl.conf
```

#### 2. Edit file as following configuration

```Ini
# Increase number of incoming connections
net.core.somaxconn = 250000

# Increase the maximum amount of option memory buffers
net.core.optmem_max = 25165824

# Increase Linux auto tuning TCP buffer limits
# min, default, and max number of bytes to use
# set max to at least 4MB, or higher if you use very high BDP paths
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 65536

# Decrease the time default value for tcp_fin_timeout connection
net.ipv4.tcp_fin_timeout = 15

# Number of times SYNACKs for passive TCP connection.
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2

# Decrease the time default value for connections to keep alive
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

# Enable timestamps as defined in RFC1323:
net.ipv4.tcp_timestamps = 1

# Allowed local port range
net.ipv4.ip_local_port_range = 2000 65000

# Turn on window scaling which can enlarge the transfer window:
net.ipv4.tcp_window_scaling = 1

# Limit syn backlog to prevent overflow
net.ipv4.tcp_max_syn_backlog = 30000

# Avoid a smurf attack
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Turn on protection for bad icmp error messages
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Turn on syncookies for SYN flood attack protection
net.ipv4.tcp_syncookies = 1

# Protect Against TCP Time-Wait
net.ipv4.tcp_rfc1337 = 1

# Turn on and log spoofed, source routed, and redirect packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# No source routed packets here
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Turn on reverse path filtering
net.ipv4.conf.all.rp_filter = 1

# Make sure no one can alter the routing tables
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# Don't act as a router
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Turn on execshild protection
kernel.randomize_va_space = 2

# Optimization for port usefor LBs Increase system file descriptor limit
fs.file-max = 65535

# Allow for more PIDs (to reduce rollover problems); may break some programs 32768
kernel.pid_max = 65536

# Increase TCP max buffer size setable using setsockopt()
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 87380 16777216

# Increase the tcp-time-wait buckets pool size to prevent simple DOS attacks
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1

# !! Disable ipv6 for security (!! Optional if you're using ipv6 !!)
net.ipv6.conf.all.disable_ipv6= 1
```

#### Reload the systemctl file

```
sysctl -p
```

## Update and Upgrade

To update installed packages

```
sudo apt-get update
```

To upgrade installed packages

```
sudo apt-get upgrade
```

## OpenSSL

**OpenSSL** is a software library to be used in applications that need to secure communications against eavesdropping or need to ascertain the identity of the party at the other end. It has found wide use in internet web servers, serving a majority of all web sites.

**Application-Layer Protocol Negotiation (ALPN)** is a Transport Layer Security (TLS) extension for application layer protocol negotiation. ALPN allows the application layer to negotiate which protocol should be performed over a secure connection in a manner which avoids additional round trips and which is independent of the application layer protocols. It is used by HTTP/2.

**Due to TLS False Start was disabled in Google Chrome from version 20 (2012) onward except for websites with the earlier Next Protocol Negotiation (NPN) extension. NPN was replaced with a reworked version, ALPN. On July 11, 2014, ALPN was published as RFC 7301.**


### Why Has HTTP/2 Stopped Working for Users of the New Version of Google Chrome?

As of May 2016, approximately 8% of the world's websites are accessible over HTTP/2. These websites all use SSL/TLS, because browsers that support HTTP/2 only upgrade the connection to the new standard if SSL/TLS is also in use. The vast majority of sites _ those running
NGINX and LiteSpeed _ depend on OpenSSL's implementation of NPN or ALPN to upgrade to HTTP/2.

**OpenSSL added ALPN support on January 2015, in version 1.0.2. Versions 1.0.1 and earlier do not support ALPN.**

Unlike a standalone web server like NGINX, OpenSSL is a core operating system library that is used by many of the packages shipped as part of a modern Linux operating system. To ensure the operating system is stable and reliable, OS distributors do not make major updates
to packages such as OpenSSL during the lifetime of each release. They do backport critical OpenSSL patches to their supported versions of OpenSSL to protect their users against OpenSSL vulnerabilities. They do not backport new features, particularly those which change the ABI of essential shared libraries.

<p align="center">
The table summarizes operating system support for ALPN and NPN.
  <a href="https://www.nginx.com/blog/supporting-http2-google-chrome-users/" target="_blank">
    <img src="https://cdn.rawgit.com/jukbot/secure-centos/master/alpn_os_support.PNG" alt="OS that support ALPN"/>
  </a>
</p>

As you can see, only Ubuntu >= 16.04 LTS supports ALPN. This means that if you're running your website on any other major operating system, the OpenSSL version shipped with the operating system does not support ALPN and Chrome users will be downgraded to HTTP/1.1

So to enable HTTP/2 on ALPN in chrome browser you need to be sure that you have already installed 

**OpenSSL that supported ALPN** which is version >= 1.0.2.

According to [Wikipedia](https://en.wikipedia.org/wiki/OpenSSL#Major_version_releases)

Other OS also supports ALPN now.

### To update you linux kernel, use:

```
sudo apt-get update
sudo apt-get upgrade
```

### To upgrade OpenSSL to the latest version:

To verify the current openssl, use command:
```
openssl version
OpenSSL 1.0.1e-fips 11 Feb 2013
```

#### Download the latest version of OpenSSL and generate the config as follows:
```shell
cd /usr/local/src/
wget https://www.openssl.org/source/openssl-1.1.0-latest.tar.gz
tar -xzf openssl-1.1.0-latest.tar.gz

cd openssl-1.1.0i
./config
```

#### Compile the source code, test and then install the packages (need root permission)

```
make
make test
sudo make install
```

**Recommended:
- This will take a while depending on your CPU capacity. If you CPU has more than 1 core, you can add suffix -j4 for using 4 cores to compile the source code. For example: `make -j4` to use all 4 cores to compile the source code.**


#### Move the old openssl installed version to the root folder for backup or delete it
```
sudo mv /usr/bin/openssl /root/
```
OR
```
sudo rm /usr/bin/openssl
```

#### Create a symbolic link
```
sudo ln -s /usr/local/bin/openssl /usr/bin/openssl
```

#### Update the new lib64 to LD_LIBRARY path and then apply the config
```
export LD_LIBRARY_PATH=/usr/local/lib:/usr/local/lib64
ldconfig
```
#### Verify the OpenSSL version
```
openssl version
```

### 4 Nginx


