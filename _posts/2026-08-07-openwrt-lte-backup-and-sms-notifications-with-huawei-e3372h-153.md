---
layout: post
title: "OpenWrt LTE Backup and SMS Notifications with Huawei E3372h-153"
---

In this article, I will present how I set up a backup LTE connection and built a custom SMS notification system using a USB modem and OpenWrt. The goal was to make my network more reliable and add a notification path that is controlled locally and does not depend on cloud push services or a specific mobile app.

<!--more-->

## Problem

I needed a notification system that would allow me to receive alerts about specific events in my home network. For my Home Assistant setup, I have native notifications configured, but they only work for a certain mobile device. I wanted to have a more generic solution that would allow me to send SMS messages to any phone number, specifically to a device that does not have the Home Assistant app installed. Additionally, once the modem was set up, I wanted to use it as a backup internet connection.

Taking the above into account, the requirements were:

* Ability to send custom SMS notifications from my local network (e.g. from Home Assistant, but not limited to it)
* Automatic failover to LTE when the primary WAN connection goes down

## Current setup and initial considerations

My network setup is based on OpenWrt running on a dedicated router. On the same local network, I also have a separate Linux server that runs the Home Assistant instance. That means that any solution had to be compatible with OpenWrt and also provide a simple way for both systems to communicate.

While OpenWrt has good support for USB modems, for my specific use case I needed a modem that:

* Can be controlled at a low level (e.g. via AT commands for SMS sending)
* Supports network modes that could be fully controlled by OpenWrt (e.g. NCM for WAN connection)

Network control should be done directly on the router. I also wanted to have full control over the WAN connection at a low level, so the decision was to connect the modem physically to the router.

The other limitation was that my router has only USB 2.0 ports, so I had to choose a modem that works well with USB 2.0. Fortunately, I had a Huawei E3372h-153 modem available, which could be used for this purpose.

That decision (to use a USB modem connected directly to the router) implied that I needed to consider how to control the SMS part. The considerations were:

* Expose the modem port to the local network and send AT commands from the Home Assistant server. Since the server has more resources and a more advanced system, it provided more options in terms of software and libraries. However, it also introduced more complexity and potential network connection issues between the server and the modem.

* Control the SMS sending directly on the router. This approach is more self-contained and does not depend on the network connection between the server and the modem. However, it also means that I need to set up a way to communicate between the server and the router, so that the server can trigger SMS sending when needed.

In the end, I decided to keep the modem handling on the router. The router is the device that should decide about WAN failover, and it is also the device that has physical access to the modem. The rest of the network should only interact with it through a small interface: request sending an SMS message or consume received SMS messages forwarded by the router.

## Preparing the modem

The key requirement here was compatibility with OpenWrt. After some research, I decided to use Huawei E3372h-153 running in non-HiLink mode. This way I could control the modem at a low level and use it as a standard network interface.

### Switching to stick mode

The device that I had was running in HiLink mode, which is a special mode that makes the modem appear as a network device with a built-in web interface. While this is convenient for casual usage, it is not ideal for my use case. I wanted OpenWrt to manage the connection directly and I wanted stable access to the serial AT command interface.

