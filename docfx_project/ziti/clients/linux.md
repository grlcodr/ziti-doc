## Linux

The linux tunneler is capable of operating in both transparent and proxy modes. In transparent mode the tunneler will
interact with the operating system to establish intercept points. These intercept points correspond directly to the
service definitions in place.

`ziti-tunnel` supports several intercept modes, which are specified with a sub-command.
Typically you will run `ziti-tunnel` with the _run_ sub-command, which determines
the preferred intercept mode based on the kernel drivers that are available on the
host.

    $ sudo ziti-tunnel run ziti.json
    [   0.000]    INFO ziti/tunnel/intercept/tproxy.New: tproxy listening on 127.0.0.1:33641
    [   0.006]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.run: using tproxy interceptor
    ...

When started on a host that lacks __iptables__, the tproxy intercept mode initializer
fails and `ziti-tunnel` attempts to use the tun intercept mode:

    $ sudo ziti-tunnel run ziti.json
    [   0.001]    INFO ziti/tunnel/intercept/tproxy.New: tproxy listening on 127.0.0.1:37313
    [   0.001]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.run: tproxy initialization failed: failed to initialize iptables handle: exec: "iptables": executable file not found in $PATH
    [   0.009]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.run: using tun interceptor

`ziti-tunnel` fails to start if no intercept modes can be successfully initialized:

    $ sudo ziti-tunnel run ziti.json
    [   0.000]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.run: tproxy initialization failed: failed to initialize iptables handle: exec: "iptables": executable file not found in $PATH
    [   0.001]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.run: tun initialization failed: failed to open tun interface (name='', mtu=0): ioctl failed with 'invalid argument'
    [   0.001]   FATAL ziti/tunnel/cmd/ziti-tunnel/subcmd.run: failed to initialize an interceptor

### Intercept Modes

#### tproxy

tproxy is the preferred intercept mode when running on a Linux kernel that has the __ip_tables__ kernel module installed.

    $ lsmod | grep ip_tables
    ip_tables              32768  5 iptable_filter,iptable_security,iptable_raw,iptable_nat,iptable_mangle

`ziti-tunnel` manipulates routing tables and firewall rules when using the tproxy
intercept mode. The NET_ADMIN Linux capability is required for these actions. The
usage example here runs ziti-tunnel with sudo as a simple way to obtain
NET_ADMIN:

    $ sudo ziti-tunnel --identity ziti.json tproxy
    [   0.000]    INFO ziti/tunnel/intercept/tproxy.New: tproxy listening on 127.0.0.1:33355
    [   0.010]    INFO ziti/tunnel/dns.NewDnsServer: starting dns server...
    [   2.018]    INFO ziti/tunnel/dns.NewDnsServer: dns server running at 127.0.0.1:53
    [   2.018]    INFO ziti/tunnel/dns.(*resolver).AddHostname: adding ziti-tunnel.resolver.test = 19.65.28.94 to resolver
    [   2.033]    INFO ziti/tunnel/dns.(*resolver).RemoveHostname: removing ziti-tunnel.resolver.test from resolver
    [   2.096]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.updateServices: starting tunnel for newly available service wttr.in
    [   2.290]    INFO ziti/tunnel/dns.(*resolver).AddHostname: adding wttr.in = 5.9.243.187 to resolver
    [   2.300]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.updateServices: service wttr.in not hostable
    [   2.300]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.updateServices: starting tunnel for newly available service ssh-local
    [   2.570]    INFO ziti/tunnel/dns.(*resolver).AddHostname: adding local.io = 169.254.1.1 to resolver

The tproxy intercept mode creates a network listener that accepts connections at a
randomly selected port on the loopback interface. Intercepted ziti service traffic
directed to the listener by two mechanisms:

