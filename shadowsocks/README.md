# Shadowsocks

The instructions of this page is mainly derived from [here](https://bit.ly/3Swe5Ew). This method has been working smoothly even under harsh restrictions, i.e., when the government cuts off the Internet access of the citizens to the outside world.

We're going to install Shadowsocks on Machine A (outside Iran), and install HAProxy to use Machine B (inside Iran) to just send/receive the incoming traffic to/from Machine A. This is how our setup will look like:

![alt text](overview.png "Overview")

>At the time of writing this page, this method did not work on an Abrvancloud VPS instance. We do not know if this method works on all the other VPS providers. Please test and ideally report back to us as an issue.

## 1. Machine B Configuration

> We recommend that you buy and use a new machine for doing this with nothing else on it. Having that said, it can be done on an existing machine as well but be warned that other programs might have side effects on this method.

Connect to Machine B:

```bash
you@localhost:~$ ssh root@200.0.0.0
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)
root@200.0.0.0:~#
```

Make sure that you have access to the outside world:

```bash
root@200.0.0.0:~# ping 4.2.2.4

PING 4.2.2.4 (4.2.2.4) 56(84) bytes of data.
64 bytes from 4.2.2.4: icmp_seq=1 ttl=51 time=83.9 ms
PING 4.2.2.4 (4.2.2.4) 56(84) bytes of data.
```

You may press Ctrl+C to end the operation. If you do not see any output like above, it means that your server has no access to the outside world, thus you will need to change your VPS.

### 1.1 Disable access logs

Edit `/etc/ssh/sshd_config`:

```bash
root@200.0.0.0:~# nano /etc/ssh/sshd_config
```

and find the `# LogLevel INFO`. Change it to:

```text
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

```text
localhost systemd-logind[706]: Session 12 logged out. Waiting for processes to exit.
```

but no IP addresses.

> Please note that this does not guarantee that the government cannot find your local IP address. Once everything is set up, you should first activate your VPN connection, then ssh into Machine A and ssh into Machine B from Machine A.

### 1.2  Install and configure HAProxy

Install HAProxy:

```bash
root@200.0.0.0:~# apt install haproxy
```

Open its configuration file:
```bash
root@200.0.0.0:~# nano /etc/haproxy/haproxy.cfg
```

Remove all the existing contents, paste the following lines, and save the file. Remember to change `100.0.0.0` in the last line to the IP address of Machine A (outside Iran).

```text
global
        ulimit-n 1048576

defaults
        log  global
        option  dontlognull
        timeout connect 1000
        timeout client 150000
        timeout server 150000


frontend ut-in
        bind *:443
  mode tcp
        default_backend ut-out

backend ut-out
  mode tcp
        server server1 100.0.0.0:443 maxconn 20480
```

Enable and restart haproxy service:

```bash
root@200.0.0.0:~# systemctl enable haproxy.service
root@200.0.0.0:~# systemctl restart haproxy.service
```

Make sure that the service is running:

```bash
root@100.0.0.0:~# systemctl status haproxy
```

If you see `Active: active (running)` in the output, it means HAProxy is running successfully.

## 2. Machine A Configuration

> We recommend that you buy and use a new machine for doing this with nothing else on it. Having that said, it can be done on an existing machine as well but be warned that other programs might have side effects on this method.

Connect to Machine A:

```bash
you@localhost:~$ ssh root@100.0.0.0
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)
root@100.0.0.0:~#
```

### 2.1  Install and configure Shadowsocks

Install shadowsocks-rust:

```bash
root@100.0.0.0:~# mkdir ss
root@100.0.0.0:~# cd ss
root@100.0.0.0:~/ss# ss_version=$(curl -s "https://api.github.com/repos/shadowsocks/shadowsocks-rust/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
root@100.0.0.0:~/ss# curl --retry 5 -LO https://github.com/shadowsocks/shadowsocks-rust/releases/download/${ss_version}/shadowsocks-${ss_version}.x86_64-unknown-linux-gnu.tar.xz
root@100.0.0.0:~/ss# tar -xvf shadowsocks* 
root@100.0.0.0:~/ss# cp -f ssserver /usr/sbin/
```

See the version of Shadowsocks by the following command to make sure that it is accessible in the copied directory:

```bash
root@100.0.0.0:~/ss# /usr/sbin/ssserver --version
```

Create the configuration file of Shadowsocks:

```bash
root@100.0.0.0:~# mkdir /etc/ss-rust
root@100.0.0.0:~# nano /etc/ss-rust/config.json
```

Paste the following in this file and save:

```json
{
    "server": "100.0.0.0",
    "mode": "tcp_and_udp",
    "server_port": 433,
    "password": "somePassword",
    "timeout": 300,
    "method": "aes-128-gcm"
}
```

Note that you need to set IP address of Machine A (outside Iran) instead of `100.0.0.0`.Also instead of `somePassowrd` you may change it to whatever you want (at least 8 characters).

Now create the systemd service:

```bash
root@100.0.0.0:~# nano /etc/systemd/system/ssserver.service
```

Paste the following in this file and save:

```ini
[Unit]
Description=shadowsocks-rust
Documentation=https://github.com/shadowsocks/shadowsocks-rust
After=network.target

