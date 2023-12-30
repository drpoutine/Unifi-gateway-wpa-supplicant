# UXG-Lite wpa_supplicant bypass ATT fiber modem
Use this guide to setup wpa_supplicant with your UXG-Lite to bypass the ATT modem.

Prerequisites:
- extracted and decoded certificates from an ATT modem

Instructions to [extract certs for newish BGW210](https://github.com/mozzarellathicc/attcerts)

## Table of Contents
- [Install wpa_supplicant](#install-wpa_supplicant-on-the-uxg-lite)
- [Copy certs and config](#copy-certs-and-config-to-uxg-lite)
- [Spoof MAC Address](#spoof-mac-address)
- [Setup network](#setup-network)
- [Test wpa_supplicant](#test-wpa_supplicant)
- [Setup wpa_supplicant service for startup](#setup-wpa_supplicant-service-for-startup)
- [Survive firmware updates](#survive-firmware-updates)

## Install wpa_supplicant on the UXG-Lite
SSH into your UXG-Lite.

> Unlike all my other Unifi devices, my SSH private key didn't work with my username, but worked with the `root` user instead. Or user + password defined in `Settings` -> `System` -> `Advanced` -> `Device Authentication`.

The UXG-Lite runs a custom Debian-based distro, so we can install the `wpasupplicant` package.
```
> apt update -y
> apt install -y wpasupplicant
```

Go ahead and create a `certs` folder in the `/etc/wpa_supplicant` folder.
```
> mkdir -p /etc/wpa_supplicant/certs
```

We'll copy files into here in the next step.

## Copy certs and config to UXG-Lite
Back on your computer, prepare your files to copy into the UXG-Lite.

These files come from the mfg_dat_decode tool:
- CA_XXXXXX-XXXXXXXXXXXXXX.pem
- Client_XXXXXX-XXXXXXXXXXXXXX.pem
- PrivateKey_PKCS1_XXXXXX-XXXXXXXXXXXXXX.pem
- wpa_supplicant.conf

```
> scp *.pem <uxg-lite>:/etc/wpa_supplicant/certs
> scp wpa_supplicant.conf <uxg-lite>:/etc/wpa_supplicant
```

Make sure in the `wpa_supplicant.conf` to modify the `ca_cert`, `client_cert` and `private_key` to use absolute paths. In this case, prepend `/etc/wpa_supplicant/certs/` to the filename strings. It should look like the following...
```
...
network={
        ca_cert="/etc/wpa_supplicant/certs/CA_XXXXXX-XXXXXXXXXXXXXX.pem"
        client_cert="/etc/wpa_supplicant/certs/Client_XXXXXX-XXXXXXXXXXXXXX.pem"
        ...
        private_key="/etc/wpa_supplicant/certs/PrivateKey_PKCS1_XXXXXX-XXXXXXXXXXXXXX.pem"
}
```

## Spoof MAC address
We'll need to spoof the MAC address on the WAN port (interface `eth1` on the UXG-Lite) to successfully authenticate with ATT with our certificates.

I know there's an option in the Unifi dashboard to spoof MAC address on the Internet (WAN) network, but this didn't seem to work when I tested it. (If anyone does test this successfully without needing the following, please let me know).

Instead, I had to manually set it up, based on these [instructions to spoof mac address](https://www.xmodulo.com/spoof-mac-address-network-interface-linux.html).

SSH back into your UXG-Lite, and create the following file.

`vi /etc/network/if-up.d/changemac`

```
#!/bin/sh

if [ "$IFACE" = eth1 ]; then
  ip link set dev "$IFACE" address XX:XX:XX:XX:XX:XX
fi
```
Replace the mac address with your gateway's address, found in the `wpa_supplicant.conf` file.

Set the permissions:
```
> sudo chmod 755 /etc/network/if-up.d/changemac
```
This file will spoof your WAN mac address when `eth1` starts up. Go ahead and run the same command now so you don't have to reboot your UXG-Lite.
```
> ip link set dev "$IFACE" address XX:XX:XX:XX:XX:XX
```

## Setup network

### Set VLAN ID on WAN connection
ATT authenticates using VLAN ID 0, so we have to tag our WAN port with that.

In your Unifi console/dashboard, under `Settings` -> `Internet` -> `Primary (WAN1)` (or your WAN name if you renamed it), Enable `VLAN ID` and set it to `0`.

Before applying, note that this change will prevent you from accessing the internet until after running `wpa_supplicant` in the next step. If you need to restore internet access before finishing this setup guide, you can always disable `VLAN ID`.

![Alt text](vlan0.png)

Apply the change, then unplug the ethernet cable from the ONT port on your ATT Gateway, and plug it into the WAN port on your UXG-Lite.

## Test wpa_supplicant
While SSHed into the UXG-Lite, run this to test the authentication.
```
> wpa_supplicant -i eth1 -D wired -c /etc/wpa_supplicant/wpa_supplicant.conf
```
Breaking down this command...
- `-i eth1` Specifies `eth1` (UXG-Lite WAN port) as the interface
- `-D wired` Specify driver type of `eth1`
- `-c <path-to>/wpa_supplicant.conf` The config file

You should see the message `Successfully initialized wpa_supplicant` if the command and config are configured correctly.

Following that will be some logs from authenticating. If it looks something like this, then it was successful!
```
eth1: CTRL-EVENT-EAP-PEER-CERT depth=0 subject='/C=US/ST=Michigan/L=Southfield/O=ATT Services Inc/OU=OCATS/CN=aut03lsanca.lsanca.sbcglobal.net' hash=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
eth1: CTRL-EVENT-EAP-PEER-ALT depth=0 DNS:aut03lsanca.lsanca.sbcglobal.net
eth1: CTRL-EVENT-EAP-SUCCESS EAP authentication completed successfully
eth1: CTRL-EVENT-CONNECTED - Connection to XX:XX:XX:XX:XX:XX completed [id=0 id_str=]
```
> If you don't see the `EAP authentication completed successfully` message, try checking to make sure the MAC address was spoofed successfully.

`Ctrl-c` to exit. If you would like to run it in the background for temporary internet access, add a `-B` parameter to the command. Running this command is still a manual process to authenticate, and it will only last until the next reboot.

## Setup wpa_supplicant service for startup
Now we have to make sure wpa_supplicant starts automatically when the UXG-Lite reboots.

Let's use wpa_supplicant's built in interface-specific service to enable it on startup. More information [here](https://wiki.archlinux.org/title/Wpa_supplicant#At_boot_.28systemd.29).

Because we need to specify the `wired` driver and `eth1` interface, the corresponding service will be `wpa_supplicant-wired@eth1.service`. This service is tied to a specific .conf file, so we will have to rename our config file.

Back in `/etc/wpa_supplicant`, rename `wpa_supplicant.conf` to `wpa_supplicant-wired-eth1.conf`.
```
> cd /etc/wpa_supplicant
> mv wpa_supplicant.conf wpa_supplicant-wired-eth1.conf
```

Then start the service and check the status.
```
> systemctl start wpa_supplicant-wired@eth1

> systemctl status wpa_supplicant-wired@eth1
```
If the service successfully started and is active, you should see similar logs as when we tested with the `wpa_supplicant` command.

Now we can go ahead and enable the service.
```
> systemctl enable wpa_supplicant-wired@eth1
```

Try restarting your UXG-Lite if you wish, and it should automatically authenticate!

## Survive firmware updates
> NOTE: This has been tested with reboots and have confirmed working with no internet connection. Just haven't confirmed if this service itself will survive an actual firmware update.

So it seems firmware updates will nuke the packages installed through `apt`, removing our wpa_supplicant service. Let's cache some files and a system service to automatically reinstall wpa_supplicant and enable and start the service again on bootup.

Let's first download the required packages (with dependencies) from debian into a persisted folder.
> TODO: confirm these files stays after a firmware update

```
> cd /persistent/dpkg/bullseye/packages
> wget http://ftp.us.debian.org/debian/pool/main/w/wpa/wpasupplicant_2.9.0-21_arm64.deb
> wget http://ftp.us.debian.org/debian/pool/main/p/pcsc-lite/libpcsclite1_1.9.1-1_arm64.deb
```

Now let's create a service file to install these packages and enable/start wpa_supplicant:

```
> vi /etc/systemd/system/setup_wpa_supplicant.service
```

Paste this as the contents:
```
[Unit]
Description=Reinstall wpa_supplicant and enable/start it
ConditionPathExists=!/sbin/wpa_supplicant

[Service]
Type=simple
ExecStartPre=dpkg -Ri /persistent/dpkg/bullseye/packages
ExecStart=systemctl enable wpa_supplicant-wired@eth1
ExecStartPost=systemctl start wpa_supplicant-wired@eth1

[Install]
WantedBy=multi-user.target
```
Now enable the service.
```
> systemctl daemon-reload
> systemctl enable setup_wpa_supplicant.service
```
This service should run on startup, and do it's thing to install and startup wpa_supplicant.

### (Optional) If you want to test this...
```
> systemctl stop wpa_supplicant-wired@eth1
> systemctl disable wpa_supplicant-wired@eth1
> apt remove wpasupplicant -y
```

Now try restarting your UXG-Lite. Upon boot up, SSH back in, and check `systemctl status wpa_supplicant-wired@eth1`.
- Alternatively, without a restart, run `systemctl start reinstall.service`, wait until it finishes, then `systemctl status wpa_supplicant-wired@eth1`.)

You should see the following:
```
Loaded: loaded (/lib/systemd/system/wpa_supplicant-wired@.service; enabled; vendor preset: enabled)
Active: active (running) ...
...
Dec 29 23:20:00 UXG-Lite wpa_supplicant[6845]: eth1: CTRL-EVENT-EAP-SUCCESS EAP authentication completed successfully
```

## Troubleshooting

Some problems I ran into...

<a id="fopen"></a>
<details>
  <summary><b>OpenSSL: tls_connection_ca_cert</b></summary>
    > OpenSSL: tls_connection_ca_cert - Failed to load root certificates error:02001002:system library:fopen:No such file or directory

Make sure in the wpa_supplicant config file to set the absolute path for each certificate, mentioned [here](#copy-certs-and-config-to-uxg-lite).
</details>