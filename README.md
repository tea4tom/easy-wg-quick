# wg-cerberus [alias of: easy-wg-quick]
easy-wg-quick - Creates Wireguard configuration for hub and peers with ease

 * [Getting Started](#getting-started)
   * [Prerequisites](#prerequisites)
   * [Installing](#installing)
 * [Usage](#usage)
   * [Sample output](#sample-output)
   * [Using generated configuration](#using-generated-configuration)
 * [Fine tuning](#fine-tuning)
   * [Disabling external interface autodetection](#disabling-external-interface-autodetection)
   * [Disabling external IP address autodetection](#disabling-external-ip-address-autodetection)
   * [Disabling random port assignment](#disabling-random-port-assignment)
   * [Disabling randomly generated internal network addresses](#disabling-randomly-generated-internal-network-addresses)
   * [Setting custom DNS](#setting-custom-dns)
   * [Setting custom client's `AllowedIPs`](#setting-custom-clients-allowedips)
   * [Choosing firewall type](#choosing-firewall-type)
   * [Choosing if PostUp/PostDown should enable/disable IP forwarding](#choosing-if-postuppostdown-should-enabledisable-ip-forwarding)
   * [Enabling IPv6](#enabling-ipv6)
   * [Enabling NDP proxy (instead of default IPv6 masquerading)](#enabling-ndp-proxy-instead-of-default-ipv6-masquerading)
   * [Redirecting DNS](#redirecting-dns)
   * [Persisting configuration with systemd](#persisting-configuration-with-systemd)
 * [License](#license)
 * [Acknowledgments](#acknowledgments)

## Getting Started

These instructions will get you a copy of the project up and running on your
local machine. This machine (called hub) will act as VPN concentrator. All
other peers connects to hub (as in a "road warrior" configuration).

This version of the script 'wg-cerberus' has been designed for an Alpine linux
router install. For that reason the IP table / routing rules have been removed 
as the configuration should already be configured.

### Prerequisites

Install [Wireguard] for your operating system on [local machine], [router],
[VPS] or [container]. This will be your hub.

As dependences `/bin/sh`, `wg`, `wg-quick`, `awk`, `grep` and `ip` commands
should be available on hub. If `ip` is not available user is required to set
`EXT_NET_IF` and `EXT_NET_IP` variables in script to external network interface
name and IP address (or edit `wghub.conf`). Optionally `qrencode` can be used
to generate [QR codes] for mobile applications.

#### Alpine Linux (Router)
    
    apk add wireguard-tools libqrencode
    
#### Debian, Ubuntu

    sudo apt install wireguard-tools mawk grep iproute2 qrencode

#### Fedora, RHEL, CentOS

    sudo dnf install wireguard-tools gawk grep iproute qrencode

#### FreeBSD

    sudo pkg install net/wireguard graphics/libqrencode

#### Installing Wireguard tools (and modules)

This script requires only tools installed, but to use Wireguard module
(or user-space implementation) is also required. Detailed install guide
for various operating systems is available at [wireguard.com/install].

Peers also requires Wireguard installed. [Android] and [iOS] are supported.

### Installing

Just download the script and make it executable with `chmod`.

When using Alpine linux there is no sudo command by default and the script is designed to be 
installed in the /usr/bin/wg-cerberus

Promote your session with su to do this then

    wget https://raw.githubusercontent.com/tea4tom/wg-cerberus-easy-wg-quick/master/wg-cerberus
    mv wg-cerberus /usr/bin/
    chmod +x /usr/bin/wg-cerberus

Or clone repository.

    git clone https://github.com/burghardt/easy-wg-quick.git

## Usage

The script usage is slightly modified to that of easy-wg-quick although the 
functionality is basically the same.

Script do not require any arguments. Just run it and it will create usable
Wireguard configuration for hub and one peer. Any sequential invocation creates
another peer configuration within same hub.

    ./wg-cerberus # 1st run creates hub configuration
    ./wg-cerberus # any other runs creates additional clients

The hub configuration has a structure within the /etc/wireguard directory, 
in the root you will find the configuration file 'wg0.conf' and two sub 
directories 'ext' and 'peers'. The first stores all the extra configuration 
that gets shared amongst peers, and the latter keeps the peers config files 
and output data.

Passing an argument to script creates peer directory with name instead of
sequence number to help remembering which config was for which device.
Following command will create `/etc/wireguard/{peer_name}/` file.

    ./wg-cerberus peer_name

### Sample setup commands

```
 apk add wireguard-tools libqrencode

 wget https://raw.githubusercontent.com/burghardt/easy-wg-quick/master/easy-wg-quick
 mv wg-cerberus /usr/bin/
 chmod +x /usr/bin/wg-cerberus
 
 wg-cerberus --setup # install config and directory structure
 
 wg-cerberus tom # create a peer in the peers directory /tom/*
 
 wg-cerberus richard # create another peer in the peers directory /richard/*
 
 wg-cerberus harry # create another peer in the peers directory /harry/*

```

### Using generated configuration

On hub configure Wireguard.

    wg-quick up wg0

On peer scan QR code or copy `etc/wireguard/peers/{peer_name}/wgc_{peer_name}.png`. 

To display QR code again use

    qrencode -t ansiutf8 < etc/wireguard/peers/{peer_name}/wgc_{peer_name}.conf

Finally on hub check if everything works with `sudo wg show`.

```
interface: wghub
  public key: kbaG3HxSDz3xhqiTNXlo1fZkFa+V6oTl+w0cSAQKxwQ=
  private key: (hidden)
  listening port: 51820

peer: th8qYu0R0mgio2wPu1kz6/5OOgi6l8iy7OobK590LHw=
  preshared key: (hidden)
  endpoint: 10.60.1.150:37218
  allowed ips: 10.127.0.10/32
  latest handshake: 50 minutes, 22 seconds ago
  transfer: 32.64 MiB received, 95.24 MiB sent
```

## Fine tuning - BELOW THIS POINT IS REFERENCE FROM EASY-WG-QUICK

### Disabling external interface autodetection

By default `easy-wg-quick` use interface with default routing done over it as
external network interface of VPN hub. If autodetection fails or generation of
configuration is done outside the hub (i.e. on [air gapped] laptop) user can
set interface name in `extnetif.txt` file with command:

    echo vtnet0 > extnetif.txt

### Disabling external IP address autodetection

By default `easy-wg-quick` uses IP address of interface that has default
routing done over it as external IP address of VPN hub. This might not be true
if hub is behind firewall or NAT/PAT/masquarading is done. User can set
prefered IP address in `extnetip.txt` file with command:

    echo 192.168.1.2 > extnetip.txt

In case of NAT/PAT/masquarading one can try to use service like [ifconfig.co]
for autodetection:

    curl ifconfig.co/ip > extnetip.txt

### Disabling random port assignment

By default `easy-wg-quick` use random port number from range 1025-65535. When
using static port number is required for firewall configuration or other
reasons user can set preferred port number (80 in this example) in `portno.txt`
file with command:

    echo 80 > portno.txt

### Disabling randomly generated internal network addresses

By default `easy-wg-quick` use randomly generated internal network addresses
for both IPv4 and IPv6. Custom network addresses can be set with the following
commands.

    echo "10.1.1."               > intnetaddress.txt   # for IPv4
    echo "fd90:d175:8e43:705d::" > intnet6address.txt  # for IPv6

Default masks are /24 for IPv4 and /64 for IPv6.

#### Setting network masks

To change default masks set new masks in files named `intnetmask.txt` (IPv4)
and `intnet6mask.txt` (IPv6).

    echo 172.16.0. > intnetaddress.txt
    echo /16       > intnetmask.txt
    echo fd9d:9648:0841:0c6e:3d28:94d9:: > intnet6address.txt
    echo /112                            > intnet6mask.txt

### Setting custom DNS

#### Setting IPv4 resolver address

By default `easy-wg-quick` uses 1.1.1.1 as it's internal DNS. You can use the
command below to serve a custom IPv4 DNS to clients.

    echo 8.8.8.8 > intnetdns.txt

#### Setting IPv6 resolver address

By default `easy-wg-quick` uses 2606:4700:4700::1111 as it's internal DNS. You
can use the command below to serve a custom IPv6 DNS to clients.

    echo 2001:4860:4860::8888 > intnet6dns.txt

### Setting custom client's `AllowedIPs`

By default, the client's `AllowedIPs` variable is set to `0.0.0.0/0, ::/0`,
directing the whole client's traffic through the VPN connection. If you want
to create a VPN [split tunneling] configuration, store required IP addresses
ranges in the `intnetallowedips.txt` file:

    echo '172.16.1.0/24, 172.16.2.0/24' > intnetallowedips.txt

### Choosing firewall type

Firewall type is guessed from operating system. For Linux `iptables` and
`ip6tables` are used. For FreeBSD basic `pf` NAT rules are implemented.
File `fwtype.txt` contains name of firewall type. To override autodetection
or disable any rules run one of the following commands:

    echo iptables  > fwtype.txt  # to choose Linux netfilter
    echo firewalld > fwtype.txt  # to choose [firewalld]
    echo pf        > fwtype.txt  # to choose OpenBSD PF
    echo custom    > fwtype.txt  # to include predefined commands from file
    echo none      > fwtype.txt  # to skip any setup during wg-quick up/down

If `fwtype.txt` contains word `custom` content of `commands.txt` is included
in the `wghub.conf` file.

Format of `commands.txt` is:

    PostUp = echo "command 1"
    PostUp = echo "command 2"
    PostUp = ...

    PostDown = echo "command 1"
    PostDown = secho "command 2"
    PostDown = ...

### Choosing if PostUp/PostDown should enable/disable IP forwarding

Sysctl command syntax is guessed from operating system. Linux and FreeBSD
are supported. As enabling IP forwarding is required for hub to forward VPN
traffic to the Internet it is managed by PostUp/PostDown settings by default.
Some application (i.e. [Docker]) might require that IP forwarding is never
disabled. In that case setting `none` in `sysctltype.txt` and managing IP
forwarding settings [elsewhere] might be required.

File `sysctltype.txt` contains name of sysctl type. To override autodetection
or disable any commands from being run use one of the following commands:

    echo linux   > sysctltype.txt  # to choose Linux sysctl command
    echo freebsd > sysctltype.txt  # to choose FreeBSD sysctl command
    echo none    > sysctltype.txt  # to skip any setup during wg-quick up/down

### Enabling IPv6

If a global unicast IPv6 address is detected on server tunnels will be created
with inner IPv6 addresses allocated. This allows hub's clients to connect over
hub's IPv6 NAT to IPv6 network.

If a global unicast IPv6 address is not detected, the existence of a file
named `forceipv6.txt` can forcibly enable IPv6 support.

    touch forceipv6.txt

To use outer IPv6 addresses (i.e. connect client to hub over IPv6) just set
`EXT_NET_IF` and `EXT_NET_IP` variables in script to external network interface
name and IPv6 address (or edit `wghub.conf`).

### Enabling NDP proxy (instead of default IPv6 masquerading)

By default `easy-wg-quick` uses IPv6 masquerading to provide IPv6 connectivity
to peers. This is easier to setup and require only single IPv6 global unicast
address to work. On the other hand network address translation (NAT) has
[issues and limitations].

[Neighbor Discovery] [Proxies (ND Proxy, NDP Proxy)] allows [end-to-end
connectivity], but requires /64 network to be assigned to hub. From this /64
network, a subnetwork has to be divided (i.e. /112) and assigned to Wireguard
interface.

To enable proxied NDP create file named `ipv6mode.txt` with `proxy_ndp` string.

    echo proxy_ndp > ipv6mode.txt

When hub has 2001:19f0:6c01:1c0d/64 assigned, part of it can be assigned to
Wireguard interface (i.e. 2001:19f0:6c01:1c0d:40/112).

    echo 2001:19f0:6c01:1c0d:40:: > intnet6address.txt
    echo /112 > intnet6mask.txt

Please note that NDP proxy mode in `easy-wg-quick` is supported only on Linux.

### Redirecting DNS

DNS redirection might be required to integrate with services like [Pi-hole] or
[Cloudflare DNS over TLS]. This could be achieved by using port 53 UDP/TCP
redirection in `wghub.conf`.

    PostUp = iptables -t nat -A PREROUTING -i %i -p udp -m udp --dport 53 -j DNAT --to-destination 1.1.1.1:53
    PostUp = iptables -t nat -A PREROUTING -i %i -p tcp -m tcp --dport 53 -j DNAT --to-destination 1.1.1.1:53
    PostDown = iptables -t nat -D PREROUTING -i %i -p udp -m udp --dport 53 -j DNAT --to-destination 1.1.1.1:53
    PostDown = iptables -t nat -D PREROUTING -i %i -p tcp -m tcp --dport 53 -j DNAT --to-destination 1.1.1.1:53

When using IPv6 similar rules should be set independently with `ip6tables`.

    PostUp = ip6tables -t nat -A PREROUTING -i %i -p udp -m udp --dport 53 -j DNAT --to-destination 2606:4700:4700::1111:53
    PostUp = ip6tables -t nat -A PREROUTING -i %i -p tcp -m tcp --dport 53 -j DNAT --to-destination 2606:4700:4700::1111:53
    PostDown = ip6tables -t nat -D PREROUTING -i %i -p udp -m udp --dport 53 -j DNAT --to-destination 2606:4700:4700::1111:53
    PostDown = ip6tables -t nat -D PREROUTING -i %i -p tcp -m tcp --dport 53 -j DNAT --to-destination 2606:4700:4700::1111:53

### Persisting configuration with systemd

[Systemd] may load configuration for both hub and clients using
`wg-quick.service`. Note that also [native support] for setting up Wireguard
interfaces exists (since version 237).

    sudo cp wghub.conf /etc/wireguard/wghub.conf
    sudo systemctl enable wg-quick@wghub
    sudo systemctl start wg-quick@wghub
    systemctl status wg-quick@wghub

## License

This project is licensed under the GPLv2 License - see the [LICENSE] file for
details.

## Acknowledgments

easy-wg-quick [] was the inspiration and source for this script.

OpenVPN's [easy-rsa] was an inspiration for writing this script.

[Wireguard]: https://www.wireguard.com/
[local machine]: https://www.wireguard.com/install/
[router]: https://openwrt.org/docs/guide-user/services/vpn/wireguard
[VPS]: https://en.wikipedia.org/wiki/Virtual_private_server
[container]: https://github.com/activeeos/wireguard-docker
[QR codes]: https://en.wikipedia.org/wiki/QR_code
[wireguard.com/install]: https://www.wireguard.com/install/
[Android]: https://play.google.com/store/apps/details?id=com.wireguard.android
[iOS]: https://itunes.apple.com/us/app/wireguard/id1441195209?ls=1&mt=8
[air gapped]: https://en.wikipedia.org/wiki/Air_gap_%28networking%29
[ifconfig.co]: https://ifconfig.co/
[split tunneling]: https://en.wikipedia.org/wiki/Split_tunneling
[firewalld]: https://firewalld.org/
[Docker]: https://success.docker.com/article/ipv4-forwarding
[elsewhere]: https://en.wikipedia.org/wiki/Sysctl
[issues and limitations]: https://en.wikipedia.org/wiki/Network_address_translation#Issues_and_limitations
[Neighbor Discovery]: https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol
[Proxies (ND Proxy, NDP Proxy)]: https://tools.ietf.org/html/rfc4389
[end-to-end connectivity]: https://en.wikipedia.org/wiki/End-to-end_principle
[Pi-hole]: https://pi-hole.net/
[Cloudflare DNS over TLS]: https://github.com/qdm12/cloudflare-dns-server
[Systemd]: https://wiki.debian.org/systemd
[native support]: https://manpages.debian.org/buster/systemd/systemd.netdev.5.en.html#%5BWIREGUARD%5D_SECTION_OPTIONS
[LICENSE]: LICENSE
[easy-rsa]: https://github.com/OpenVPN/easy-rsa