[Service]
User=root
Group=root
RemainAfterExit=yes
ExecStart=/usr/sbin/ssserver -c /etc/ss-rust/config.json
ExecReload=/usr/bin/kill -HUP \$MAINPID
ExecStop=/usr/bin/kill -s STOP \$MAINPID
LimitNOFILE=infinity
RestartSec=3s
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable the `ssservice` service and restart it:

```bash
root@100.0.0.0:~# systemctl daemon-reload
root@100.0.0.0:~# systemctl enable ssservice
root@100.0.0.0:~# systemctl restart ssservice
```

Make sure that the service is running:

```bash
root@100.0.0.0:~# systemctl status ssserver
```

If you see `Active: active (running)` in the output, it means Shadowsocks is  running successfully.

### 2.2 Generate and modify the Shadowsocks key

Generate the Shadowsocks key:

```bash
root@100.0.0.0:~# cd ~/ss
root@100.0.0.0:~/ss# chmod +x ./ssurl
root@100.0.0.0:~/ss# /ssurl --encode /etc/ss-rust/config.json
```

You should be able to see something with this format:

```text
ss://YNVzLTRyfowfgW86Hjh3ThlwMkkFHuUisRNGaw@100.0.0.0:433
```

This key needs a small change, so that the clients in Iran can connect to our Shadowsocks. At almost end of the key you see the IP address of Machine A (in this case `100.0.0.0`). This needs to be changed to the IP address of Machine B (in this case `200.0.0.0`). Open a text editor and make this change. It would be something like this (only the IP address is changed):

```text
ss://YNVzLTRyfowfgW86Hjh3ThlwMkkFHuUisRNGaw@200.0.0.0:433
```

**This is the key that needs to be entered in the Shadowsocks client applications. You shall share this key with the people that you trust, so that they can connect to the Internet too.**

## 3. Connect your device to the unrestricted Internet

For the users in Iran, depending on the device that is supposed to be connected to Shadowsocks, you would need to install a Shadowsocks client.

### 3.1 Recommended apps for different devices

After installing any of the following applications, enter the above Shadowsocks key that you generated.

[Android Outline](https://play.google.com/store/apps/details?id=org.outline.android.client&hl=de&gl=US)

[iOS Outline](https://apps.apple.com/us/app/outline-app/id1356177741)

[Windows Outline](https://github.com/Jigsaw-Code/outline-client)

[macOS Outline](https://apps.apple.com/us/app/outline-secure-internet-access/id1356178125?mt=12)

[GNU/Linux](https://github.com/Jigsaw-Code/outline-client)