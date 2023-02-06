# zProtection
A look into DDoS protecting OpenVPN servers using iptables BPF

# Premise
The arms race in the world of DDoS protection has affected almost every service type. VPN servers are no exception and also become the target of large and complex DDoS attacks much like game servers, websites, and so on. It is a known fact that protecting your service from DDoS isn't something that you can do alone by plastering your server full of iptables rules so it is important that your server is with a network that also aids in DDoS defense and at least does the bulk of the work. This page is not intended to serve as a be all end all of DDoS attacks, but to simply detail findings of the zFutureNetwork which has been dealing with attacks complex, small, simple, and large since the service's existence. This page assumes that you already have a server with a DDoS protected network.



# Best Practices
Unless absolutely necessary, no server should blanket accept all traffic by default. This means that you would have to block specific attacks one by one, rather than whitelisting traffic that you need and default dropping anything that doesn't match your acceptance parameters. This is far easier to do.

To set the default policy of iptables to DROP, we can use the following command

> iptables -P INPUT DROP

But before we do this, we should ensure that we are allowing the right traffic so we don't get locked out:

> iptables -I INPUT -p tcp --dport 22 -s 10.0.0.1 -j ACCEPT

> iptables -I INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

The first command allows TCP to the SSH port which in this example is 22 from our IP address of 10.0.0.1. We do this so we do not get kicked out of SSH. Replace 10.0.0.1 with the IP address you're using. In our case, we have a static IP tunnel which we use for accessing our services so we do not need to continuously update this IP address.

The second command tells iptables to accept any packet that belongs to an ESTABLISHED connection, or a packet related to an already ESTABLISHED connection. This is absolutely crucial and is how we are able to drop all traffic not explicitly accepted without having to whitelist every service on the internet. Any connection initiated by *our* server will be marked as ESTABLISHED and allowed.

# OpenVPN Filtering
Now our initial firewall is setup, we can start with filtering inbound packets to our OpenVPN port. We don't want to accept all traffic to it because then while every port on our service is protected, our OpenVPN port won't be - this leaves a vulnerability in our DDoS defense. 

Taking a look at how the client initiates a new OpenVPN connection we can look at a capture file in Wireshark that has packets containing the first packet sent by the client to the server.

## P_CONTROL_HARD_RESET_CLIENT_V2

![Sample 1](https://i.imgur.com/Thfn3jz.png)

Looking at the Wireshark dissection for the OpenVPN protocol we can see numerous characteristics of the first OpenVPN packet, which is called P_CONTROL_HARD_RESET_CLIENT_V2. This has the byte of 0x38 which consists of the opcode 0x07 + the key ID. This first byte remains the same for all new connection attempts. This is at UDP offset 8, so udp[8] in libpcap syntax. This puts our first part of our BPF rule at udp[8]=0x38, but this is just 1 byte we're checking and isn't very secure as any packet containing the correct first byte will not be dropped. So let's keep looking.

## Session ID

![Session ID](https://i.imgur.com/2cEsdyR.png)

 The next part of the packet contains an 8 byte long session ID. This is a random value used to identify the TLS session, and because it is a random value we can't do much with this. It will not be included as en evaluated field for this guide.
 
