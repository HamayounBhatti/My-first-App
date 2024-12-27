# pktriot
This is the client application for the Packetriot network.  An account with packetriot.com is required before you continue.  This document will describe how to configure and use the pktriot client to setup traffic rules and run your tunnel.

# Docker Notes
Using the pktriot container is the same as the normal client but you'll need to prepend commands with `docker exec -it container-name`.  It's important to use the flags `-it` since some commands require your input.

Below are some examples to explore the differences when using the container.  Continuing reading for more use-caes and examples.

```
# fetch the image
docker pull packetriot/pktriot:latest

# create an unconfigured container to run continously
docker run -id --restart always --name hello-world packetriot/pktriot:latest

# configure
docker exec -it hello-world pktriot configure

# add a new rule
docker exec -it hello-world pktriot route http --add example.com --webroot /data/example.com

# restart the container to realize the configuration and rules
docker restart hello-world
```

These examples skip over some important details, please read our (section)[https://packetriot.com/docs#docker] for more details.

# Configuration
After installing the pktriot client on your system, or locally, you will need to configure the client.  This will identify and authenticate you, and will result in a secret token being exchanged for future authentication and a new tunnel associated to your account with a assigned hostname.
```
pktriot configure

Authenticate this client by visiting this URL:
https://packetriot.com/client/identauth/507a06ba243f1f5435ef9a71052a986880fb43c486a25d4ab224506688132f3a8794d35ab2b3e0161e531f13a1503f601ab15a4bca43d01624fbd75ebf67af132f9063e80781e20717b57647a282332b8b...

Identified and authenticated!
```
When you run this command you will prompted to use two default locations for the result configuration file: your users' home directory (e.g. ~/.pktriot/config.json) or the a system-level directory (/etc/pktriot/config.json).  Use the `--config` flag along to specify another location.  It's important to note that if you do this, you will need to specify it for all future commands.  The pktriot client checks and uses one of the defaults when no custom location is specified.

As you complete the configuration process you will be prompted to select a region for the server you want to connect to.  We suggest you choose one that is geographically closer to you since that will provide the best network performance.

```
Choose the region for the edge server to connect to:
+--------------------+
| #   | Region       |
+--------------------+
| 1   | us-east      |
+--------------------+
| 2   | us-west      |
+--------------------+
| 3   | eu-central   |
+--------------------+

Input selection [#]: 

```

# Tunnel Information
Using the `info` command with the pktriot client will print out the information about the tunnel such as its assigned hostname, the hostname and IP address of the server you connect to, and all of the rules you've setup.

```
pktriot info 

Client:
	Hostname: small-fire-20477.pktriot.net
	Server: us-east-65319.packetriot.net
	IP: 159.203.126.35
```

# Test Tunnel Connectivity
Now that configuration is complete we should test that the client is working correctly.  When no rules exist the tunnel will create a default rule for itself.  This will allow us to test connectivity with the client.  Start the client and visit the hostname assigned to your tunnel.

```
pktriot start

Connecting...

Running HTTP services:
+----------------------------------------------------------------------------------------------+
| Domain                         | Destination   | HTTP   | Secure   | TLS   | Document Root   |
+----------------------------------------------------------------------------------------------+
| small-fire-20477.pktriot.net   |               | 0      | true     | 0     | --              |
+----------------------------------------------------------------------------------------------+

```

Now visit https://small-fire-20477.pktriot.net in your web browser.  It should provide a simple response back.  If this works then you should be able to continue and start adding rules.

# Using Custom Domains
Packetriot will work with custom domains but first you need to verify ownership of the domain.  Log into your account and visit the Domains section of the dashboard.  Add a new domain for verification.  It will present a long random value that you will need to cut and paste into a DNS TXT record using the registrar that you purchased or service you manage your domain with.  

Every few minutes the Packetriot main site will check for this TXT records existence and once found it will mark the domain as owned and verified by you.

From this point on you can use the hostname assigned to your tunnel, and the IP address of the server you connect to, for setting up records for your custom domain.  Some providers allow you setup a CNAME for a root domain (e.g. example.com) but that is rare.  So you will need to use a IP address for A records in most cases.  Subdomains can be setup with a CNAME so its suggest you use the hostname assigned to your tunnel.

# Subdomains for Assigned Hostname
You can setup subdomains with the hostname assigned to your tunnel.  For example, the tunnel in the example above is small-fire-20477.pktriot.net.  You can setup rules using the subdomain blog.small-fire-20477.pktriot.net and they will resolve to the tunnel.  You can use as many subdomains as you'd like with  your assigned hostname.

# Continous Operation
We include a systmed service unit that you can use if you want to run pktriot as a service.  If you installed with an RPM or debian package you can use the commands to enable and start the pktriot client.  Note, you will want to have configured and setup some rules.  Read the notes below for more on that.

```
systemctl enable pktriot
systemctl start  pktriot
```

Also note, if you want to run pktriot as a system service, you may need to create some users and probably do not want to run as root.  You can use typical users like nobody/nogroup, but make sure that they have read access to the path you're using for your configuration: system (/etc/pktriot) or custom path.

# Rules
The following sections will explore a few examples for HTTP and TCP traffic rules.  For these examples we use the domain packetdemo.com as our custom domain that've already verified with Packetriot earlier.

**Important to note** When you make changes to your rules you will need to restart your pktriot client.  It will not dynamically check changes in the configuration file and realize them.

# HTTP Traffic Rules
We will walk through some typical examples of HTTP traffic rules.  We'll examine serving static content, TLS termination and transparent proxying.  We'll also examine how to setup security through Lets Encrypt or custom certificates.

## Serving Static Files
In this example we will be serving out some static files from a directory on our local file-system.  Let's say we're building a website in /var/www/blog.example.com.  We can setup a rule that will service this content and use Lets Encrypt to secure it.  We'll also set a flag to redirect all insecure (HTTP) requests to using HTTPS.

```
pktriot route http add --domain blog.example.com --webroot /var/www/blog.example.com --letsencrypt --redirect
```

## Serving Static Files with Custom Certs
Let's reuse the example above but suppose that we used another service to generate our TLS certificates.  We'll have been given, or created, a fullchain certificate and a private key.  Let's create a traffic rule uses the custom cert/key and serves out our static files for us.  This rule will also redirect to HTTPS.

```
pktriot route http add --domain blog.example.com --webroot /var/www/blog.example.com --ca /path/to/fullchain.pem --key /path/to/private-key.pem --redirect
```
## TLS Termination and Reverse Proxy
In this example we will suppose that we have a locally running server using port 8080.  It can be a new application we're developing, so we'll give it the domain app.packetdemo.com.  We will use the pktriot client to terminate TLS traffic and then proxy it to this HTTP service.  We will use Lets Encrypt for simplicity and enable secure redirection as well.

```
pktriot route http add --domain app.packetdemo.com --destination 127.0.0.1 --http 8080 --letsencrypt --redirect
```

## Tranparent Reverse Proxy
This example will assume that we have a server that's been configured to serve HTTP and HTTPS traffic.  We'll also assume that it's running on default ports for those services but we will specify for completeness since the ports can be configured any way.  The pktriot client will need a rule that requests both port 80 (HTTP) and 443 (HTTP) traffic on the server is connects to, but it will transparent proxy to the destination that will take care of TLS handshaking and performing encryption operations.  

Let's say this is a custom application running on our private network.  We'll give it the domain name custom.packetdemo.com.

```
pktriot route http add --domain custom.packetdemo.com --destination 192.168.1.101 --http 80 --tls 443 
```

# TCP Traffic Rules
Below are sections describing how to create rules for forwarding TCP traffic to services running on your local or private network.

## Port Allocation
The Packetriot service uses the port range 22,000-22,999.  It assumes most traffic will be SSH, since its popular, and the hope is the 22k range will reduce the number of numbers to memorize.  

To forward ports we first need to allocate them.  Every plan includes at least one port per tunnel, so let's allocate one.

```
pktriot route tcp allocate

Allocated port 22197
```

## Port Forwarding
Now that a TCP port has been allocated, you can add or edit the rules as often as you'd like on your end.  Let's say for example that we want to forward traffic on this port on the server we connect to, to a an SSH server running on our private network.  Below is an example for a command that will do that.

```
pktriot route tcp forward --port 22197 --destination 192.168.1.101 --dstport 22
```

We can change the forwarding the destination and port we forward to by just using the `forward` subcommand again with the same port and it will overwrite any existing rules.

```
pktriot route tcp forward --port 22197 --destination 192.168.1.131 --dstport 5000
```

## Port Releasing
A TCP port is released when a tunnel is shutdown or deleted via the Packetriot dashboard, so by the user manually through the client.  You can use the following command to release a port and give it back to the server.  It will be available to another user now.

```
pktriot route tcp relase --port 22197
```

# Firewalls
You can setup the equivalent of firewall rules for all HTTP and TCP traffic rules you setup.  On Packetriot you will need a "Pro" account of higher to use these functions.

Like traffic rules, changes to the firewall for traffic rules will not be realized until we restart your pktriot client.

Let's discuss some of the mechanics with how firewall rules are processed by the Packetriot server.  If you only allow IPs or a network, then all other traffic will be dropped.  If you only drop certain IPs or networks, then all other traffic will be allowed in.

There is no need to use a combination of drop or allow rules (although you could...).

## IP Filtering
Let's say you want to limit incoming traffic to a website you are hosting locally and serving using pktriot to a group of friends.  With their IP addresses we can do that.  We'll assume that this is for that application we're building that we setup the app.packetdemo.com rule for earlier in this document.

```
pktriot route http firewall allow --domain app.packetdomain.com --network 128.98.22.124
pktriot route http firewall allow --domain app.packetdomain.com --network 104.4.21.123
```

The server that your tunnel connect receive these rules and start filtering traffic.  If the incoming traffic is not from either of these IPs the traffic will be dropped and will not be steered to your client.

## Filter by Network
You can also filter by a network as well.  The firewall can use CIDR to denote a network and filter using that as well.  Here is an example of a network and also an example from above for a single IP adddress.

```
pktriot route http firewall allow --domain app.packetdomain.com --network 128.98.22.0/24
pktriot route http firewall allow --domain app.packetdomain.com --network 104.4.21.123/32
```

The /24 is a network and /32 indicates it's for the address leading to the mask.  



