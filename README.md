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
 
## HMAC

![HMAC](https://i.imgur.com/UXys8dO.png)

The next part is a 20 (or sometimes 16) byte long value. This field contains the HMAC signature of the encapsulation header and will differ between all clients, so this isn't worth evaluating for this guide.

## Message ID

![Message ID](https://i.imgur.com/8cesMCu.png)

Next we have a 4 byte long message ID starting at offset 37 and is an incremental number identifying the packet sequence and is primarily used for replay protection within OpenVPN. For example, the first packet sent by the client (such as P_CONTROL_HARD_RESET_CLIENT_V2) will have a message ID of 1. This is helpful, because now we know every new packet must contain a 0x01 at the location of the message ID field inside the packet. Since it is 4 bytes long and we're dealing with new connection packets only, the message ID will be preceeded by a number of 0 bytes. Our message ID is 0x00000001. This puts our BPF rule at udp[8]=0x38 and udp[37:4]=0x00000001.

## Timestamp

![Net Time](https://i.imgur.com/sDAZKs8.png)

The next segment contains a 4 byte long timestamp of the time the packet was sent by the client in Unix Epoch format. So Converting the bytes seen in the screenshot above (0x60, 0x23, 0xc9, 0xa4) to decimal, we get 1612958116, this is our Unix Epoch time. Converting this to a human readable time and date we get Wednesday, 10 February 2021 11:55:16, which is what Wireshark is reporting to us. Using BPF, we could use some logical operators to tell iptables to check if the timestamp is possible at the time the rule is active. For example, we know that we shouldn't be receiving packets with a timestep that below 1600000000 because that dates back to September 2020. At the time of writing this, I am unsure how it would be best to implement this rule, but I suppose it doesn't hurt to tell iptables to drop packets that have an unrealistic timestamp. For example, if we wanted to tell iptables to only accept packets containing a timestamp from after September 2020, we could use the following BPF rule:

> udp[41:4]>=0x5f5e1000
This says that the timestamp must be greater than or equal to 1600000000, which is Unix Epoch time and is an ever incrementing number.

I will not be including this in the final rules for now, but with XDP, some wonderful things would be possible.

## Packet ID + Array Length

![Array Length](https://i.imgur.com/AZm75PL.png)

Our next segment contains an array length of the packet ID which is 1 byte. Since this is the first packet received, both of these things are set to 0 bytes. This is very similar to the Message ID field in the packet. Knowing this, we can end our BPF filter with udp[45]=0x00 and udp[46:4]=0x00000000.

So far we now have udp[8]=0x38 and udp[37:4]=0x00000001 and udp[45]=0x00 and udp[46:4]=0x00000000 but we should be a little bit more specific. We are using the UDP protocol and our OpenVPN port is 41100, so we should tell iptables BPF to accept packets with that port only as well. Our final BPF filter becomes 

> udp dst port 41100 and udp[8]=0x38 and udp[37:4]=0x00000001 and udp[45]=0x00 and udp[46:4]=0x00000000
![tcpdump](https://i.imgur.com/pkv8ZZG.png)
