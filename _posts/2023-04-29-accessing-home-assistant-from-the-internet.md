---
layout: post
title: Accessing Home Assistant from the Internet
---

Home Assistant is an open-source home automation platform that allows you to control and automate various aspects of your home, such as lighting, temperature, security, and entertainment systems. It is designed to work with a wide range of devices and platforms, including smart speakers, sensors, cameras, and other Internet of Things (IoT) devices.

In this article I will present how to set up secure remote access for Home Assistant local installation.

<!--more-->

## Plan

Before we begin opening our Home Assistant installation to the world, it is necessary to ensure that our connection is properly secured. Additionally, it would be a significant advantage if we could access it both locally (i.e., on the local LAN or Wi-Fi) and remotely, in an instantly interchangeable way (without taking any additional steps, such as connecting to a VPN).

To achieve the above, I will use Nginx as an authenticated SSL/TLS reverse proxy that will handle all incoming requests, and a technique called DNS spoofing, for convenient switching between local and remote connections.

## Spoof external domain with local DNS

Once the external domain is known, it will need to be spoofed with the local DNS. The requirement here is that we have control of the DNS used in our local LAN or Wi-Fi. The easiest method is to modify `/etc/hosts` on our router and/or machine (laptop or phone).

Assuming that our domain is `home.example.com` and our Home Assistant installation runs on a machine with the IP address `192.168.1.2`, the configuration could look like this:

`/etc/hosts`

    192.168.2.1 home.example.com

On a machine connected to the local LAN, the above configuration can be tested with the following command:

    dig home.example.com

The response should return something like:

    ;; QUESTION SECTION:
    ;home.example.com.          IN      A

    ;; ANSWER SECTION:
    home.example.com.   0       IN      A       192.168.1.2

The next step is to set up an Nginx reverse proxy, which will handle incoming requests that come to `home.example.com`.

## Create nginx reverse proxy

Nginx is a powerful web server and reverse proxy server that can act as an intermediary between client devices and backend servers.

The installation process varies among different Linux distributions, and for the rest of this article, I will assume that we have a running server, preferably on the same machine as Home Assistant.

An example of a default configuration could look like this:

