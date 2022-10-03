# Internet For Iran
[![Twitter](https://img.shields.io/twitter/url/https/twitter.com/fold_left.svg?style=social&label=Follow%20%40InternetForIran)](https://twitter.com/InternetForIran)

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
| Developer / Sys Admin / DevOps engineer anywhere         | We have reports that V2Ray VMess and ShadowSocks are working inside Iran even at times when most other tools and protocols don't. We haven't been able to reliably deploy and test this (there are many configuration options and it's not clear which methods are working). Please create an issue or send a PR if you know how it works and how to deploy it. <br />We also need your help with improving this document: Do you see a potential security issue? Can you help make the deployment process easier or automate the installation through the use of docker containers and shell scripts? Contributions are welcome :) |

### Overview

To get around the restrictions, you need to have two servers:

- **Machine A**: a machine outside Iran (e.g. on DigitalOcean or other providers). We’ll use **100.0.0.0** as the IP address of this machine througout this document. **This document assumes the OS running on Machine A is  Ubuntu Server 20.04.**
- **Machine B**: a machine inside an Iranian datacenter with a public address. We’ll use **200.0.0.0** as the IP address of this machine throughout this document. **This document assumes the OS running on Machine B is  Ubuntu Server 20.04.** 


## Security Considerations

By following this document and setting up these machines, you will need to share the IP address of Machine B with other people in order for them to be able to connect to the VPN or Tor bridge that you set up. The government can connect this IP address with your identity through the provider that you purchased Machine B from.

For example, you buy Machine B from Afranet, and share the VPN connection details with your friends which includes the IP address 200.0.0.0. One of your friends who has this information on his phone gets arrested and the security forces find this IP address on his phone, they can easily check with Afranet and find out the identity of the person who purchased this machine from them and come after you.

As much as possible, try to get one of your friends outside Iran (or a use the identity and debit card of a recently diseased person) to purchase Machine B so that even if the identity of the owner of the machine is leaked, no one can be arrested.

Also, make sure to disable access logs on Machine B as soon as you get access to it (otherwise your own IP address will be logged on the Machine B, and if the machine is compromised it can be traced back to you).
We'll explain how to do this later.

## Methods

### Method 1: [OpenConnect](openconnect/README.md)

### Method 2: [Shadowsocks](shadowsocks/README.md)