* Firewall Rules (iptables)

    The TPROXY iptables target is the primary intercept mechanism used by the tproxy
    intercept mode. The TPROXY target essentially sends packets to a local listener
    without actually modifying the packet's destination address fields. See
    https://www.kernel.org/doc/Documentation/networking/tproxy.txt and
    `iptables-extensions(8)` for more details on the TPROXY target.

    First, the tproxy interceptor links a new iptables chain to the PREROUTING chain:

        $ sudo iptables -nt mangle -L PREROUTING | grep NF-INTERCEPT
        NF-INTERCEPT  all  --  0.0.0.0/0            0.0.0.0/0

    Then it creates rules in the new chain for each intercepted service. You can view
    the tproxy rules in play:

        $ sudo iptables -nt mangle -L NF-INTERCEPT
        Chain NF-INTERCEPT (1 references)
        target     prot opt source               destination         
        TPROXY     tcp  --  0.0.0.0/0            5.9.243.187          /* wttr.in */ tcp dpt:443 TPROXY redirect 127.0.0.1:33355 mark 0x1/0x1
        TPROXY     tcp  --  0.0.0.0/0            169.254.1.1          /* ssh-local */ tcp dpt:22 TPROXY redirect 127.0.0.1:33355 mark 0x1/0x1
        TPROXY     tcp  --  0.0.0.0/0            1.2.3.4              /* netcat */ tcp dpt:22169 TPROXY redirect 127.0.0.1:33355 mark 0x1/0x1

    Packets with a destination address that matches the intercept address of a Ziti
    service are directed to ziti-tunnel's network listener (127.0.0.1:33355 in the
    examples above). This effectively enables `ziti-tunnel` to capture packets that
    are destined for any address using a single listener (and a single port).

    NOTE: _netfilter_ rules were considered when implementing ziti-tunnel's tproxy
    intercept mode. _netfilter_ is a slightly more modern than _iptables_ and has
    a supported netlink API for manipulating rules without "shelling out" to the
    `iptables` command line utility. _netfilter_ was ultimately abandoned because
    netfilter tproxy support requires kernel configuration options (`CONFIG_NFT_TPROXY`,
    `CONFIG_NFT_SOCKET`) that are not enabled in the default kernels of many common
    Linux distributions.

* Local Routes

    The TPROXY target is only valid in the PREROUTING iptables chain, which is
    traversed by incoming packets that were routed to the host over the network. A
    _local_ route is necessary in order to get locally generated packets to traverse
    the PREROUTING chain:

        $ ip route show table local
        local 1.2.3.4 dev lo proto kernel scope host src 1.2.3.4
        local 5.9.243.187 dev lo proto kernel scope host src 5.9.243.187
        local 169.254.1.1 dev lo proto kernel scope host src 169.254.1.1

#### tun

The tun intercept mode creates an ephemeral tun interface and configures it with the
IP addresses of the services that are being proxied.

`ziti-tunnel` manipulates network interfaces when using the tun intercept mode,
which requires the NET_ADMIN Linux capability. The usage example here runs
ziti-tunnel with sudo as a simple way to obtain NET_ADMIN:

    $ sudo ziti-tunnel --identity ziti.json tun
    [   0.010]    INFO ziti/tunnel/dns.NewDnsServer: starting dns server...
    [   2.012]    INFO ziti/tunnel/dns.NewDnsServer: dns server running at 127.0.0.1:53
    [   2.012]    INFO ziti/tunnel/dns.(*resolver).AddHostname: adding ziti-tunnel.resolver.test = 19.65.28.94 to resolver
    [   2.031]    INFO ziti/tunnel/dns.(*resolver).RemoveHostname: removing ziti-tunnel.resolver.test from resolver
    [   2.089]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.updateServices: starting tunnel for newly available service wttr.in
    [   2.280]    INFO ziti/tunnel/dns.(*resolver).AddHostname: adding wttr.in = 5.9.243.187 to resolver
    [   2.282]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.updateServices: service wttr.in not hostable
    [   2.282]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.updateServices: starting tunnel for newly available service ssh-local
    [   2.502]    INFO ziti/tunnel/dns.(*resolver).AddHostname: adding myhost.io = 169.254.1.2 to resolver
    [   2.505]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.updateServices: service ssh-local not hostable
    [   2.505]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.updateServices: starting tunnel for newly available service netcat
    [   2.506]    INFO ziti/tunnel/cmd/ziti-tunnel/subcmd.updateServices: service netcat not hostable
    ...

The addresses that `ziti-tunnel` adds to the tun interface are __point-to-point__
addresses:

    $ ip addr show dev tun0
    10: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 65535 qdisc fq_codel state UNKNOWN group default qlen 500
        link/none 
        inet 169.254.1.1/32 scope host tun0
           valid_lft forever preferred_lft forever
        inet 169.254.1.1 peer 5.9.243.187/32 scope host tun0
           valid_lft forever preferred_lft forever
        inet 169.254.1.1 peer 169.254.1.2/32 scope host tun0
           valid_lft forever preferred_lft forever
        inet 169.254.1.1 peer 1.2.3.4/32 scope host tun0
           valid_lft forever preferred_lft forever

