---
layout: post
title: "Mobilus Cosmo GTW: Reverse Engineering a Radio Shutter Device"
---

Here, I will present my findings from reverse-engineering a radio shutter device, the Mobilus Cosmo GTW. This device is used to control the opening and closing of roller radio shutters. In this article, I will describe my motivation, how I approached the problem, what tools I used, and how I managed to create a native Python client to control the device.

<!--more-->

<p class="message">
The full code described in this article can be found <a href="https://github.com/zpieslak/mobilus-client">here</a>.
</p>

## Motivation

As a smart home fan and an owner of radio shutters, I was looking for a way to control them remotely. The ideal solution would be to integrate the shutters with my Home Assistant installation so I could control them through the HA web interface or mobile app. Then I could group the shutters or create automations to control them based on the time of day, weather, or other conditions.

Unfortunately, the shutters I have are controlled by a proprietary radio protocol, which is not supported by any of the existing integrations. What's more, the manufacturer does not provide any API or SDK to control the shutters. The only way to control them is by using the remote control that comes with the shutters or through the web interface provided by the Mobilus Cosmo GTW (a device also provided by the manufacturer).

## First Approach

Since there isn't an official API, I had to consider other options. The first thing that came to my mind was to use what the manufacturer provides. The Mobilus Cosmo GTW is a device that can be used to control the shutters through the local web interface that can be accessed from a browser. What's more, the device can be controlled remotely and it can be integrated with Google Home or Amazon Alexa.

Since it exposes the shutters to Google Home, it can be controlled by Home Assistant. This is the first solution I decided to go with. The diagram below shows the communication between the devices.



     +--------------------------+    +-------------------+    +---------------------+
     |  Local                   |    |  Remote           |    |  Local              |
     |        +---------+       |    |                   |    |                     |
     |        | Shutter |       |    |                   |    |                     |
     |        +----+----+       |    |                   |    |                     |
     |             |            |    |                   |    |                     |
     |  +----------+----------+ |    |  +--------------+ |    |  +----------------+ |
     |  |  Mobilus Cosmo GTW  |-|----|->|  Google Home |-|----|->| Home Assistant | |
     |  +---------------------+ |    |  +--------------+ |    |  +----------------+ |
     +--------------------------+    +-------------------+    +---------------------+


As shown in the diagram above, the communication is inefficient. The signal has to go from the Mobilus Cosmo GTW to Google Home, and then to Home Assistant. This is not ideal, as it introduces additional latency and potential points of failure. In fact, the integration was not stable. Sometimes the shutters did not respond to the commands, or the commands were delayed.

## Second Approach

Since the first solution was not stable and there were no alternative solutions available, I decided to reverse-engineer the communication between the Mobilus Cosmo GTW and the shutters. The idea was to create a native Python client that would communicate directly with the Mobilus Cosmo GTW, without the need for Google Home or Home Assistant. Then the client can be used as a base for a custom Home Assistant plugin.

## Reverse-engineering

The first step was to identify the structure of the Mobilus Cosmo GTW web interface. I used the developer tools in the browser to inspect the network traffic. I found that the JavaScript front-end communicates with the back-end through a WebSocket connection. Unfortunately, the WebSocket messages were encrypted, so I could not use them directly.

The next step was to analyze the encrypted messages. How are they encrypted? How can I decrypt them? I needed to find a way to easily debug the JavaScript code. Since the device exposes a web application and has hardcoded JavaScript file paths, I decided to "trick" the back-end and put the file behind a proxy that I control. This way, I was able to intercept the messages and inspect them.

### Putting the javascript file behind the proxy

The first step was to set a static IP address for the device. Since I have an OpenWrt router, it was easy to just tie the MAC address of the device to a predefined IP in the `/etc/ethers` file and let the DHCP server do the rest.

    FF:FF:FF:FF:FF:FF 192.168.1.2

