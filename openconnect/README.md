# OpenConnect
We're going to install a VPN server on **Machine A** (outside Iran) and connect to this VPN server from **Machine B** (inside an Iranian datacenter). Then we can install a VPN server and a Tor bridge on **Machine B**, and share the connection details with people we know.

## 1. Machine A Configuration
> We recommend that you buy and use a new machine for doing this with nothing else on it. But it can be done on an existing machine as well. We are using Docker to set up the VPN server and it shouldn't have side effects on the rest of your system.

This is the server that is outside Iran. You might have issue connecting to this server directly using your residential or mobile network connections. If that’s the case, you can first ssh into Server B and then ssh into Server A from there.

```bash
you@localhost:~$ ssh root@200.0.0.0
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)
root@200.0.0.0:~# ssh root@100.0.0.0
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)
root@100.0.0.0:~# 
```
### 1.1 Install and configure the OpenConnect server

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

## 2. Machine B Configuration

> Use a brand new machine for this part. The changes we make  to the routes and the VPN connection we initiate from Machine B will cause disruptions for other software you might be running on this machine.

Connect to Machine B:

```bash
you@localhost:~$ ssh root@200.0.0.0
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)
root@200.0.0.0:~#
```

### 2.1 Disable access logs

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

### 2.2 Change sources.list

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

### 2.3 Configure routes

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

### 2.4 Install and configure the OpenConnect client

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

### 2.5 Install and configure the OpenConnect server

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

### 2.6 Install and configure Wireguard and IPSec VPN servers

