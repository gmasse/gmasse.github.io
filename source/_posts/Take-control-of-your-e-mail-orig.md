---
title: Take control of your e-mail
date: 2019-11-08 18:13:02
tags:
---
When it's time to talk about regaining control of personal data, e-mail are on the top of my list.

I have never trusted Google to deal with my data (my photos, my e-mail, or my documents). Even if I admit Gmail offer a very convenient service and a wonderful anti-spam (thanks to all the data shared by [1.5 billion active users and their 10 millions of spam and malicious emails received every minute][1]).

When your business relies on digital exchanges, your e-mail provider must be very reliable. A chance that the Simple Mail Transfer Protocol is robust, allowing your precious e-mail to be retransmitted in case of outage of a gateway. In fact, a gateway could be [off-line during days][2] before the e-mail would be returned to its sender.

By the time, in addition to SMTP, other protocols and standards has been added to provide a complete e-mail service:
- IMAP (used by your client to retrieve e-mail from the server)
- TLS/SSL (for security and privacy)
- SPF, DKIM and DMARC (to authenticate senders and fight against phishing and spam)
- Sieve (to define filtering rules)

## Hey! I don't want to build a rocket. I just need an e-mail service that respect my privacy.

Answer is simple: rely on a service provided by a recognized European entity (respecting [GDPR][3], which is normally mandatory). In short, you can trust [OVHcloud e-mail solutions](https://www.ovh.ie/emails/).

# Let's do it!

For fun or if your needs are a bit specific, you can also do it yourself.

In my all life, I have built 3 times a complete e-mailing system for my personal use. Obsolescence of the underlying Debian system, and therefore security, was the main driver. This last time, I defined these requirements:
- Operating System abstraction (ie I can easily upgrade/change the OS without risking a major impact on the e-mail application stack)
- Sustainable Open-Source community supporting the entire stack
- Hardware abstraction

## Docker to the rescue

Looking for a stateless e-mail stack, I have rapidly identified [docker-mailserver](https://github.com/tomav/docker-mailserver) as a good candidate. Anti-spam is based on [SpamAssassin](https://spamassassin.apache.org) (a personal choice based on my experience) and configuration is very simple (no web interface, no additional dabatase, [KISS][4])). If [docker-mailserver](https://github.com/tomav/docker-mailserver) does not fit you, have a look at those alternatives: [mailcow](https://github.com/mailcow/mailcow-dockerized) or [Mailu](https://github.com/Mailu/Mailu).

### Requirements

#### Basic configuration

1 CPU
2 GB RAM + Swap (or 3 GB if you don't want to use swap)
8 GB Storage for System and Container (don't forget to add the Swap size if you need it)
\+ Storage for e-mail accounts
{% colorquote info %}
ClamAV is memory consuming (more than 800MB for the virus signature database). If you don't run ClamAV (for example if you use another ant-virus or gateway), you should be able to run with 512MB of RAM. 
{% endcolorquote %}

#### Advanced configuration

For e-mail accounts storage, my first thought was to use Object Storage but I was not able to find any Open-Source IMAP server. Then I have decided to go with a scalalable Block Storage solution (like Ceph-based OVH Block Storage).

For SMTP output, IP reputation is very important. Default VM's IP seems not to be a convenient choice. My preference went to 'Static' IP (like an OVH Fail-Over IP). Another advantage is I could decide to replace the VM chile keeping the same IP.

### Step by step, using OpenStack-based OVH Public Cloud

I assume a new Public Cloud Project has been created, and you have downloaded the OpenRC environment file.
First, load the environment: `source openrc.sh`.

#### 1. Purchase a 'Static' IP

Using Customer Interface:
![Screenshot](https://github.com/ovh/docs/raw/develop/pages/platform/public-cloud/buy_failover_ip/images/buyfailoverip1.png)

#### 2. Build the VM

If needed, push your SSH public key to OpenStack region:
`openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey`

Download [my cloudinit file](https://github.com/gmasse/emailgw/blob/master/cloudinit):
`curl -fsSL https://raw.githubusercontent.com/gmasse/emailgw/master/cloudinit`
and update IP with your own 'Static' IP:
`vi cloudinit`

Then boot the VM:
`nova boot --flavor s1-4 --image "Ubuntu 18.04" --key-name mykey --user-data cloudinit email`
Save the VM id:
`SRV_ID=4e960a2d-c354-45cc-9d58-e0fb84ef2dbc` (replace with your own VM id).

#### 3. Add Block Storage

There are two types of block storage: classic and high-speed (`cinder type-list`). Performances of classic storage seems acceptable for an entry level mail gateway.

To create a 10GB volume:
`cinder create --volume-type classic 10`
Then save the volume id:
`VOL_ID=0e3a4a91-a69b-41ab-b0c6-184d86f89ab4` (replace with your own volume id).

Finally attach the storage device to your VM as /dev/sdb:
`nova volume-attach $SRV_ID $VOL_ID /dev/sdb`

#### 4. Run the container


## Other benefits

Scale On Demand:
    Unlimited Horizontal Scaling for Storage (Ceph backend)
    Vertical Scaling for CPU and Memory


[1]: <https://venturebeat.com/2019/02/06/gmail-is-now-blocking-100-million-more-spam-emails-a-day-thanks-to-tensorflow/>
[2]: <https://tools.ietf.org/html/rfc5321#section-4.5.4.1> '4-5 days recommended in RFC 5321'
[3]: <https://en.wikipedia.org/wiki/General_Data_Protection_Regulation> 'General Data Protection Regulation'
[4]: <https://en.wikipedia.org/wiki/KISS_principle> 'Keep It Simple, Supide'