In order to be able to pass all communication through the reverse proxy, I assigned a custom domain name to the device. I added the following line to the `/etc/hosts` file:

    192.168.1.2 mobilus.lan

The next step was to create the reverse proxy itself. I used the `Nginx` server for this purpose. I created a new configuration file `/etc/nginx/sites-available/mobilus-cosmo-gtw` with the following content:

    server {
      listen 80;
      server_name mobilus.lan;

      proxy_set_header Connection upgrade;
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr;

      location / {
        proxy_pass http://192.168.1.2;
      }
    }

    server {
      listen 8884;
      server_name mobilus.lan;

      proxy_set_header Connection upgrade;
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr;

      location / {
        proxy_pass http://192.168.1.2:8884;
      }
    }

Since the device operates on standard HTTP port 80 and uses WebSockets on port 8884, I created two server blocks.

With the above configuration, I was able to control the communication of the device so that each request was passed through the Nginx server. To test it, I visited `http://mobilus.lan` in the browser and checked the Nginx logs with the following command:

    tail -f /var/log/nginx/access.log

The last step was to identify the JavaScript file that was responsible for the communication with the back-end. I found two files, located at `/scripts/app-ac3f8b0743.js` and `/scripts/vendor-e721c51d80.js`. I downloaded both files, put them into a local directory `/media/data/scripts/`, and updated the Nginx configuration file to serve the files from that directory:

    server {
      listen 80;
      server_name mobilus.lan;

      proxy_set_header Connection upgrade;
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr;

      location / {
        proxy_pass http://192.168.1.2;
      }

      location /scripts {
        root /media/data/;
      }
    }

    server {
      listen 8884;
      server_name mobilus.lan;

      proxy_set_header Connection upgrade;
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr;

      location / {
        proxy_pass http://192.168.1.2:8884;
      }
    }

I visited `http://mobilus.lan` in the browser and tested manually to see if it was still working properly, despite the fact that the JavaScript file was served from my local directory.

### Preparation of the JavaScript File

Once the JavaScript file was served from my local directory, I was able to modify it. Since the file was minified, I used the `ESLint` tool to unminify it and make the code more readable. After that, I was able to inspect the code by inserting console logs in the right places and checking the browser console.

I ran the following command to unminify the JavaScript file:

    eslint --fix --config eslint.config.mjs /media/data/scripts/app-ac3f8b0743.js

The `eslint.config.mjs` file contained the following configuration:

    import pluginJs from '@eslint/js'
    import prettierConfig from 'eslint-config-prettier'
    import eslintPluginPrettierRecommended from 'eslint-plugin-prettier/recommended'

    export default [
      pluginJs.configs.recommended,
      prettierConfig,
      eslintPluginPrettierRecommended,
    ]


### Inspecting the JavaScript Code

After investigating the code, trial and error, and a lot of console logs, I was able to find out the following things:

1. The WebSocket protocol was used to communicate with the back-end MQTT server.
2. The messages consisted of two parts: an unencrypted header, which contained message metadata, and an encrypted body.
3. The encryption was done using the AES algorithm with the initialization vector (IV) created from the header and keys depending on the message type.
4. The first request was an authentication message, encrypted with the key being the user password. In response, private and public keys were returned, which were used to encrypt further messages.

With the above information, I was able to create a Python client that was able to communicate with the Mobilus Cosmo GTW. The client was able to authenticate, send commands to the shutters, and receive the responses.

When creating the client, I tried to use minimal external libraries. For the MQTT client, I used the `paho-mqtt` library, which is a standard library for MQTT communication in Python and is also used in Home Assistant. The code can be found in <a href="https://github.com/zpieslak/mobilus-client">this repository</a>.

## Conclusions

The reverse-engineering process was a great learning experience. I was able to learn how the Mobilus Cosmo GTW works, how the communication works, and how to create a native Python client to control the shutters. The client can be used as a base for a custom Home Assistant plugin, which will allow me to control the shutters directly from the Home Assistant web interface or mobile app.
