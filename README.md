# **attnative6**
### Scripts to enable native dual-stack IPv6 on AT&amp;T with your own router

If you skip to the [installation](#installation) section, do read through all of this at some point.

---
### [PROBLEM](#problem)&emsp;&emsp;[SOLUTION](#solution)&emsp;&emsp;[REPO CONTENTS](#repo-contents)
### [INSTALLATION](#installation)&emsp;[TROUBLESHOOTING](#troubleshooting)&emsp;[UNINSTALLATION](#uninstallation)
---
## PROBLEM
A bypassed router on AT&T's residential u-Verse network does not pull a DHCPv6 lease from upstream DHCPv6 servers.

This is because most DHCPv6 clients available as of this writing (December 2017) default to sending a DUID-LL or DUID-LLT, and do not support configuring a [DUID-EN](https://tools.ietf.org/html/rfc3315#section-9.3).

A DUID-EN is stored in the Residential Gateway (RG) by the manufacturer, and is sent with every DHCPv6 request from the RG. No DUID-EN, no DHCPv6 assignment.

AT&T has long offered 6rd on its network as an alternative for IPv6 connectivity that, frankly, lacks for nothing in functionality compared to native IPv6 for the vast majority of users. But a thing is possible, so we must do it.

### Investigation
How to figure out the DUID-EN assigned to us? According to RFC 3315, a DUID-EN is comprised of 2 fields - an IANA-assigned enterprise number and an arbitrary, vendor-defined hunk of data.

By examining the DHCPv6 requests sent by the RGs with the help of **`tcpdump`**, the format of the DUID-EN has been decoded. And it turns out that it's quite simple:

<pre>
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   unsigned integer: IANA-assigned Private Enterprise Number   |    Field 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-------------------
|    IEEE-assigned OUI (Organizationally Unique Identifier)     |    Field 2
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  of RG mfg'r (2wire or Arris) | Hyphen ("-")  |               |    Also field 2
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               |
.                                                               .    Still field 2
.                    Serial number of RG                        .
.                     (variable length)                         .    Yup, field 2
.                                                               .
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
</pre>

Everything is ASCII except for the Private Enterprise Number (PEN) at the beginning. In other words, it's a `uint32` followed by something like `001F2E-123456789012`.

One PEN, `3561`, has been discovered so far. It's listed [here](https://www.iana.org/assignments/enterprise-numbers/enterprise-numbers) (warning: ridiculously long text file) as belonging to `The Broadband Forum (formerly 'ADSL Forum')`, so that seems right.

Two OUIs, `00D09E` and `001E46`, have been discovered so far. Those are listed [here](http://standards-oui.ieee.org/oui.txt) (warning: another ridiculously long text file) as belonging to `2Wire Inc` and `ARRIS Group, Inc.`, and as they were sent respectively by a 2Wire Pace 5286AC and an Arris Motorola NVG599, those seem right too.

Hmm, that's enough information to reconstruct each RG's DUID-EN. Go configure your DHCPv6 clients for a DUID-EN now and enjoy. Thanks for reading!

...Oh right, most DHCPv6 clients. No DUID-EN support.

[UP](#attnative6)

## SOLUTION
[dibbler](http://klub.com.pl/dhcpv6/) is a DHCPv6 server, relay, and client, and it *does* support configuring a DUID-EN. We can use the client portion (aptly named **`dibbler-client`**) for our purposes.

**Obvious Q:** "B-b-but I've never heard of dibbler! Why would I want to install some janky Warsaw Bloc DHCPv6 client?"

**A:** It *is* janky! Don't start it up with its default configuration before you edit the configuration file to make it send a DUID-EN, or it won't actually send the DUID-EN! (If only I'd known this months ago, when I first looked into dibbler and gave up on it as broken.)

But it does work. It made beautiful-looking DHCPv6 packets (fine, *datagrams*) and sent them out, and although AT&T's servers proceeded to cheerfully ignore everything contained within them except the parts where dibbler asked for an IPv6 address and a prefix delegation (PD), *the servers replied*.

### Caveats
1. AT&T delegates a /60, and by default **`dibbler-client`** will split that into a /68 for every non-WAN interface it finds on the system, basically because it arbitrarily expects to receive a /56. A partial remedy is to split the delegation yourself, which just means that you divide it up however you like and assign the new subnets to the network interface(s) on your LAN.

2. Things **`dibbler-client`** will do ("its job"):
   * bind the IA_NA ("IPv6 address") to your WAN interface
   * ~~improperly split the IA_PD ("prefix delegation") from AT&T and bind a /68 within it to your LAN interface~~ just kidding, there's a helper script

   Things **`dibbler-client`** won't do ("not its job") that nevertheless are really basic IPv6 things:
   * set up a default IPv6 route, aka an IPv6 default gateway - it's the kernel's job to listen to Router Advertisements on the WAN, ~~but it often doesn't do so by default~~ again, there's a helper script
   * broadcast Router Advertisements to the LAN - on Linux we use **`radvd`** or **`dnsmasq`** for this
   * send LAN clients DNS6 server information - via your Router Advertisements to the LAN in the Recursive DNS Server (RDNSS) option, in DHCPv6 assignments, or both
   * any sort of IPv6 firewalling

   Things my helper script will do:
   * properly split the /60 prefix delegation from AT&T and bind a /64 within it to your LAN interface
   * make the kernel listen to the Router Advertisements that AT&T sends about every 20 minutes, allowing your kernel to learn an IPv6 default gateway
   * if **`rdisc6`** is installed, immediately solicit a Router Advertisement packet (OK, OK, it's a *da-ta-gram*) from AT&T, so you aren't left waiting for that long before you can use IPv6

3. Sorry, BSDudes, **`dibbler-client`** isn't included in BSD or pfSense's package repositories. It builds nicely from the [sources elsewhere on github](https://github.com/tomaszmrugalski/dibbler), though. Just remember to create at least the `/var/lib/dibbler` directory before starting it, because **`make install`** doesn't. Also, write a config file at `/etc/dibbler/client.conf`, which may be/start out as the one you generate with my installer script.

[UP](#attnative6)

## REPO CONTENTS

**`install-attnative6`** is a script to make configuring **`dibbler-client`** and getting native IPv6 easier. It does the following:
1. Prompts for the necessary information to generate a working configuration for **`dibbler-client`** from **`client.conf.template`**

2. Also generates a (Linux-specific, but potentially portable) helper script from **`pdsplit.sh.template`**, which will be called when **`dibbler-client`** gets a DHCPv6 assignment to:
   * assign the first of the 16 /64s contained in a delegated /60 to a single LAN interface (fair warning: it assumes the PD's for a /60 and that there's a /64 in it)
   * force the kernel to listen to Router Advertisements coming in on the WAN
   * if **`rdisc6`** is installed, force AT&T to send a Router Advertisement

3. Checks for running instances of **`dibbler-client`** and kills them

4. Checks for an old **`dibbler-client`** configuration, possibly created by the instance(s) it just killed, and gets rid of them - said old configuration will probably interfere with sending the DUID-EN generated in 1.

5. Prompts to install the configuration generated in 1. and the helper script generated in 2. to appropriate directories

[UP](#attnative6)

## INSTALLATION
Sorry, instructions for Linux only for now.

BSDudes need to compile and install the client and can run the generator script, but the helper script shouldn't be installed without at least converting all the relevant calls to Linux's **`ip`** into calls to BSD's **`ifconfig`**.
### Prerequisites
1. Disable the current DHCPv6 client from running on your WAN interface, if any.

2. Get and install **`dibbler-client`** from your Linux distribution's package repository.

   If the package installer prompts you to ask if you want to start the client during boot, **REFUSE** for now.

   If you didn't refuse, don't worry, the installer script can remedy your error later.

3. (Technically optional, practically essential) Also get and install **`rdisc6`**, which is contained in a package named **`ndisc6`**. This will save you up to 20 minutes before IPv6 works.
### Installation
1. Clone the repository, or download the first release and untar it.

2. **`cd`** to where the release ended up on your system, run **`./install-attnative6`** as root, and answer the questions.

   Note that the generated files will make **`dibbler-client`** start only on the WAN interface you specified, and will only assign the first /64 in the delegated prefix to the LAN interface. If this is not what you want, edit the appropriate files later. Both of them are probably in the same place, after all...
### Usage
1. Once the files were correctly generated and installed, start the client, most likely with **`systemctl start dibbler-client`** as root.

   It should go without saying, but if you were using a 6rd tunnel before, stop and remove the tunnel, stop it from being autocreated, rchange your firewall rules if they referenced the tunnel device name, and make any necessary changes to your DNS6, DHCPv6, and Router Advertisement daemon configurations before you start **`dibbler-client`** for the first time.

2. Read the log files in `/var/log/dibbler` to verify that the client and helper script are working. The contents should be pretty self-explanatory.

3. Once the client is fully working, make it start at boot, most likely with **`systemctl enable dibbler-client`**.

[UP](#attnative6)

## TROUBLESHOOTING
If you're running into problems, edit `/etc/dibbler/client.conf` to change the log level to 8, and also to make any other changes you want there and in the helper script at `/etc/dibbler/pdsplit.sh`.

Restart the client, most likely with **`systemctl restart dibbler-client`** as root.

To start over, uninstall everything as below, reinstall the client, and run **`./install-attnative6`** again.

It's also possible that while you were not bypassed, your Residential Gateway got a DHCPv6 lease. The configuration file that my script generates for dibbler-client makes it behave pretty much identically to my NVG599 when doing DHCPv6 - but if your RG/firmware does it differently, you probably need to wait up to 14 days for the lease to expire on AT&T's end.

(You can get DHCPv6 working sooner if you can craft a DHCPv6 release packet to the same server that's currently holding your lease with pretty much the exact same options as those originally sent by your RG - there is some leeway but I couldn't say just how much - and inject it onto the WAN interface. AT&T will reply if you do this successfully. Unless you happen to have packet captures of the initial request, however, this is... kinda unrealistic. But if you *do* have packet captures, please let me know and I'll update this script to account for your model of RG!)

[UP](#attnative6)

## UNINSTALLATION

To remove everything dibbler-related, run **`apt-get purge dibbler-client`** and **`rm -f /etc/dibbler /var/log/dibbler /var/lib/dibbler`**, both as root.

---
Good luck, and send your comments to kangscinate <æt þͤ gee-maille dawt cawm>!