OpenConnect usually has problems connecting on Apple products. You can use [Algo](https://github.com/trailofbits/algo) to install Wireguard and IPSec VPNs on Machine B so that Apple products can connect as well. Follow the instructions on https://github.com/trailofbits/algo and choose Local installation mode to install it locally on Machine B. More information here: https://github.com/trailofbits/algo/blob/master/docs/deploy-to-ubuntu.md

You can download and  use this file as `config.cfg` for the Algo installation: https://pastebin.com/raw/iARF0fGL  
This configuration file includes 100 users by default and has WireGuard and IPSec enabled. 

### 2.7 Tor

Tor connections are unstable. There are two methods for getting it work, and each method might work at different times. You can do both and test their connection to see which one works for you best.

#### 2.7.1 Method 1: Install and configure a Tor bridge inside Iran

Installing a Tor bridge will help people connect to Tor through Machine B which is easily accessible from within Iran. First make sure that the VPN is connnected (step 3.4). Then install Tor from the Ubuntu repositories to get an initial Tor connection up and runnning, then add the official Tor repository and reinstall/update Tor from the official repo.

```bash
root@200.0.0.0:~# apt install tor 
root@200.0.0.0:~# apt install apt-transport-tor
root@200.0.0.0:~# CODENAME=`lsb_release -c | grep Codename | cut -d: -f2 | tr -d [:space:] `
root@200.0.0.0:~# ARCH=`dpkg --print-architecture`
root@200.0.0.0:~# echo "deb [arch=$ARCH signed-by=/usr/share/keyrings/tor-archive-keyring.gpg] tor://apow7mjfryruh65chtdydfmqfpj5btws7nbocgtaovhvezgccyjazpqd.onion/torproject.org $CODENAME main" > /etc/apt/sources.list.d/tor.list
root@200.0.0.0:~# apt update
root@200.0.0.0:~# apt install tor obfs4proxy
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

Restart Tor and get your bridge line:

```bash
root@200.0.0.0:~# systemctl restart tor
root@200.0.0.0:~# cat /var/lib/tor/pt_state/obfs4_bridgeline.txt 
# obfs4 torrc client bridge line
#
# This file is an automatically generated bridge line based on
# the current obfs4proxy configuration.  EDITING IT WILL HAVE
# NO EFFECT.
#
# Before distributing this Bridge, edit the placeholder fields
# to contain the actual values:
#  <IP ADDRESS>  - The public IP address of your obfs4 bridge.
#  <PORT>        - The TCP/IP port of your obfs4 bridge.
#  <FINGERPRINT> - The bridge's fingerprint.

Bridge obfs4 <IP ADDRESS>:<PORT> <FINGERPRINT> cert=yyyyyyyy iat-mode=0
```

Copy `Bridge obfs4 <IP ADDRESS>:<PORT> <FINGERPRINT> cert=xxxxxxxx iat-mode=0` and replace `<IP ADDRESS` with `200.0.0.0` (Machine B IP address), replace `<PORT>` the port specified for `ServerTransportListenAddr` in `/etc/tor/torrc` (9888 by default), and `<FINGERPRINT>` with the output of this command:

```bash
root@200.0.0.0:~# cat /var/lib/tor/fingerprint
Unnamed xxxxxxxx
```

The bridge line should look like this now:

```
Bridge obfs4 200.0.0.0:9888 Unnamed xxxxxxxx cert=yyyyyyyy iat-mode=0
```



> Currently the Tor fails to confirm reachability of the bridge and you'll see `Your server has not managed to confirm reachability for its ORPort(s)` in `/var/log/syslog`. But you can connect to your bridge by specifying a custom bridge in Orbot or Tor Browser.

#### 2.7.2  Method 2: Proxy Tor traffic from Iran to a bridge outside Iran

This method was developed by [@meskio](https://github.com/meskio) and originally published [here](https://github.com/net4people/bbs/issues/127). We're just changing the vocabulary so that it matches the rest of this document.

First, you need to set up a bridge on Machine A (outside Iran). We'll do this using Docker - install it if you haven't already by following the instructions here https://docs.docker.com/engine/install/ubuntu/#set-up-the-repository

```bash
root@100.0.0.0:~# mkdir bridge
root@100.0.0.0:~# cd bridge
root@100.0.0.0:~/bridge# wget https://gitlab.torproject.org/tpo/anti-censorship/docker-obfs4-bridge/-/raw/main/docker-compose.yml
```

Edit `bridge/.env` with the following content, changing `you@email.com` with an email address that you have access to but cannot be traced back to you:

```ini
# Set required variables
OR_PORT=3344
PT_PORT=3355
EMAIL=your@email.com

# If you want, you could change the nickname of your bridge
#NICKNAME=DockerObfs4Bridge

# Configure the bridge so it will not be distributed by bridgedb:
OBFS4_ENABLE_ADDITIONAL_VARIABLES=1
OBFS4V_BridgeDistribution=none
```

Start the bridge:

```bash
root@100.0.0.0:~/bridge# docker compose up -d
```

Get it's bridge line:

```bash
root@100.0.0.0:~/bridge# docker exec bridge-obfs4-bridge-1 get-bridge-line
obfs4 100.0.0.0:3355 AAABBBBCCCDDDD cert=abcdx iat-mode=0
```

Then on Machine B (inside Iran), you have two options for forwarding the Tor traffic to your bridge on Machine A:

1. Using SSH port forwarding
2. Using kcptun

##### Using SSH port forwarding

```bash
root@200.0.0.0:~# ssh -L 4457:127.0.0.1:4457 100.0.0.0:3355
```

##### Using kcptun

kcptun is a network enhancement proxy that tunnel a stream based traffic over a UDP transport protocol.

Run the following commands on Machine A:

```bash
root@100.0.0.0~# mkdir kcptun
root@100.0.0.0~# cd kcptun
root@100.0.0.0~/kcptun# wget https://github.com/xtaci/kcptun/releases/download/v20220628/kcptun-linux-amd64-20220628.tar.gz
root@100.0.0.0~/kcptun# server_linux_amd64 -t "127.0.0.1:3355" -l "0.0.0.0:7923" -mtu 1400 --nocomp -sndwnd 16384 --rcvwnd 16384 --datashard 0 --parityshard 0 --crypt aes --smuxver 2 --key "MY_PRE_SHARED_KEY"
```

Make sure to replace `MY_PRE_SHARED_KEY` for the `--key` parameter with a randomly generated string, and write it down. We'll need this value in a moment.

and then on Machine B:

```bash
root@200.0.0.0~# mkdir kcptun
root@200.0.0.0~# cd kcptun
root@200.0.0.0~/kcptun# wget https://github.com/xtaci/kcptun/releases/download/v20220628/kcptun-linux-amd64-20220628.tar.gz
root@200.0.0.0~/kcptun# client_linux_amd64 -l "0.0.0.0:3355" -r "100.0.0.0:7923" -mtu 1400 --nocomp -sndwnd 16384 --rcvwnd 16384 --datashard 0 --parityshard 0 --crypt aes --smuxver 2 --key "MY_PRE_SHARED_KEY"
```

Replace `MY_PRE_SHARED_KEY` with the same random string from the previous step. Also change `100.0.0.0` with the IP  address of Machine A.

### 2.8 Set up a standalone Snowflake Proxy

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

## 3. Connecting to the VPN and Tor from your local devices

### 3.1 OpenConnect

You need to install the OpenConnect app on your devices (Android, Windows and Linux) in order to connect to your VPN.

Once installed, use `200.0.0.0:8443` as host and the username and password you created in step 2.6. 

> Each OpenConnect VPN account can be used by multiple users simultaneously. There's no need to create multiple user profiles if you want the VPN to be used by multiple people at the same time.

### 3.2 Wireguard and IPSec

Algo saves the connection profile configuration files in `/path/to/algo/configs/200.0.0.0/` . Compress and copy this folder to your local machine and share them with your family and friends. More information here: https://github.com/trailofbits/algo#configure-the-vpn-clients

> Each Wireguard and IPSec configuration profile (user) can only be used on a single device si	multaneously. If want to connect with both your laptop and mobile phone at the same time, use two separate configuration profiles. Using a single profile for connecting from multiple devices causes connectivity issues.

### 3.3 Tor

#### 3.3.1 Bridge inside Iran

> This is for using the bridge created in section 3.7.1

In Orbot or Tor Browser, enable `Use Bridges` and select the `Custom Bridges` option, and enter the bridge line you constructed in section 3.7.1 in the `Paste Bridges` section. You should now be able to connect and use Tor in seconds.

#### 3.3.2 Bridge outside Iran with a proxy inside Iran

> This is for using the bridge created in section 3.7.2

distribute the bridgeline you got in section 3.7.2, replacing the IP address with the one of Machine B (200.0.0.0):

```
obfs4 200.0.0.0:3355 AAABBBBCCCDDDD cert=abcdx iat-mode=0
```
