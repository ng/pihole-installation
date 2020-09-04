# OpenVPN x Pi-hole on AWS Lightsail (and optional SSL)

No frills, barebones instructions to get everything working on a $3.50/mo AWS Lightsale instance. 

# Workflow
## Infrastructure
* Create AWS Lightsail instance
  * Configure firewall
* Provision and associate static IP
## From within VPS
* Install OpenVPN
* Save OpenVPN client credentials
* Install Pi-hole
* Optional:
  * SSL via certbot
  * Enable snapshots
## Finishing up
* Connect to VPN from your device

# AWS Lightsail
## Provision VPS
[Create an AWS Lightsail instance](https://lightsail.aws.amazon.com/ls/webapp/home/instances). Be sure to pay attention to the region. 
This will cost $3.50/mo without any snapshot storage (additional cost).

* Platform: Linux/Unix
* Blueprint: 
  * OS Only
  * Ubuntu 20.04 LTS
* Instance plan cost: $3.50

## Configure firewall 
Ensure the following ports are exposed through the *Networking* tab. SSH and HTTP are exposed by default.

Application | Protocol | Port | Description
----------- | -------- | ---- | -----------
SSH | TCP | 22 | Exposed by default
HTTP | TCP | 80 | Exposed by default
HTTPS | TCP | 443 | SSL, optional
Custom | UDP | 1194 | OpenVPN

# Provision and assign static IP address
[Create a static IPs and attach it to the previously created instance](https://lightsail.aws.amazon.com/ls/webapp/create/static-ip). Static IPs are free when attached to instances in the *same region*, otherwise they incur fees. 

# SSH into the VPS
SSH to the previously created instance. Amazon provides a browser based client accessible in the instance's *Connect* tab.

Switch to super user.
```
sudo su -
```

# Install OpenVPN
Run the OpenVPN install script maintained by [github.com/Nyr/openvpn-install](https://github.com/Nyr/openvpn-install) with the following:

```
wget https://git.io/vpn -O openvpn-install.sh && bash openvpn-install.sh
```

Accept all defaults. Feel free to change the default client profile name.

# Configure OpenVPN 
The following configuration changes need to be made:
1. Do not send all traffic through the VPN
1. Comment out any `dhcp-option DNS` lines
1. Add a `dhcp-options DNS` line to use the Pi-hole for DNS

Edit the following file:
```
vi /etc/openvpn/server/server.conf
```

## Do not send all traffic through the VPN
```
;push "redirect-gateway def1 bypass-dhcp"
```

## Remove all `dhcp-option` lines, for example:
```
;push "dhcp-option DNS 8.8.8.8"
```

## Add a `dhcp-options DNS` line to use the Pi-hole for DNS
```
push "dhcp-option DNS 10.8.0.1"
```

## Restart OpenVPN
```
sudo systemctl restart openvpn-server@server
```
# Save OpenVPN client credentials
The default client credentials are stored in `/root/client.ovpn`. If a custom client named was specified, the filename will be different.

Save a copy of this file to your local machine with `scp` or, alternatively, just copy the contents to a file.

# Install Pi-hole
Defaults can be used for most of the Pi-hole installation except for the following:
* Interface: `tun0` 
* Static IP:
  * IPv4: `10.8.0.1/24`
  * IPv4 gateway: accept default (your instance's gateway)

Run the installation script:
```
curl -sSL https://install.pi-hole.net | bash
```

Once the installation has completed make note of the admin password.

# Connect to VPN from your device
Install the OpenVPN client on Mac or Android. Import the configuration file that was previously saved.

# Optional
## SSL
If you have a domain you'd like to use it's a good idea to set up SSL.  
The process is simple using `letsencrypt`. The [following article](https://www.gilbertotorres.com/install-letsencrypt-ssl-into-pi-hole-server/) is very helpful.

Note, the cronjob in the article is incorrect. Instead, it should be:
```
47 5 * * * root certbot renew --quiet --no-self-upgrade --renew-hook "cat \$RENEWED_LINEAGE/privkey.pem \$RENEWED_LINEAGE/cert.pem > \$RENEWED_LINEAGE/combined.pem;systemctl reload-or-try-restart lighttpd"
```

## Snapshots
Snapshots can be enabled from *Snapshots* tab. A 20gb image will cost approximately $0.05 USD/GB-mo which is $1.50 USD.