To overcome this, I flashed the device with firmware that switches it to non-HiLink (stick) mode. To do this, a special flashing tool was needed, as well as putting the modem into a special flashing mode. The process was a bit tricky, but after following the instructions carefully, I was able to successfully flash the modem. I was following the steps found [here](https://zaage.it/tutorials/flashing-huawei-e3372h-4g-lte-from-hilink-to-modem-mode).

## Testing with AT commands

Before I could interact with the modem, I had to switch its USB mode with `usb-modeswitch`. By default, the device was exposed as a virtual CD-ROM with vendor software instead of a modem, so the serial interfaces were not available yet. In OpenWrt this can be handled by installing the `usb-modeswitch` package and reconnecting the modem.

After switching the mode, the modem ports appeared and I could talk to it with AT commands.

The relevant part of the system log looked like this:

```text
Fri Aug  7 20:36:50 2026 kern.info kernel: [   18.228191] usbcore: registered new interface driver option
Fri Aug  7 20:36:50 2026 kern.info kernel: [   18.233911] usbserial: USB Serial support registered for GSM modem (1-port)
Fri Aug  7 20:36:50 2026 kern.info kernel: [   18.241041] option 1-1:1.0: GSM modem (1-port) converter detected
Fri Aug  7 20:36:50 2026 kern.info kernel: [   18.247543] usb 1-1: GSM modem (1-port) converter now attached to ttyUSB0
Fri Aug  7 20:36:50 2026 kern.info kernel: [   18.254615] option 1-1:1.1: GSM modem (1-port) converter detected
Fri Aug  7 20:36:50 2026 kern.info kernel: [   18.260980] usb 1-1: GSM modem (1-port) converter now attached to ttyUSB1
```

To open a serial connection to the modem, I used `picocom`, which is a simple terminal program for serial communication.

```bash
picocom -b 115200 /dev/ttyUSB0
```

This command opens a serial connection to the modem. I was able to send AT commands and receive responses.

The first command was just a simple sanity check:

```bash
AT
```

The expected response is:

```bash
OK
```

Then I checked the available port modes:

```bash
AT^SETPORT=?
```

Before the change, my modem was configured as:

```text
^SETPORT:A1,A2;12,1,16,A1,A2
```

For Huawei modems, the important values are usually:

* `12` – PCUI (AT command) interface.
* `16` – NCM network interface.
* `1` or `10` – modem interface, depending on the firmware.
* `A1` / `A2` – virtual CD-ROM / SD card interfaces, which are not needed in my setup.

I needed to put the USB device into a mode that exposes the NCM interface and the serial port for AT commands. This can be done with the following command:

```bash
AT^SETPORT="FF;12,16"
```

The important part is to keep the PCUI interface enabled. Without it, the modem may stop exposing the AT command port and recovering it becomes much harder.

To verify that the change was successful, I sent the following command:

```bash
AT^SETPORT?
```

The modem may return the following response:

```bash
^SETPORT:FF;12,16
```

After that, I unplugged and plugged the modem again, so the operating system could enumerate the new interfaces.

In OpenWrt, I will be using the NCM interface for network connectivity, and the PCUI interface for sending AT commands (e.g. for SMS).

To verify that the modem is working correctly, I sent the following command:

```bash
AT+CEREG?
```

The modem may return the following response:

```bash
+CEREG: 0,1
```

This means that the modem is registered to the LTE network.

Taking all these steps into account, I was able to successfully prepare the modem for integration with OpenWrt.

## Stable serial device names

After I configured the modem, I noticed that the serial device names can change on each reboot. For example, one time the modem appeared as `/dev/ttyUSB0` and the next time as `/dev/ttyUSB1`. This can be problematic, since I need to know which device name to put in the configuration file.

After some research, I found that the best way to solve this is to create a stable symlink for the modem. On a regular Linux distribution I would usually use a `udev` rule, but OpenWrt does not use `udev`. Instead, it uses `hotplug` to handle device events. Taking this into account, I created a script in `/etc/hotplug.d/tty/` that creates known symlinks for the modem based on its USB vendor and product IDs.

`/etc/hotplug.d/tty/20-usb-serial`

```bash
#!/bin/sh

[ "${ACTION}" = "bind" -o "${ACTION}" = "unbind" ] || exit 0
[ "${SUBSYSTEM}" = "usb-serial" ] || exit 0
[ -n "${DEVICENAME}" -a -n "${DEVPATH}" ] || exit 1

if [ "${ACTION}" = "bind" ]; then
  subsystem="$(basename $(readlink /sys${DEVPATH}/../subsystem))"

  [ "$subsystem" = "usb" ] || exit 0

  replace_whitespace="s/^[ \t]*|[ \t]*$//g; s/[ \t]+/_/g"
  manufacturer="$(cat /sys${DEVPATH}/../../manufacturer | sed -E "${replace_whitespace}")" || manufacturer="$(cat /sys${DEVPATH}/../../idVendor)"
  product="$(cat /sys${DEVPATH}/../../product | sed -E "${replace_whitespace}")" || product="$(cat /sys${DEVPATH}/../../idProduct)"
  serial="$(cat /sys${DEVPATH}/../../serial | sed -E "${replace_whitespace}")"
  interface="$(cat /sys${DEVPATH}/../bInterfaceNumber)"
  port="$(cat /sys${DEVPATH}/port_number)"

  replace_chars="s/[^0-9A-Za-z#+.:=@-]/_/g"
  id_link=$(echo "${subsystem}"-"${manufacturer}"_"${product}${serial:+_}${serial}"-if"${interface}${port:+-port}${port}" | sed "${replace_chars}")
  path_link=$(echo "${DEVPATH}${port:+-port}${port}" | sed "s%/devices/%%; s%/${DEVICENAME}%%g; ${replace_chars}")

  mkdir -p /dev/serial/by-id /dev/serial/by-path
  ln -sf "/dev/${DEVICENAME}" "/dev/serial/by-id/${id_link}"
  ln -sf "/dev/${DEVICENAME}" "/dev/serial/by-path/${path_link}"
elif [ "${ACTION}" = "unbind" ]; then
  [ -d /dev/serial ] || exit 0
  for link in $(find /dev/serial -type l); do
    [ -L ${link} -a "$(readlink ${link})" = "/dev/$DEVICENAME" ] && rm ${link}
  done
fi
```

The script hooks into OpenWrt's hotplug system and creates symlinks in `/dev/serial/by-id` and `/dev/serial/by-path` for the modem. This ensures that the modem can always be accessed via a stable path, regardless of how the kernel enumerates USB devices.

The output can be checked with:

```bash
ls -l /dev/serial/by-id/
```

In my setup it looked like this:

```text
lrwxrwxrwx    1 root     root            12 Aug  7 20:37 usb-HUAWEI_MOBILE_HUAWEI_MOBILE-if00-port0 -> /dev/ttyUSB0
lrwxrwxrwx    1 root     root            12 Aug  7 20:37 usb-HUAWEI_MOBILE_HUAWEI_MOBILE-if01-port0 -> /dev/ttyUSB1
```

From this output, the important part for SMS handling is the stable path to the AT command port. In my setup, `if00-port0` is the PCUI/AT port, so I use `/dev/serial/by-id/usb-HUAWEI_MOBILE_HUAWEI_MOBILE-if00-port0` later in the `smstools3` configuration.

The LTE data connection is configured separately in the next section using NCM.

## OpenWrt configuration (NCM)

To use the modem as a network interface, I used it in NCM mode. This allows the modem to appear as a standard network interface in OpenWrt, which can be used for routing traffic.

First, I installed the needed packages:

```bash
opkg update
opkg install kmod-usb-net-cdc-ncm kmod-usb-net-huawei-cdc-ncm comgt-ncm kmod-usb-serial-option luci-proto-ncm
```

Because the modem mode was already set with `AT^SETPORT="FF;12,16"`, I did not need `usb-modeswitch`. After unplugging and plugging the modem again, it appeared directly with the NCM and serial interfaces exposed.

Since NCM is a special type of network interface, the modem also exposes a control interface at `/dev/cdc-wdm0`. This is used by the `comgt-ncm` package to manage the connection.

My OpenWrt `/etc/config/network` configuration looks like this:

```text
config interface 'wan2'
        option apn 'internet'
        option device '/dev/cdc-wdm0'
        option dns '8.8.8.8 8.8.4.4'
        option proto 'ncm'
        option metric '20'
        option mode 'lte'
        option peerdns '0'
```

I also modified the interface to the WAN firewall zone and added wan2 to the same zone. My `/etc/config/firewall` configuration looks like this:

```text
config zone
        option name 'wan'
        list network 'wan'
        list network 'wan2'
        option input 'REJECT'
        option output 'ACCEPT'
        option forward 'REJECT'
        option masq '1'
        option mtu_fix '1'
```


My main internet connection has a lower metric, so it will be preferred over the LTE connection. The LTE connection will only be used when OpenWrt removes the primary default route, for example when the main WAN interface goes down.

However, there is one important limitation of this setup. It does not cover all cases where the WAN interface is still up, but the upstream provider is broken somewhere further away. Existing connections also should not be expected to migrate transparently between interfaces. For a more advanced setup, `mwan3` would be a better solution, because it can actively check connectivity and make failover decisions based on real reachability instead of only interface state. Since I did not want to add more complexity, I decided to keep it simple and use metric-based failover.

## SMS notifications

Once the modem was working correctly, I could use it to send and receive SMS messages. For this purpose, I used `smstools3`, which is a simple and lightweight SMS gateway that can be used on OpenWrt.

To install it, I used the following command:

```bash
opkg install smstools3
```

I used the serial symlink that answers AT commands and does not conflict with the NCM connection handling:

* `/dev/serial/by-id/usb-HUAWEI_MOBILE_HUAWEI_MOBILE-if00-port0`

My `/etc/smsd.conf` configuration looks like this:

```text
#
# Description: Main configuration file for the smsd
#

autosplit = 3
decode_unicode_text = yes
incoming_utf8 = yes
checked = /var/spool/sms/checked
devices = GSM1
eventhandler = /usr/bin/smsd-event-handler.sh
failed = /var/spool/sms/failed
incoming = /var/spool/sms/incoming
logfile = 1
loglevel = 5
outgoing = /var/spool/sms/outgoing
receive_before_send = no
sent = /var/spool/sms/sent
whitelist = /etc/smsd/whitelist

[GSM1]
baudrate = 115200
device = /dev/serial/by-id/usb-HUAWEI_MOBILE_HUAWEI_MOBILE-if00-port0
incoming = yes
init = AT+CPMS="ME","ME","ME"
memory_start = 0
```

The `logfile = 1` setting makes `smsd` log to stdout, which works well with the OpenWrt init script and `logread`. The spool directories are stored under `/var/spool/sms`. On OpenWrt this is usually a tmpfs location, which is fine for queued SMS messages in my setup. This is ephemeral storage, so if the router is restarted, any queued messages will be lost. However, this is acceptable for my use case.

The `decode_unicode_text` and `incoming_utf8` settings are used for received messages. Outgoing messages with non-GSM characters should be tested separately. In my case, the notification messages are simple text messages, but if I wanted to send Polish characters or other Unicode content, I would verify the exact encoding behavior before relying on it.

The configuration also provides a hook script for different events. In my case, I used it to forward received messages to MQTT.

To ensure that SMS messages can be sent only to approved destination numbers, I created a whitelist file. This is important because later SMS sending will be triggered through MQTT. Even if MQTT is available only in the local network, it is still better to have one more simple safety layer. This whitelist limits outgoing recipients; it does not replace MQTT authentication, topic ACLs, or validation of incoming SMS senders.

To do this, I created the `/etc/smsd/whitelist` file with the following content:

```bash
mkdir -p /etc/smsd
cat > /etc/smsd/whitelist <<EOF
48XXXXXXXXX
EOF
```

Then I enabled and started the `smstools3` service:

```bash
/etc/init.d/smstools3 enable
/etc/init.d/smstools3 restart
```

To test SMS sending manually, I created a file in the outgoing spool directory:

```bash
cat > /var/spool/sms/outgoing/test.sms <<EOF
To: 48XXXXXXXXX

Test message from OpenWrt
EOF
```

After a moment, the file should be moved to `sent`. To debug the sending process, I checked the logs with:

```bash
logread -f
```

The relevant log lines looked like this:

```text
Sat Aug  8 14:25:46 2026 daemon.info smsd[2805]: 2026-08-08 14:25:46,5, smsd: SMS To: 48XXXXXXXXX. Moved file /var/spool/sms/outgoing/test.sms to /var/spool/sms/checked
Sat Aug  8 14:25:48 2026 daemon.info smsd[2805]: 2026-08-08 14:25:48,5, GSM1: SMS sent, Message_id: 255, To: 48XXXXXXXXX, sending time 2 sec.
```

## Sending SMS messages from MQTT

Since Home Assistant runs on a different machine, I needed a way to trigger SMS messages through the router. I could have exposed a REST API on the router, but that would require additional software and configuration. Instead, I decided to use MQTT, which is already used in my Home Assistant setup. This way, any system that can publish messages to MQTT can trigger SMS sending.

The idea is simple:

* Any system publishes a message to MQTT
* OpenWrt subscribes to a topic
* The message is forwarded as SMS

The communication between the components looks like this:

```text
Home Assistant / other systems
          |
          v
        MQTT
          |
          v
     mqtt2sms.sh
          |
          v
      smstools3
          |
          v
 Huawei E3372h-153
```

The script is responsible only for one thing: receive JSON from MQTT and create an outgoing `smstools3` message file.

MQTT should not be exposed publicly for this use case. In my setup it is available only in the local network, and SMS sending is additionally limited by the `smstools3` recipient whitelist. The whitelist is only a last-resort constraint on destination numbers, not an authentication mechanism. If this setup was used in a larger network, I would also add broker authentication, topic ACLs, and probably some simple rate limiting.

`/usr/bin/mqtt2sms.sh`

```bash
#!/bin/sh
set -eu

log() { logger -t mqtt2sms "$*"; }

cleanup() {
 log "Shutting down"
 exit 0
}

trap cleanup INT TERM

[ -d "$OUTDIR" ] || {
 log "Output directory $OUTDIR does not exist"
 exit 1
}

log "Starting: broker=$BROKER topic_send=$TOPIC_SEND outdir=$OUTDIR"

mosquitto_sub -h "$BROKER" -t "$TOPIC_SEND" -F '%p' |
while IFS= read -r payload; do
 [ -n "$payload" ] || continue

 to="$(printf '%s' "$payload" | jq -er '.to' 2>/dev/null)" || continue
 message="$(printf '%s' "$payload" | jq -er '.message' 2>/dev/null)" || continue

 tmp_file="$(mktemp "/tmp/.mqtt2sms.XXXXXX")"
 printf 'To: %s\n\n%s\n' "$to" "$message" > "$tmp_file"
 mv -f "$tmp_file" "$OUTDIR/$(date +%s)_mqtt_$$.sms"

 log "Queued SMS to $to"
done
```

Then I made it executable:

```bash
chmod +x /usr/bin/mqtt2sms.sh
```

The custom configuration is stored in UCI:

`/etc/config/mqtt2sms`

```text
config mqtt2sms 'main'
        option broker '192.168.2.1'
        option topic_send 'sms/send'
        option topic_received 'sms/received'
        option outdir '/var/spool/sms/outgoing'
```

And the init script starts the subscriber under `procd`:

`/etc/init.d/mqtt2sms`

```bash
#!/bin/sh /etc/rc.common

START=95
STOP=10
USE_PROCD=1

PROG=/usr/bin/mqtt2sms.sh

start_service() {
  config_load mqtt2sms

  local broker topic_send outdir

  config_get broker main broker
  config_get topic_send main topic_send
  config_get outdir main outdir

  procd_open_instance
  procd_set_param command "$PROG"
  procd_set_param env BROKER="$broker" TOPIC_SEND="$topic_send" OUTDIR="$outdir"
  procd_set_param respawn 3600 5 5
  procd_set_param stdout 1
  procd_set_param stderr 1
  procd_close_instance
}

service_triggers() {
  procd_add_reload_trigger "mqtt2sms"
}
```

Then I enabled and started it:

```bash
/etc/init.d/mqtt2sms enable
/etc/init.d/mqtt2sms start
```

The integration can be tested from any machine that has access to the MQTT broker:

```bash
mosquitto_pub -h 192.168.2.1 -t sms/send -m '{"to":"48XXXXXXXXX","message":"Test message from MQTT"}'
```

After publishing the message, the router should create a file in `/var/spool/sms/outgoing`, and `smstools3` should send it.

From the software side, this became a small event-driven integration. I like this setup because each part has a clear responsibility. MQTT is only a transport layer, `mqtt2sms.sh` only validates and converts messages to the `smstools3` format, and `smstools3` is responsible for talking to the modem. This makes the solution easy to debug and easy to extend later.

## Home Assistant integration

Home Assistant can easily publish messages to MQTT using built-in automation.

My example automation:

```yaml
alias: State Notification
description: ""
triggers:
  - entity_id:
      - input_boolean.custom_state
    from: "on"
    to: "off"
    trigger: state
conditions: []
actions:
  - action: mqtt.publish
    data:
      evaluate_payload: false
      qos: 0
      topic: sms/send
      payload: "{\"to\": \"48XXXXXXXXX\",\"message\": \"Sample message\"}"
mode: single
```

This allows Home Assistant to send SMS notifications without any direct integration with the modem. It also means that the same mechanism can be reused by other systems, as long as they can publish a message to MQTT.

## Receiving SMS messages

Receiving SMS messages is not required for notifications, but once the modem is already connected to the router it is a useful addition. For example, it can be used to forward incoming SMS messages to Home Assistant or to keep a simple audit trail of messages received by the SIM card.

`smstools3` calls the `eventhandler` script for different events. In the configuration above, the event handler path is:

```text
eventhandler = /usr/bin/smsd-event-handler.sh
```

For forwarding received SMS messages to MQTT, I used the following script:

`/usr/bin/smsd-event-handler.sh`

```bash
#!/bin/sh

. /lib/functions.sh

EVENT="$1"
FILE="$2"

publish_received() {
  local broker topic_received from sent content payload

  config_load mqtt2sms
  config_get broker main broker
  config_get topic_received main topic_received

  logger -t smsd-event-handler "Incoming SMS: $FILE"
  from="$(sed -n 's/^From:[[:space:]]*//p' "$FILE" | head -n 1)"
  sent="$(sed -n 's/^Sent:[[:space:]]*//p' "$FILE" | head -n 1)"
  content="$(sed -n '/^$/,$p' "$FILE" | tail -n +2)"
  payload="$(jq -cn --arg from "$from" --arg sent "$sent" --arg message "$content" \
    '{from: $from, sent: $sent, message: $message}')"

  if ! mosquitto_pub -h "$broker" -t "$topic_received" -m "$payload"; then
    logger -t smsd-event-handler "Failed to publish received SMS to $topic_received"
  fi
}

case "$EVENT" in
  RECEIVED)
    publish_received
    ;;
esac
```

This script also uses `jq` to safely generate the JSON payload.

The message published to MQTT has the following structure:

```json
{
  "from": "48XXXXXXXXX",
  "sent": "26-08-07 12:00:00",
  "message": "Example incoming message"
}
```

After that, Home Assistant can subscribe to the `sms/received` topic and use incoming messages in automations. For example, I can forward each received SMS to my phone:

{% raw %}
```yaml
alias: Modem Notification
triggers:
  - trigger: mqtt
    topic: sms/received
actions:
  - action: notify.mobile_app_phone
    data:
      title: "SMS from {{ trigger.payload_json.from }}"
      message: >-
        {{ trigger.payload_json.message }}
        Sent: {{ trigger.payload_json.sent }}
mode: single
```
{% endraw %}

One use case for receiving SMS messages may be to use them as a command interface. For example, I can send a specific SMS to the modem and trigger an action on the router, or pass it to MQTT and then to Home Assistant. This can be useful for remote control or for triggering specific automations when I am not at home. Also, the SMS layer is independent from the internet connection, so it can be used even when the main WAN connection is down. If I use it this way, I will need to add sender validation and a very small command set, because incoming SMS messages should not be treated as trusted input by default.

## Conclusion

Setting up the Huawei E3372h-153 with OpenWrt allowed me to solve two problems with one device. The modem provides a backup LTE connection, and at the same time it can be used as a local SMS gateway for Home Assistant and other systems in my network.

The process involved switching the modem to stick mode, exposing the right interfaces, configuring NCM in OpenWrt, and adding a small MQTT layer on top of `smstools3`. This keeps the setup simple and avoids any direct dependency between Home Assistant and the modem itself.

The final solution is lightweight and does not depend on any external notification service. It can also be extended later, for example by adding more notification scenarios or by using received SMS messages in Home Assistant automations.
