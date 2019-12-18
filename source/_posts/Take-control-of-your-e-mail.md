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

Looking for a stateless e-mail stack, I have rapidly identified [docker-mailserver](https://github.com/tomav/docker-mailserver) as a good candidate. Anti-spam is based on [SpamAssassin](https://spamassassin.apache.org) (a personal choice based on my experience) and configuration is very simple (no web interface, no additional dabatase, [KISS][4])).
If [docker-mailserver](https://github.com/tomav/docker-mailserver) does not fit you, have a look at those alternatives: [mailcow](https://github.com/mailcow/mailcow-dockerized) or [Mailu](https://github.com/Mailu/Mailu).

This (almost) stateless approach enables to easily upgrade or downgrade the VM (vertical scaling).

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


#### 1. Instanciate a VM

If needed, push your SSH public key to OpenStack region `openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey`

We can not purchase any IP without a running instance. So we will instanciate the VM without any OS:
{% codeblock lang:Bash line_number:false %}
# Download cloudinit script that will help later to setup our VM
curl -fsSL https://raw.githubusercontent.com/gmasse/emailgw/master/cloudinit
# Instanciate the VM
openstack server create --flavor s1-4 --key-name mykey --user-data cloudinit --image "rescue-ovh" email
{% endcodeblock %}
{% colorquote info %}
The [cloudinit script](https://github.com/gmasse/emailgw/blob/master/cloudinit) will update the system, install packages for [docker](https://www.docker.com) and [unbound](https://nlnetlabs.nl/projects/unbound "a dns resolver"), and configure network with local dns and the static IP.
{% endcolorquote %}


#### 2. Purchase a 'Static' IP

Using Customer Interface:
![Screenshot](https://github.com/ovh/docs/raw/develop/pages/platform/public-cloud/buy_failover_ip/images/buyfailoverip1.png)

And assign it to the VM named `email`.


#### 3. Add Block Storage

There are two types of block storage: classic and high-speed (`openstack volume type list`). Performances of classic storage seems acceptable for an entry level mail gateway.

Create a 10GB volume and attache the volume to the instance:
{% codeblock lang:Bash line_number:false %}
# Create the volume
openstack volume create --type classic --size 10 email_storage
# Attach the volume to the VM
openstack server add volume email email_storage --device=/dev/sdb
{% endcodeblock %}


#### 4. Build the VM

Pass the Static IP as meta-data (replace `1.2.3.4` with your own IP) and build the VM:
{% codeblock lang:Bash line_number:false %}
openstack server rebuild --property static_ip=1.2.3.4 --image "Ubuntu 18.04" email
{% endcodeblock %}


#### 5. Setup Storage

Connect to the VM: `ssh ubuntu@1.2.3.4`
{% codeblock lang:Bash line_number:false %}
# LVM setup
sudo pvcreate /dev/sdb
sudo vgcreate vg-mail /dev/sdb
sudo lvcreate -n lv-mail -l 100%FREE vg-mail
# Format filesystem
sudo mkfs -t ext4 -L MAIL -E nodiscard /dev/vg-mail/lv-mail
# Mount it
sudo mkdir -p /mnt/mail
echo -e 'LABEL=MAIL\t/mnt/mail\text4\tdefaults,noatime\t0 2' | sudo tee -a /etc/fstab
sudo mount /mnt/mail
{% endcodeblock %}


#### 6. Configure local DNS resolver

{% colorquote info %}
Many Anti-spam blacklist systems rely on DNS, some of those blacklists rate-limit can be exceeded when using mutualized DNS resolver (your provider or global anycast like [Quad9](https://www.quad9.net)). Using your own local resolver with your Static IP would solve this issue.
{% endcolorquote %}

Configure [unbound](https://nlnetlabs.nl/projects/unbound) to allow local resolution and use the Static IP for external request (replace `1.2.3.4` with your own IP):
{% codeblock /etc/unbound/unbound.conf.d/docker.conf lang:plaintext %}
server:
    interface: 127.0.0.1
    interface: ::1
    interface: 172.17.0.1
    access-control: 172.16.0.0/12 allow
    outgoing-interface: 1.2.3.4
{% endcodeblock %}
Reload service: `sudo systemctl reload unbound`.

{% colorquote success %}
To test your local DNS resolver, you can use a DNS reflector (like [DNS Paranoia](https://dnsparanoia.com/debug_dns_with_reflector.php)) and check it answers your Static IP:
{% endcolorquote %}
{% codeblock lang:Shell line_number:false %}
$ dig +noall +answer reflect.dnsp.co
reflect.dnsp.co.	0	IN	A	1.2.3.4
{% endcodeblock %}

#### 7. Generate Let's encrypt SSL certificate

To generate certificate, run:
{% codeblock lang:Bash line_number:false %}
sudo docker run --rm -ti -v $PWD/log/:/var/log/letsencrypt/ -v $PWD/etc/:/etc/letsencrypt/ -p 80:80 certbot/certbot certonly --standalone -d mail.mydomain.com
{% endcodeblock %}


#### 9. Configure firewall

Review the [rules](https://github.com/gmasse/emailgw/tree/master/etc/iptables), then copy and apply:
{% codeblock lang:Bash line_number:false %}
sudo curl -o /etc/iptables/rules.v4 https://raw.githubusercontent.com/gmasse/emailgw/master/etc/iptables/rules.v4
sudo curl -o /etc/iptables/rules.v6 https://raw.githubusercontent.com/gmasse/emailgw/master/etc/iptables/rules.v6
sudo ip6tables-restore --noflush < /etc/iptables/rules.v6
sudo iptables-restore --noflush < /etc/iptables/rules.v4
{% endcodeblock %}

{% colorquote warning %}
Do not restart the service `iptables-persistent` because it will overwrite the Docker default rules and prevent from accessing the container.
{% endcolorquote %}


#### 10. Configure the container

Download the configuration files:
{% codeblock lang:Bash line_number:false %}
sudo mkdir /mnt/mail/docker-mailserver
sudo chown ubuntu:ubuntu /mnt/mail/docker-mailserver
ln -s /mnt/mail/docker-mailserver/ ~/docker-mailserver
cd ~/docker-mailserver/
curl -o setup.sh https://raw.githubusercontent.com/tomav/docker-mailserver/master/setup.sh; chmod a+x ./setup.sh
curl -o docker-compose.yml https://raw.githubusercontent.com/gmasse/emailgw/master/docker-mailserver/docker-compose.yml
curl -o env-mailserver https://raw.githubusercontent.com/gmasse/emailgw/master/docker-mailserver/env-mailserver
{% endcodeblock %}
Update `env-mailserver` according to your setup (`OVERRIDE_HOSTNAME` and `POSTMASTER_ADDRESS`).
{% colorquote info %}
By default, [docker-mailserver](https://github.com/tomav/docker-mailserver) uses `Maildir` format for mailboxes. This is a very common format where each e-mail is stored in one file. For better performances and/or to use [Alternate storage](https://wiki.dovecot.org/MailboxFormat/dbox#Alternate_storage), you can rely on Dovecot `sdbox` and `mdbox` (supported since the merge of my [pull request](https://github.com/tomav/docker-mailserver/pull/1314)).
{% endcolorquote %}

Update and save `.env` file:
{% codeblock ~/docker-mailserver/.env lang:plaintext %}
HOSTNAME=mail
DOMAINNAME=mydomain.com
CONTAINER_NAME=mail
EXTERNAL_IP=1.2.3.4
{% endcodeblock %}


#### 11. Run it

Create at least one e-mail account: `sudo ./setup.sh email add <user@domain> [<password>]`
Then launch the container: `sudo docker-compose up -d mail`


#### 12. To go further

[Generate DKIM keys](https://github.com/tomav/docker-mailserver#generate-dkim-keys)
[Update the container](https://github.com/tomav/docker-mailserver#restart-and-update-the-container)


## To be continued...

Let's encrypt SSL auto-renew
Coudinit: add storage and dns configuration
Tiered storage, using Dovecot Alternate storage feature: recent emails on high-speed block storage, archives on normal speed
Volume (auto?) resize
IPv6
E-mail configuration auto-discover



[1]: <https://venturebeat.com/2019/02/06/gmail-is-now-blocking-100-million-more-spam-emails-a-day-thanks-to-tensorflow/>
[2]: <https://tools.ietf.org/html/rfc5321#section-4.5.4.1> '4-5 days recommended in RFC 5321'
[3]: <https://en.wikipedia.org/wiki/General_Data_Protection_Regulation> 'General Data Protection Regulation'
[4]: <https://en.wikipedia.org/wiki/KISS_principle> 'Keep It Simple, Stupid'
