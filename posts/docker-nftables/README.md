# How to manage Docker's firewall manually with nftables

## Introduction — Docker's mangling

Docker manipulates `iptables` rules.<sup><a name="fn1_source">[1](#fn1_dest)</sup> Some may hate this intrusiveness, while some may be amazed by the apparent magic that it produces. As for me, I hold as principle that I should be the only one in control of the firewall. There is some [evidence](https://blog.newsblur.com/2021/06/28/story-of-a-hacking/) to corroborate this intuition.

A basic band-aid solution is to use the firewall of your cloud hosting provider (like [Hetzner Cloud Firewall](https://docs.hetzner.com/cloud/firewalls/faq/) or [DigitalOcean Cloud Firewall](https://docs.digitalocean.com/products/networking/firewalls/)), but:
1. You shouldn't rely on that alone (in accordance with a [layered defense](https://www.webopedia.com/definitions/layered-defense/) strategy)
2. They have [limitations](https://docs.digitalocean.com/products/networking/firewalls/#limits)
3. Maybe your provider doesn't offer a cloud firewall solution

A second possible strategy, which is the one I will describe in details, is to disable Docker's management of `iptables`, and to write the needed rules yourself. As a side effect, you get to deepen your understanding of both Docker and firewalls.

I will be using `nftables`, as `iptables` is deprecated<sup><a name="fn2_source">[2](#fn2_dest)</sup> and will be showing rules for IPv4 only, as this is my setup. Disclaimer: I am **not** a networking expert, and have figured out the rules mostly by trial and error. Please do contribute if you find any imprecision or error.



## Docker configuration

### Networking prerequisites
You should know about [CIDR notation](https://www.shellhacks.com/cidr-notation-explained-examples/) and that the private subnets you can safely use are:
- 10.0.0.0/8
- 172.16.0.0/12
- 192.168.0.0/24

You should also have a basic knowledge of `nftables` syntax.

### daemon.json
This file is used to configure Docker. You can find it (or create it) in `/etc/docker/daemon.json`. In order for us to define `nftables` rules, we need to assign static (private) IPs to each container, and in order to do that, we need to define our own Docker network.

But first, I like to restrict Docker's access to the private address space, so that I can still run other services, since by default Docker hords ALL of the private (IPv4) address space! <sup><a name="fn3_source_1">[3](#fn3_dest)</sup>

Here is my configuration file, it allows for `256` custom networks of `255` IPs each, with `255` IPs for the default bridge network.

```jsonc
{
  "iptables" : false, // Disables iptables management
  "bip": "10.1.1.1/24", // Default bridge network subnet
                        // Must be a valid IP + subnet mask
                        // See https://github.com/docker/for-mac/issues/218#issuecomment-386489719
  "fixed-cidr": "10.1.1.0/25", // Subnet for static IPs in the default bridge network
  "default-address-pools": [
    {
      "base":"10.2.0.0/16", // Available space for custom Docker networks
      "size":24 // Size of a custom network
    }
  ]
}
```

Feel free to adjust this file to your needs. The official documentation is lacking, but I found some useful info here: [[3]](#fn3_dest).

### Creating your custom network

Here is the command to use:

```
docker network create my-docker-subnet --subnet 10.2.0.0/24 -o com.docker.network.bridge.name=user0
```

It will create a custom bridge network named `my-docker-subnet`, with interface name `user0` (deterministic name, useful for `nftables` rules), with subnet `10.2.0.0/24`.

## Containers configuration

I will use Docker Compose only. I think that a concrete example is best to understand how to do it, and also I don't have a deep enough understanding to explain the theory behind it.

Here is a (partial) example of a mail server with IP `10.2.0.2`, listening on gateway IP `10.2.0.1`, and Nextcloud (with its database and Redis) on IPs `10.2.0.3` to `10.2.0.5`.

I don't map ports for Nextcloud, because requests are proxied through Nginx (running on the host, not as a container) directly to address `10.2.0.5`.

Mail server:

```yaml
services:
  mailserver:
    image: mailserver/docker-mailserver
    ...
    ports:
      - "10.2.0.1:25:25"
      - "10.2.0.1:465:465"
      - "10.2.0.1:587:587"
      - "10.2.0.1:993:993"
    networks:
      docker-subnet:
        ipv4_address: 10.2.0.2

networks:
  docker-subnet:
    external: true
```

Nextcloud:
```yaml
services:
  db:
    image: mariadb
    ...
    networks:
      my-docker-subnet:
        ipv4_address: 10.2.0.3

  redis:
    image: redis
    ...
    networks:
      my-docker-subnet:
        ipv4_address: 10.2.0.4

  app:
    image: nextcloud
    extra_hosts:
      - "mail.test.com:10.2.0.1" # Required to be able to send mail from the nextcloud container
    depends_on:
      - db
      - redis
    networks:
      my-docker-subnet:
        ipv4_address: 10.2.0.5

networks:
  my-docker-subnet:
    external: true
```

## nftables configuration

As above, I just give a (partial) working example with comments inside:

```bash
#!/bin/nft -f

define WAN_IFC = eth0
define DOCKER_IFC = user0
define SERVER_IP = ENTER_PUBLIC_IP_HERE
define MAILSERVER_PRIV_IP = 10.2.0.2

flush ruleset

table main_firewall_table {
  chain prerouting {
    type nat hook prerouting priority dstnat
    policy accept
    ct state invalid drop
    
    # If a packet wants to access "mail-related" ports on server IP, redirect to the Docker container
    ip daddr $SERVER_IP tcp dport { 25, 465, 587, 993 } dnat to $MAILSERVER_PRIV_IP
  }

  chain input {
    type filter hook input priority filter
    policy drop
    ct state invalid drop
    ct state established,related accept
    iif lo accept
    icmp accept

    # Allow outgoing from my custom Docker network
    iifname $DOCKER_IFC accept

    # For Nginx
    ip daddr $SERVER_IP tcp dport 80 accept # HTTP
    ip daddr $SERVER_IP tcp dport 443 accept # HTTPS
  }

  chain forward {
    type filter hook forward priority filter
    policy drop
    ct state vmap { established : accept, related : accept, invalid : drop }

    # Allowed TCP ports to Mailserver
    ip daddr $MAILSERVER_PRIV_IP tcp dport 25 accept # SMTP
    ip daddr $MAILSERVER_PRIV_IP tcp dport 465 accept # ESMTP (implicit TLS)
    ip daddr $MAILSERVER_PRIV_IP tcp dport 587 accept # ESMTP (explicit TLS => STARTTLS)
    ip daddr $MAILSERVER_PRIV_IP tcp dport 993 accept # IMAP4 (implicit TLS)

    # Allow outgoing from my custom Docker network
    iifname $DOCKER_IFC accept
  }

  chain postrouting {
    type nat hook postrouting priority srcnat
    policy accept

    # Modify Docker exiting traffic as coming from server IP
    oifname $WAN_IFC iifname $DOCKER_IFC snat to $SERVER_IP
  }
}
```

### Debugging

If you need to debug what's happening, add this at the end:

```
table ip traceall_table {
  chain prerouting { type filter hook prerouting priority -350; policy accept; meta nftrace set 1; }
  chain output { type filter hook output priority -350; policy accept; meta nftrace set 1; }
}
```

Then run `nft monitor trace`.

## (Bonus) Nginx configuration

For Nextcloud, just use:

```
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://10.2.0.5:80;
}
```

## Footnotes

<a name="fn1_dest">1</a>: [**^**](#fn1_source) https://docs.docker.com/network/iptables/

<a name="fn2_dest">2</a>: [**^**](#fn2_source) _"nftables is the default and recommended firewalling framework in Debian"_ https://wiki.debian.org/nftables

<a name="fn3_dest">3</a>: ^ <sup>[*a*](#fn3_source_1)</sup> <sup>[*b*](#fn3_source_2)</sup> https://straz.to/2021-09-08-docker-address-pools/
