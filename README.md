## Introduction
Internet is heavily restricted on mobile (3G/4G) and residential (ADSL/TD-LTE) networks and connecting to VPNs and websites outside Iran is close to impossible, Tor is not working reliably as the Tor bridges are outside Iran and mostly inaccessible to people inside Iran. On the other hand, the government has not yet blocked the Internet access on machines located inside Iranian data centers, and people can easily connect to these websites and servers.

### How can I help?

We need servers and we need help setting those servers up.

| Who am I?                                                | How can I help?                                              |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| Iranian expat                                            | If you can, purchase a server inside Iran and send us the IP address and ssh credentials by emailing InternetForIran@proton.me. We will set up the server and send the VPN details back to you to share with your friends and family inside Iran. |
| Tech company owner/founder/C-level exec inside Iran      | You probably already have servers inside Iran. Use this document to convert one of your servers (or a new server) to set up a VPN for your team. Ask your team to use it and share the VPN details and the Tor bridge with their family and friends. You'll have a real business usecase to justify creating the VPN server even if the government comes to your door. |
| Developer / Sys Admin / DevOps engineer inside Iran      | **We do not recommend you purchasing servers from Iranian data centers for setting up VPN services  yourself. The server IP address which you will share with your friends and family can be easily traced back to your identity.** <br />Contact your friends and family outside Iran and ask them to purchase a server from an Iranian datacenter and give you the details. Set it up by following this guide and share the details with your friends and family. |
| Someone who has a recently deceased relative inside Iran | First of all, sorry for your loss. :( One way you can help is to use the identity and debit card of your deceased family member to purchase a server in one of the Iranian datacenters and send us the IP address and ssh credentials by emailing InternetForIran@proton.me. We will set up the server and send the VPN details back to you to use and share with your friends and family. The government won't be able to arrest your deceased family member, and won't be able to prove that you used their identity and debit card information to purchase the servers.<br />**Remember, it is important that you use both their identity AND debit card to make the purchase, if you use your own card or identity, the government will be able to trace it back to you.** |
| Regular person in Iran                                   | **We do not recommend you purchasing servers from Iranian data centers for setting up VPN services  yourself. The server IP address which you will share with your friends and family can be easily traced back to your identity.**<br /> Send this document to your technical friends. Ask your family members outside Iran to purchase server. Retweet and like our tweets and get the word out. |
| VPN provider outside Iran                                | We need VPNs outside Iran (helps us replace Machine A below with a VPN). Please send us VPN connection details (preferably without data usage limits, OpenVPN and OpenConnect work best) by emailing InternetForIran@proton.me. |
| Hacker group                                             | If you compromise a server inside Iran and gain ssh access to, use this guide to set up a VPN server on and share the details with us and your followers. |
| Developer / Sys Admin / DevOps engineer anywhere         | We have reports that V2Ray VMess and ShadowSocks are working inside Iran even at times when most other tools and protocols don't. We haven't been able to reliably deploy and test this (there are many configuration options and it's not clear which methods are working). Please create an issue or send a PR if you know how it works and how to deploy it. <br />We also need your help with improving this document: Do you see a potential security issue? Can you help make the deployment process easier, maybe through docker or shell scripts for taking care of the installation? Contributions are welcome :) |

### Overview

To get around the restrictions, you need to have two servers:

- **Machine A**: a machine outside Iran (e.g. on DigitalOcean or other providers). We’ll use **100.0.0.0** as the IP address of this machine througout this document. **This document assumes the OS running on Machine A is  Ubuntu Server 20.04.**
- **Machine B**: a machine inside an Iranian datacenter with a public address. We’ll use **200.0.0.0** as the IP address of this machine throughout this document. **This document assumes the OS running on Machine B is  Ubuntu Server 20.04.** 

We're going to install a VPN server on **Machine A** (outside Iran) and connect to this VPN server from **Machine B** (inside an Iranian datacenter). Then we can install a VPN server and a Tor bridge on **Machine B**, and share the connection details with people we know.



## 1. Security Considerations

By following this document and setting up these machines, you will need to share the IP address of Machine B with other people in order for them to be able to connect to the VPN or Tor bridge that you set up. The government can connect this IP address with your identity through the provider that you purchased Machine B from.

For example, you buy Machine B from Afranet, and share the VPN connection details with your friends which includes the IP address 200.0.0.0. One of your friends who has this information on his phone gets arrested and the security forces find this IP address on his phone, they can easily check with Afranet and find out the identity of the person who purchased this machine from them and come after you.

As much as possible, try to get one of your friends outside Iran (or a use the identity and debit card of a recently diseased person) to purchase Machine B so that even if the identity of the owner of the machine is leaked, no one can be arrested.

Also, make sure to disable access logs on Machine B as soon as you get access to it (otherwise your own IP address will be logged on the Machine B, and if the machine is compromised it can be traced back to you).
We'll explain how to do this later.

## 2. Machine A Configuration
> We recommend that you buy and use a new machine for doing this with nothing else on it. But it can be done on an existing machine as well. We are using Docker to set up the VPN server and it shouldn't have side effects on the rest of your system.

This is the server that is outside Iran. You might have issue connecting to this server directly using your residential or mobile network connections. If that’s the case, you can first ssh into Server B and then ssh into Server A from there.

```bash
you@localhost:~$ ssh root@200.0.0.0
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)
root@200.0.0.0:~# ssh root@100.0.0.0
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)
root@100.0.0.0:~# 
```
### 2.1 Install and configure the OpenConnect server

Deploy and run ocserv using Docker. Ocserv will act as the OpenConnect server:

```bash
root@100.0.0.0:~# mkdir ocserv
root@100.0.0.0:~# cd ocserv
root@100.0.0.0:~/ocserv# wget -qO- https://pastebin.com/raw/WZymtWi2 > ocserv.conf
root@100.0.0.0:~/ocserv# touch ocpasswd
root@100.0.0.0:~/ocserv# cd 
root@100.0.0.0:~# docker run --name ocserv --privileged -e NO_TEST_USER=1 -v $PWD/ocserv/:/etc/ocserv/ -p 8443:443 -p 8443:443/udp -d tommylau/ocserv
```

This should start the docker container running the OpenConnect server. Now you need to create a new user for connecting this this server - replace `USERNAME` with whatever username you want:

```bash
root@100.0.0.0:~# docker exec -ti ocserv ocpasswd -c /etc/ocserv/ocpasswd -g "Route,All" USERNAME
Enter password: 
Re-enter password:
```

> It won't show the password you're typing, don't get confused.

## 3. Machine B Configuration

> Use a brand new machine for this part. The changes we make  to the routes and the VPN connection we initiate from Machine B will cause disruptions for other software you might be running on this machine.

Connect to Machine B:

```bash
you@localhost:~$ ssh root@200.0.0.0
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)
root@200.0.0.0:~#
```

### 3.1 Disable access logs

Edit `/etc/ssh/sshd_config`:

```bash
nano /etc/ssh/sshd_config
```

 and find the `# LogLevel INFO`. Change it to:

```
LogLevel QUIET
```

Reload `ssh` and `sshd` services:

```bash
root@200.0.0.0:~# systemctl reload ssh 
root@200.0.0.0:~# systemctl reload sshd
```

Logout and ssh into the server again.

```bash
root@200.0.0.0:~# exit
Connection to YOUR_LOCAL_IP closed.
you@localhost:~$ ssh root@200.0.0.0
```

After logging in, clear out the existing access logs:

```bash
root@200.0.0.0:~# echo > /var/log/auth.log
root@200.0.0.0:~# rm /var/log/auth.log.*
```

logout and log back in again, check and make sure `sshd` is not logging your IP in `/var/log/auth.log`:

```bash
root@200.0.0.0:~# tail /var/log/auth.log
```

You should see entries like

```
localhost systemd-logind[706]: Session 12 logged out. Waiting for processes to exit.
```

but no IP addresses.

> Please note that this does not guarantee that the government cannot find your local IP address. Once everything is set up, you should first activate your VPN connection, then ssh into Machine A and ssh into Machine B from Machine A.

### 3.2 Change sources.list

Some providers (such as Afranet) deploy the machines with their own version of sources.list which causes Ubuntu to download packages from providers' repository mirrors. To increase security, we need to change it back to the official Ubuntu repository mirrors:

```bash
root@200.0.0.0:~# nano /etc/apt/sources.list
```

Make sure all lines start with

```
deb http://archive.ubuntu.com/ubuntu
```

or 

```
deb http://archive.ubuntu.com/ubuntu
```

If you see lines like 

```
deb http://ubuntu.mirror.afranet.com/ubuntu
```

change `ubuntu.mirror.afranet.com` on those lines to `archive.ubuntu.com` and you should be good.

Then update the apt repositories:

```bash
root@200.0.0.0:~# apt update
```

### 3.3 Configure routes

To maintain ssh connection from your local network to Machine A after connecting it to the VPN server on Machine B, you need to add direct routes to Iranian IP addresses on Machine A. To do that we'll create a bash script and a systemd service to make sure that it's executed on boot.

First create the bash script and move it to `/usr/local/sbin`:

```bash
root@200.0.0.0:~# wget -qO- https://pastebin.com/raw/isEgF5tv > iran_ip_range.json
root@200.0.0.0:~# GATEWAY=`ip route show | grep 'default via' | cut -d' ' -f3`
root@200.0.0.0:~# apt install jq
root@200.0.0.0:~# for range in $(jq .[] iran_ip_range.json | sed 's/"//g' | xargs); do   echo "ip route add $range via $GATEWAY" >> /usr/local/sbin/add-routes.sh ; done;
root@200.0.0.0:~# chmod +x /usr/local/sbin/add-routes.sh
```

Replace `DEFAULT_GATEWAY_IP` with your gateway ip.

Now create the systemd service:

```bash
root@200.0.0.0:~# nano /etc/systemd/system/add-routes.service
```

Paste the following in this file and save:

```ini
[Unit]
Description=Router configuration service
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=no
RestartSec=3600min
User=root
ExecStart=/bin/bash /usr/local/sbin/add-routes.sh

[Install]
WantedBy=multi-user.target
```

Enable the `add-routes` service and start it:

```bash
root@200.0.0.0:~# systemctl daemon-reload
root@200.0.0.0:~# systemctl enable add-routes.service
root@200.0.0.0:~# systemctl start add-routes.service
```

### 3.4 Install and configure the OpenConnect client

OpenConnect is a VPN software that uses SSL for connecting and transferring network data. You need to install and configure OpenConnect client to connect to the OpenConnect server that you previously installed on Machine A:

```bash
root@200.0.0.0:~# apt install openconnect
```

Open a `screen` to be able to start the OpenConnect VPN client and leave it running after you log out of  Machine B:

```bash
root@200.0.0.0:~# screen
```

Inside the screen, start the OpenConnect client:

```bash
root@200.0.0.0:~# openconnect --user test 100.0.0.0:8443 

POST https://100.0.0.0:8443/
Connected to 100.0.0.0:8443
SSL negotiation with 100.0.0.0
Server certificate verify failed: signer not found

Certificate from VPN server "100.0.0.0" failed verification.
Reason: signer not found
To trust this server in future, perhaps add this to your command line:
    --servercert pin-sha256:xxxxxxxxxxxxxxxxxxxxx
Enter 'yes' to accept, 'no' to abort; anything else to view: yes 
Connected to HTTPS on 100.0.0.0
XML POST enabled
Please enter your username.
Group: [Route[Exclude CN]|All Proxy|All]:All
POST https://100.0.0.0:8443/auth
XML POST enabled
Please enter your username.
POST https://100.0.0.0:8443/auth
Please enter your password.
Password:
POST https://100.0.0.0:8443/auth
Got CONNECT response: HTTP/1.1 200 CONNECTED
CSTP connected. DPD 90, Keepalive 32400
Connected as 192.168.xx.xx, using SSL + LZ4, with DTLS + LZ4 in progress
DTLS handshake failed: Error in the push function.
(Is a firewall preventing you from sending UDP packets?)
```

You are now connected to the VPN server on machine A. To exit the screen, press `Ctrl+A+D`. You can resume the screen connection (to see the VPN connection output or stop the connection by pressing `Ctrl+C`) by running the following command:

```bash
root@200.0.0.0:~# screen -r
```

### 3.5 Install and configure the OpenConnect server

This can be done using the exact same process as was done on Machine A:

```bash
root@200.0.0.0:~# mkdir ocserv
root@200.0.0.0:~# cd ocserv
root@200.0.0.0:~/ocserv# wget -qO- https://pastebin.com/raw/WZymtWi2 > ocserv.conf
root@200.0.0.0:~/ocserv# touch ocpasswd
root@200.0.0.0:~/ocserv# cd 
root@200.0.0.0:~# docker run --name ocserv --privileged -e NO_TEST_USER=1 -v $PWD/ocserv/:/etc/ocserv/ -p 8443:443 -p 8443:443/udp -d tommylau/ocserv
```

This should start the docker container running the OpenConnect server. Now you need to create a new user for connecting this this server - replace `USERNAME` with whatever username you want:

```bash
root@200.0.0.0:~# docker exec -ti ocserv ocpasswd -c /etc/ocserv/ocpasswd -g "Route,All" USERNAME
Enter password: 
Re-enter password:
```

> It won't show the password you're typing, don't get confused.

### 3.6 Install and configure Wireguard and IPSec VPN servers

OpenConnect usually has problems connecting on Apple products. You can use [Algo](https://github.com/trailofbits/algo) to install Wireguard and IPSec VPNs on Machine B so that Apple products can connect as well. Follow the instructions on https://github.com/trailofbits/algo and choose Local installation mode to install it locally on Machine B. More information here: https://github.com/trailofbits/algo/blob/master/docs/deploy-to-ubuntu.md

You can download and  use this file as `config.cfg` for the Algo installation: https://pastebin.com/raw/iARF0fGL  
This configuration file includes 100 users by default and has WireGuard and IPSec enabled. 

### 3.7 Install and configure a Tor bridge

Installing a Tor bridge will help people connect to Tor through Machine B which is easily accessible from within Iran. First make sure that the VPN is connnected (step 3.4). Then install Tor from the Ubuntu repositories to get an initial Tor connection up and runnning, then add the official Tor repository and reinstall/update Tor from the official repo.

```bash
root@200.0.0.0:~# apt install tor
root@200.0.0.0:~# apt install apt-transport-tor
root@200.0.0.0:~# CODENAME=`lsb_release -c | grep Codename | cut -d: -f2 | tr -d [:space:] `
root@200.0.0.0:~# ARCH=`dpkg --print-architecture`
root@200.0.0.0:~# echo "deb [arch=$ARCH signed-by=/usr/share/keyrings/tor-archive-keyring.gpg] tor://apow7mjfryruh65chtdydfmqfpj5btws7nbocgtaovhvezgccyjazpqd.onion/torproject.org $CODENAME main" > /etc/apt/sources.list.d/tor.list
root@200.0.0.0:~# apt update
root@200.0.0.0:~# apt install tor
```

Now empty out `/etc/tor/torrc` and open it using an editor:

```bash
root@200.0.0.0:~# echo > /etc/tor/torrc
root@200.0.0.0:~# nano /etc/tor/torrc
```

and paste this into the file, making sure to replace  `200.0.0.0` with the real IP address of Machine B and YOU@EMAIL.COM with an email address that you have access to but cannot be traced back to you:

```
BridgeRelay 1

# Replace "TODO1" with a Tor port of your choice.
# This port must be externally reachable.
# Avoid port 9001 because it's commonly associated with Tor and censors may be scanning the Internet for this port.
ORPort 200.0.0.0:8888

ServerTransportPlugin obfs4 exec /usr/bin/obfs4proxy

# This port must be externally reachable and must be different from the one specified for ORPort.
# Avoid port 9001 because it's commonly associated with Tor and censors may be scanning the Internet for this port.
ServerTransportListenAddr obfs4 0.0.0.0:9888

# Local communication port between Tor and obfs4.  Always set this to "auto".
# "Ext" means "extended", not "external".  Don't try to set a specific port number, nor listen on 0.0.0.0.
ExtORPort auto

# Replace "<address@email.com>" with your email address so we can contact you if there are problems with your bridge.
# This is optional but encouraged.
ContactInfo YOU@EMAIL.COM

```

Restart Tor:

```bash
root@200.0.0.0:~# systemctl restart tor
```

> Currently the Tor fails to confirm reachability of the bridge and you'll see `Your server has not managed to confirm reachability for its ORPort(s)` in `/var/log/syslog`. But you can connect to your bridge by specifying a custom bridge in Orbot or Tor Browser.

### 3.8 Set up a standalone Snowflake Proxy

You can also set up a standalone Snowflake proxy on Machine B to help censored users connect to the Tor network. 

The docker-compose version on Ubuntu 20.04 is a bit old, you need to first add the official docker registry and install  docker from there: https://docs.docker.com/engine/install/ubuntu/#set-up-the-repository

Once the docker installation is done, run the container:

```bash
root@200.0.0.0:~# mkdir snowflake
root@200.0.0.0:~# cd snowflake
root@200.0.0.0:~# wget https://gitlab.torproject.org/tpo/anti-censorship/docker-snowflake-proxy/-/raw/main/docker-compose.yml
root@200.0.0.0:~# docker compose up -d
```

Your snowflake proxy is now up and running and will output some usage statistics every hour on docker logs (first output will appear one hour after starting the docker container):

```
root@200.0.0.0:~# docker compose logs snowflake-proxy
```

## 4. Connecting to the VPN and Tor from your local devices

### 4.1 OpenConnect

You need to install the OpenConnect app on your devices (Android, Windows and Linux) in order to connect to your VPN.

Once installed, use `200.0.0.0:8443` as host and the username and password you created in step 2.6. 

> Each OpenConnect VPN account can be used by multiple users simultaneously. There's no need to create multiple user profiles if you want the VPN to be used by multiple people at the same time.

### 4.2 Wireguard and IPSec

Algo saves the connection profile configuration files in `/path/to/algo/configs/200.0.0.0/` . Compress and copy this folder to your local machine and share them with your family and friends. More information here: https://github.com/trailofbits/algo#configure-the-vpn-clients

> Each Wireguard and IPSec configuration profile (user) can only be used on a single device si	multaneously. If want to connect with both your laptop and mobile phone at the same time, use two separate configuration profiles. Using a single profile for connecting from multiple devices causes connectivity issues.

### 4.3 Tor

In Orbot or Tor Browser, enable `Use Bridges` and select the `Custom Bridges` option, and enter `200.0.0.0:8888` in the `Paste Bridges` section. You should now be able to connect and use Tor in seconds.