The tun interface itself is assigned a link-local address, 169.254.1.1 in this
case, and each intercepted service is represented by a point-to-point address with
the remote address matching the intercept IP of the Ziti service. The tun intercept
mode uses point-to-point addresses instead of local routes because local routes
would result in the Linux networking stack receiving the packets that are routed to
the tun interface. The point-to-point addresses ensure that the packets are delivered
"to the wire", which mean that, for a tun interface, the packets will be picked up
by `ziti-tunnel` when it reads data from the tun interface.

#### proxy

[!include[](./proxy-example.md)]

### DNS Server
ziti-tunnel runs an internal DNS server by default. The DNS server must be first in the host's resolver
configuration (e.g. resolve.conf). A self-test is performed when ziti-tunnel starts to ensure that its
internal DNS server is configured in the system resolver:

    INFO[0002] dns server started on 127.0.0.1:53           
    INFO[0002] adding ziti-tunnel.resolver.test -> 19.65.28.94 to resolver 
    INFO[0002] removing ziti-tunnel.resolver.test from resolver 

The test involves inserting a known hostname/IP address into the internal DNS server, and using the system
resolver to retrieve the address of the hostname. *ziti-tunnel will exit if the DNS self-test fails.*

Linux distributions typically manage the contents of /etc/resolv.conf, so simply editing the file
will only work for a short time until /etc/resolv.conf is overwritten by the managing process.

Resolver configuration changes must survive restarts of the Linux name resolution manager. Linux
distrubutions use one of several name resolution managers. The simplest way to determine which name
resolution manager is being used by your Linux distrubtion is to look at /etc/resolv.conf:

    $ ls -l /etc/resolv.conf

* If /etc/resolv.conf is a regular file, then it is most likely being managed by `dhclient`.
* If /etc/resolv.conf is a symlink to a file in /run/systemd/resolve, then it is being
  managed by `systemd-resolved`

#### dhclient

If your Linux distribution uses dhclient, you can configure the system resolver to use
ziti-tunnel's internal DNS server first by adding the following to /etc/dhcp/dhclient.conf:

    prepend domain-name-servers 127.0.0.1;

Then restart network manager. Unless you know the name of the NetworkManager systemd
service on your Linux distrubtion, it's probably easiest to reboot the host.

#### systemd-resolved

    $ sudo ln -sf /run/systemd/resolve/resolv.conf /etc
    $ echo -e "[Resolve]\nDNS=127.0.0.1" | sudo tee /etc/systemd/resolved.conf.d/ziti-tunnel.conf
    $ sudo systemctl restart systemd-resolved

If you are unable to control the resolver on your operating system, ziti-tunnel can use/update a hosts file for
any hostnames that it tunnels:

    ziti-tunnel run --resolver file:///etc/hosts "${HOME}/ziti.json"

#### IP Address Assignment

If the service specifies a hostname for its address, ziti-tunnel resolves the hostname and adds the result to its
internal DNS server:

    [0127]  INFO adding myservice.mydomain.com -> 45.60.32.165 to resolver

If the service hostname does not resolve, ziti-tunnel will find an unused link-local address and assign it to
the route for the service:

    [0012]  INFO adding bogushost.net -> 169.254.1.4 to resolver
    [0012]  INFO ziti/tunnel/protocols/tcp.Listen: Accepting on 169.254.1.4:25 service=telnet

### Troubleshooting

The simplest step you can take toward a diagnosis is to reduce the minimum message log level by adding `--verbose` to your command. This usually means lower-level DEBUG messages and above are emitted in addition to the default level of INFO level and above e.g. WARN, ERROR, etc...

If the tunneler is crashing then it may be crucial to collect and analyze the core dump file. You may need to enable saving core dumps depending upon your OS configuration. 

You can see how dump files are handled by inspecting this file, which is from Ubuntu 20.10.

```bash
❯ cat /proc/sys/kernel/core_pattern
|/usr/share/apport/apport %p %s %c %d %P %E
```

In this case the dump is handled by `apport` which saves the file in `/var/crash`. I'll need to follow the `apport` documentation to learn how to unpack and parse the dump file.