`/etc/nginx/nginx.conf`

    # Defines the number of worker processes.
    worker_processes auto;

    events {}

    http {
      # Include the mime.types file, which contains mappings of file extensions
      # to their corresponding MIME types.
      include mime.types;

      # Defines the default MIME type of a response
      default_type application/octet-stream;

      # Enables or disables the use of sendfile() system call to serve static files.
      sendfile on;

      # Sets the maximum size of the types hash tables.
      types_hash_max_size 4096;

      # Enables or disables emitting nginx version on error pages and in the “Server”
      # response header field.
      server_tokens off;

      # Enable access log
      access_log /var/log/nginx/access.log;

      # Include server specific files
      include /etc/nginx/conf.d/*.conf;
    }

And the one corresponding to home assistant host:

`/etc/nginx/conf.d/home.conf`

    server {
      listen 80;
      server_name home.example.com;

      proxy_buffering off;

      proxy_set_header Connection upgrade;
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr;

      location / {
        proxy_pass http://localhost:8123;
      }
    }

We would also need to change the default Home Assistant configuration (the assumption here is that Home Assistant runs on the same machine as nginx):

`/var/lib/home_assistant/configuration.yaml`

    http:
      use_x_forwarded_for: true
      trusted_proxies:
        - 127.0.0.1

We can then access home assistant installation at `http://home.example.com`.

However, there are a few drawbacks here. The endpoint is served on port 80 and does not offer any encryption. Additionally, our Home Assistant installation may be vulnerable to brute force attacks, in which attackers attempt to guess our username and password.

## Secure connection with SSL/TLS

To secure a connection, SSL/TLS encryption can be used. To achieve this, a self-signed certificate will be employed.

First, a new RSA private key and a Certificate Signing Request (CSR) for an SSL/TLS certificate need to be generated. This can be done with the OpenSSL command-line tool. For simplicity private key won't be encrypted with passphrase (`nodes` flag). Additionally, the key size will be set to 4096 bits with `rsa:4096`:

    openssl req -new -newkey rsa:4096 -nodes -keyout home.key -out home.csr

Then, using the same tool a self-signed X.509 SSL/TLS certificate can be generated.

    openssl x509 -req -sha256 -days 3650 -in home.csr -signkey home.key -out home.crt -extfile openssl.ini

Where `extfile` flag consist SAN (Subject Alternative Name) extension contents. This is mostly required for Chrome browser compatibility.

Contents of `openssl.ini`

    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid:always,issuer:always
    basicConstraints = CA:TRUE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEnciph
    subjectAltName = DNS:example.com, DNS:*.example.com
    issuerAltName = issuer:copy

For additional security we could also generate dhparam file

    openssl dhparam -out dhparam.pem 4096

Once the `home.crt` file is generated, it should be transferred as a trusted authority to all devices that will be accessing the installation. This step is necessary to avoid browser warnings when accessing the home assistant URL. I've also observed that this is required for the mobile app.

## SSL Client certificate authentication

While TLS/SSL encryption is a robust security protocol that safeguards the transmission of data between a client and server by encrypting the data, it primarily ensures the authenticity of the server to the client. However, this isn't enough because the server does not have the same assurance about the client. This is where SSL client authentication comes in. It acts as a two-way handshake, ensuring both parties can trust each other, not just the client trusting the server.

### Create server ssl certificate

First, server ssl certificate for client authorization needs to be created. It can be done with the following command:

    openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes -subj '/CN=example.com' -keyout home_client.key -out home_client.crt

Once it is generated, we can update nginx configuration that reflects SSL changes:

`/etc/nginx/conf.d/home.conf`

    server {
      listen 443 ssl http2;
      server_name home.example.com;

      add_header Strict-Transport-Security "max-age=63072000" always;

      ssl_protocols TLSv1.3;

      ssl_certificate /etc/nginx/ssl/home.crt;
      ssl_certificate_key /etc/nginx/ssl/home.key;

      ssl_client_certificate /etc/nginx/ssl/home_client.crt;
      ssl_verify_client on;

      ssl_session_cache shared:SSL:10m;

      ssl_dhparam /etc/nginx/ssl/dhparam.pem;

      proxy_buffering off;

      proxy_set_header Connection upgrade;
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr;

      location / {
        proxy_pass http://localhost:8123;
      }
    }

Now, when trying to access our home assistant installation a "400 Bad Request" error should be served.

In order to access the server, a client device needs to provide a signed ssl certificate.

### Create client ssl certificate

Each device that will access nginx reverse proxy needs to provide (preferably) own ssl client certificate that was signed with server certificate and key (`home_client.crt` and `home_client.key` generated in previous step). The process consists of 3 steps.

Create client key and certificate signing request (csr):

    openssl req -new -newkey rsa:4096 -nodes -keyout client.key -out client.csr

Generate ssl client certificate using `home_client.key` and `home_client.crt` (valid for 10 years):

    openssl x509 -req -sha256 -days 3650 -in client.csr -CA home_client.crt -CAkey home_client.key -out client.crt

This step is not required, however to ease the transfer of the files to the device, the `client.crt` and `client.key` will be packed into `client.p12` file:

    openssl pkcs12 -legacy -export -inkey client.key -in client.crt -out client.p12

Once `client.p12` is generated it can be transfered to the client device.

## Transfer certificate to client

Transferring the `client.p12` file is a crucial step. This should be accomplished via a local network and/or a physical connection (e.g., USB). For security reasons, copying this file remotely over the internet should be avoided.

The installation process for the p12 file can vary between devices, but for convenience, it could be transferred to a mobile phone (for use with the native home assistant app) and/or a laptop browser (like Chrome).

Once this step is completed, we can access `https://home.example.com`. The browser or app should prompt for certificate confirmation. After validation, we should be able to see the home page.

## Point ISP IP to the external domain

Once the home assistant installation is secured, it's time to consider opening it up for global access.

If your Internet Service Provider (ISP) assigns you a static IP address, you can simply create an A record in your domain's DNS settings. For instance, here's how it can be done:

    home.example.com. 60 IN A 8.8.8.8

In this case, `home.example.com` is the domain that will direct to our home assistant installation, and `8.8.8.8` is our external IP address.

However, in many scenarios, the external IP is dynamic, which means that the `A record` in the DNS settings will need to be updated periodically to accommodate the changing IP. This can be accomplished using a Dynamic DNS service. A host of companies offer this service, including some that provide it free of charge, such as [Duck DNS](https://www.duckdns.org/)).

## Open port on router

By default, incoming SSL traffic arrives via port 443. To intercept this traffic and redirect it to the home assistant installation, certain modifications will be required on your home router.

The setup procedures can differ significantly among various router software. I will demonstrate how this can be accomplished with a router running OpenWRT.

The following entry can be added to the firewall configuration.

`/etc/config/firewall`


    config redirect 'home_assistant'
      option src 'wan'
      option dest 'lan'
      option proto 'tcp'
      option src_dport '443'
      option dest_ip '192.168.1.2'
      option dest_port '443'
      option target 'DNAT'

## Conclusion

Opening my home assistant to remote access allowed me to monitor and control the setup from anywhere. Even with remote access, the setup remains secure thanks to SSL user certificate authentication, while DNS spoofing ensures a smooth transition between Wi-Fi and cellular networks, offering an advantage over VPN-based solutions.
