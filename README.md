SSH Proxy Command -- connect.c
==============================

`connect.c` is a simple relaying command to make network connection
via SOCKS and https proxy. It is mainly intended to be used as proxy
command of OpenSSH. You can make SSH session beyond the firewall with
this command,

Features of `connect.c` are:

* Supports SOCKS (version 4/4a/5) and https CONNECT method.
* Supports NO-AUTH and USERPASS authentication of SOCKS5
* You can input password from tty, `ssh-askpass` or environment variable.
* Run on UNIX or Windows platform.
* You can compile with various C compiler (cc, gcc, Visual C, Borland C. etc.)
* Simple and general program independent from OpenSSH.
* You can also relay local socket stream instead of standard I/O.

You can clone [source code on github](https://github.com/gotoh/ssh-connect) .


## What is proxy command?

OpenSSH development team decides to stop supporting SOCKS and any
other tunneling mechanism. It was aimed to separate complexity to
support various mechanism of proxying from core code. And they
recommends more flexible mechanism: ProxyCommand option instead.

Proxy command mechanism is delegation of network stream
communication. If ProxyCommand options is specified, SSH invoke
specified external command and talk with standard I/O of thid
command. Invoked command undertakes network communication with
relaying to/from standard input/output including iniitial
communication or negotiation for proxying. Thus, ssh can split out
proxying code into external command.

The `connect.c` program was made for this purpose.

## How to Use

### Get Source

You can get source code from [project  page](https://github.com/gotoh/ssh-connect).
```
git clone https://github.com/gotoh/ssh-connect.git
```

### Compile and Install

In most environment, you can compile `connect.c` simply. On UNIX
environment, you can use cc or gcc. On Windows environment, you can
use Microsoft Visual C, Borland C or Cygwin gcc.

| environment | command to compile |
|-------------|--------------------|
| UNIX cc | `cc connect.c -o connect` |
| UNIX gcc | `gcc connect.c -o connect` |
| Solaris | `gcc connect.c -o connect -lnsl -lsocket -lresolv` |
| Microsoft Visual C/C++ |  `cl connect.c wsock32.lib advapi32.lib` |
| Borland C |  `bcc32 connect.c wsock32.lib advapi32.lib` |
| Cygwin gcc | `gcc connect.c -o connect` |
| Mac OS/Darwin | `gcc connect.c -o connect -lresolv` |


To install connect command, simply copy compiled binary to directory
in your `PATH` (ex. `/usr/local/bin`). Like this:

```
$ cp connect /usr/local/bin
```

### Modify your `~/.ssh/config`

Modify your `~/.ssh/config` file to use connect command as proxy
command. For the case of SOCKS server is running on firewall host
socks.local.net with port 1080, you can add `ProxyCommand` option in
`~/.ssh/config`, like this:

```
Host remote.outside.net
  ProxyCommand connect -S socks.local.net %h %p
```

`%h` and `%p` will be replaced on invoking proxy command with target
hostname and port specified to SSH command.

If you hate writing many entries of remote hosts, following example
may help you.

```
## Outside of the firewall, use connect command with SOCKS conenction.
Host *
  ProxyCommand connect -S socks.local.net %h %p

## Inside of the firewall, use connect command with direct connection.
Host *.local.net
  ProxyCommand connect %h %p
```

If you want to use http proxy, use `-H` option instead of `-S` option
in examle above, like this:

```
## Outside of the firewall, with HTTP proxy
Host *
  ProxyCommand connect -H proxy.local.net:8080 %h %p

## Inside of the firewall, direct
Host *.local.net
  ProxyCommand connect %h %p
```


### Use SSH

After editing your `~/.ssh/config` file, you are ready to use ssh. You
can execute ssh without any special options as if remote host is IP
reachable host. Following is an example to execute hostname command on
host `remote.outside.net`.

```
local$ ssh remote.outside.net hostname
Hello, this is remote.outside.net
remote$
```


### Have a trouble?

If you have trouble, execute connect command from command line with `-d`
option to see what is happened. Some debug message may appear and
reports progress. This information may tell you what is wrong. In this
example, error has occurred on authentication stage of SOCKS5
protocol.

```
$ connect -d -S socks.local.net unknown.remote.outside.net 110
DEBUG: relay_method = SOCKS (2)
DEBUG: relay_host=socks.local.net
DEBUG: relay_port=1080
DEBUG: relay_user=gotoh
DEBUG: socks_version=5
DEBUG: socks_resolve=REMOTE (2)
DEBUG: local_type=stdio
DEBUG: dest_host=unknown.remote.outside.net
DEBUG: dest_port=110
DEBUG: Program is $Revision: 1.20 $
DEBUG: connecting to xxx.xxx.xxx.xxx:1080
DEBUG: begin_socks_relay()
DEBUG: atomic_out()  [4 bytes]
DEBUG: >>> 05 02 00 02
DEBUG: atomic_in() [2 bytes]
DEBUG: <<< 05 02
DEBUG: auth method: USERPASS
DEBUG: atomic_out()  [some bytes]
DEBUG: >>> xx xx xx xx ...
DEBUG: atomic_in() [2 bytes]
DEBUG: <<< 01 01
ERROR: Authentication faield.
FATAL: failed to begin relaying via SOCKS.
```


## More Detail

Command line usage is here:

```
usage:  connect [-dnhs45] [-R resolve] [-p local-port] [-w sec]
		[-H [user@]proxy-server[:port]]
		[-S [user@]socks-server[:port]]
		host port
```

host and port is target hostname and port-number to connect.


`-H` [user@]server[:port]::
  Specify hostname and port number of http proxy server to
  relay. If port is omitted, 80 is used.

`-h`::
  Use HTTP proxy via proxy server sepcified by environment variable
  `HTTP_PROXY`.

`-S` \[_user_@]_server_\[:_port_]::
  Specify hostname and port number of SOCKS server to
  relay. Like `-H` option, port number can be omit and default is 1080.

`-s`::
  Use SOCKS proxy via SOCKS server sepcified by environment variable
  `SOCKS5_SERVER`.


`-4`:: Use SOCKS version 4 protocol.
  This option must be used with `-S`.
`-5`:: Use SOCKS version 5 protocol.
  This option must be used with `-S`.

`-R` _method_:: The method to resolve hostname.  3 keywords (`local`,
  `remote`, `both`) or dot-notation IP address is allowed. Keyword
  both means; _"Try local first, then remote"_. If dot-notation IP
  address is specified, use this host as nameserver (UNIX
  only). Default is remote for SOCKS5 or local for others. On SOCKS4
  protocol, remote resolving method (remote and both) use protocol
  version 4a.

`-p` _port_:: Accept on local TCP port and relay it instead of standard input
and output. With this option, program will terminate when remote or
local TCP session is closed.

`-w` _timeout_:: Timeout seconds for connecting to remote host.

`-a` _auth_:: option specifiys user intended authentication methods
separated by comma. Currently `userpass` and `none` are
supported. Default is userpass. You can also specifying this parameter
by the environment variable `SOCKS5_AUTH`.

`-d`: Run with debug message output. If you fail to connect, use this
option to see what is done.

As additional feature, 
you can omit port argument when program name is special format
containing port number itself like "connect-25". For example:

```
$ ln -s connect connect-25
$ ./connect-25 smtphost.outside.net
220 smtphost.outside.net ESMTP Sendmail
QUIT
221 2.0.0 smtphost.remote.net closing connection
$
```

This example means that the command name "connect-25" indicates port
number 25 so you can omit 2nd argument (and used if specified
explicitly).
This is usefull for the application which invokes only with hostname
argument.


### Specifying user name via environment variables


There are 5 environemnt variables to specify user name without command
line option. This mechanism is usefull for the user who using another
user name different from system account.

| variable | description |
| ----- | ---- |
| `SOCKS5_USER` | Used for SOCKS v5 access.  |
| `SOCKS4_USER` | Used for SOCKS v4 access.  |
| `SOCKS_USER` |  Used for SOCKS v5 or v4 access and varaibles above are not defined.  |
|  `HTTP_PROXY_USER` |   Used for HTTP proxy access.  |
| `CONNECT_USER` |  Used for all type of access if all above are not defined.  |

Following table describes how user name is determined. Left most number is order to check. If variable is not defined, check next variable, and so on.

| Protocol | variables order |
|----|----|
| SOCKS v5 | `SOCKS5_USER` or `SOCKS_USER` or `CONNECT_USER` or ask |
| SOCKS v4 | `SOCKS4_USER` or `SOCKS_USER` or `CONNECT_USER` or ask |
| HTTP Proxy | `HTTP_PROXY_USER` or `CONNECT_USER`  or ask |


### Specifying password via environment variables

There are 5 environemnt variables to specify password. If you use this
feature, please note that it is not secure way.


| variable | description |
|----|----|
| `SOCKS5_PASSWD` |  Used for SOCKS v5 access. This variables is compatible with NEC SOCKS implementation. |
| `SOCKS5_PASSWORD` |  Used for SOCKS v5 access if `SOCKS5_PASSWD` is not defined. |
| `SOCKS_PASSWORD` | Used for SOCKS v5 (or v4) access all above is not defined.  |
| `HTTP_PROXY_PASSWORD` | Used for HTTP proxy access.  |
| `CONNECT_PASSWORD` |Used for all type of access if all above are not defined.  |

Following table describes how password is determined. First, left most variable is checked.
If the variable is not defined, check next variable, and so on. Finally ask to user interactively using external program or tty input.

| Protocol | variables order |
|----|----|
| SOCKS v5 | `SOCKS5_PASSWD` or `SOCKS5_PASSWORD` or `SOCKS_PASSWORD` or ask |
| HTTP Proxy | `HTTP_PROXY_PASSWORD` or `CONNECT_PASSWORD`  or ask |




## Limitations

### SOCKS5 authentication

Only NO-AUTH and USER/PASSWORD authentications are supported. GSSAPI
authentication (RFC 1961) and other draft authentications (CHAP, EAP,
MAF, etc.) is not supported.


### HTTP authentication

BASIC authentication is supported but DIGEST authentication is not.


### Switching proxy server on event

There is no mechanism to switch proxy server regarding to PC
environment. This limitation might be bad news for mobile user. Since
I do not want to make this program complex, I do not want to support
although this feature is already requested. Please advice me if there
is good idea of detecting environment to swich and simple way to
specify conditioned directive of servers.

One tricky workaround exists. It is replacing `~/.ssh/config` file by
script on ppp up/down.

There's another example of wrapper script (contributed by Darren
Tucker). This script costs executing ifconfig and grep to detect
current environment, but it works.  Note that you should modify
addresses if you use it.

```
#!/bin/sh
## ~/bin/myconnect --- Proxy server switching wrapper

if ifconfig eth0 |grep "inet addr:192\.168\.1" >/dev/null; then
	opts="-S 192.168.1.1:1080"  
elif ifconfig eth0 |grep "inet addr:10\." >/dev/null; then
	opts="-H 10.1.1.1:80"
else
	opts="-s"
fi
exec /usr/local/bin/connect $opts $@
```


## Tips

### Proxying socket connection

In usual, `connect.c` relays network connection to/from standard
input/output. By specifying -p option, however, `connect.c` relays local
network stream instead of standard input/output. With this option,
connect command waits connection from other program, then start
relaying between both network stream.

This feature may be useful for the program which is hard to SOCKSify.


### Use with ssh-askpass command

`connect.c` ask you password when authentication is required. If you
are using on tty/pty terminal, connect can input from terminal with
prompt. But you can also use ssh-askpass program to input password. If
you are graphical environment like X Window or MS Windows, and program
does not have tty/pty, and environment variable `SSH_ASKPASS` is
specified, then `connect.c` invoke command specified by environment
variable SSH_ASKPASS to input password. ssh-askpass program might be
installed if you are using OpenSSH on UNIX environment. On Windows
environment, pre-compiled binary is available from here.

This feature is limited on window system environment.

And also useful on Emacs on MS Windows (NT Emacs or Meadow). It is
hard to send passphrase to connect command (and also ssh) because
external command is invoked on hidden terminal and do I/O with this
terminal. Using ssh-askpass avoids this problem.


### Use for Network Stream of Emacs

Although `connect.c` is made for OpenSSH, it is generic and independent
from OpenSSH. So we can use this for other purpose. For example, you
can use this command in Emacs to open network connection with remote
host over the firewall via SOCKS or HTTP proxy without SOCKSifying
Emacs itself.

There is sample code: 
https://github.com/gotoh/ssh-connect/blob/master/relay.el

With this code, you can use `relay-open-network-stream` function instead
of `open-network-stream` to make network connection. See top comments of
the source for more detail.


### Remote resolver


If you are SOCKS4 user on UNIX environment, you might want specify
nameserver to resolve remote hostname. You can do it specifying `-R`
option followed by IP address of resolver.


### Hopping Connection via SSH

Conbination of ssh and connect command have more interesting
usage. Following command makes indirect connection to host2:port from
your current host via host1.

```
$ ssh host1 connect host2 port
```

This method is useful for the situations like:

- You are outside of organizasion now, but you want to access an
  internal host barriered by firewall.

- You want to use some service which is allowed only from some limited hosts.

For example, I want to use local NetNews service in my office from
home. I cannot make NNTP session directly because NNTP host is
barriered by firewall. Fortunately, I have ssh account on internal
host and allowed using SOCKS5 on firewall from outside. So I use
following command to connect to NNTP service.

```
$ ssh host1 connect news 119
200 news.my-office.com InterNetNews NNRP server INN 2.3.2 ready (posting ok).
quit
205 .
$
```

By combinating hopping connection and relay.el, I can read NetNews
using http://www.gohome.org/wl/[Wanderlust] on Emacs at home.

```
                        |
    External (internet) | Internal (office)
                        |
+------+           +----------+          +-------+           +-----------+
| HOME |           | firewall |          | host1 |           | NNTP host |
+------+           +----------+          +-------+           +-----------+
 emacs <-------------- ssh ---------------> sshd <-- connect --> nntpd
       <-- connect --> socksd <-- SOCKS -->
```

As an advanced example, you can use SSH hopping as fetchmail's plug-in
program to access via secure tunnel. This method requires that connect
program is insatalled on remote host. There's example of .fetchmailrc
bellow. When fetchmail access to mail-server, you will login to remote
host using SSH then execute connect program on remote host to relay
conversation with pop server. Thus fetchmail can retrieve mails in
secure.

```
poll mail-server
  protocol pop3
  plugin "ssh %h connect localhost %p"
  username "username"
  password "password"
```


## Break The More Restricted Wall

If firewall does not provide SOCKS nor HTTPS other than port 443, you
cannot break the wall in usual way. But if you have you own host which
is accessible from internet, you can make ssh connection to your own
host by configuring sshd as waiting at port 443 instead of standard

22. By this, you can login to your own host via port 443. Once you
have logged-in to extenal home machine, you can execute connect as
second hop to make connection from your own host to final target host,
like this:

```
internal$ cat ~/.ssh/config
Host home
    ProxyCommand connect -H firewall:8080 %h 443

Host server # internal
    ProxyCommand ssh home connect %h %p
    
internal$ ssh home
You are logged in to home!
home# exit
internal$ ssh server
You are logged in to server!
server# exit
internal$
```

This way is similar to "Hopping connection via SSH" except configuring
outer sshd as waiting at port 443 (https). This means that you have a
capability to break the strongly restricted wall if you have own host
out side of the wall.

```
                        |
      Internal (office) | External (internet)
                        |
+--------+         +----------+                 +------+          +--------+
| office |         | firewall |                 | home |          | server |
+--------+         +----------+                 +------+          +--------+
   <------------------ ssh --------------------->sshd:443
    <-- connect --> http-proxy <-- https:443 -->                      any
                                                 connect <-- tcp -->  port
```

NOTE: If you wanna use this, you should give up hosting https
      service at port 443 on you external host 'home'.


## F.Y.I.

### Difference between SOCKS versions

SOCKS version 4 is first popular implementation which is documented
[here](http://www.socks.nec.com/protocol/socks4.protocol). Since this
protocol provide IP address based requesting, client program should
resolve name of outer host by itself. Version 4a (documented [here](http://www.socks.nec.com/protocol/socks4a.protocol) is
enhanced to allow request by hostname instead of IP address.

SOCKS version 5 is re-designed protocol stands on experience of
version 4 and 4a. There is no compativility with previous
versions. Instead, there's some improvement: IPv6 support, request by
hostname, UDP proxying, etc.


### Configuration to use HTTPS

Many http proxy servers implementation supports https CONNECT method
(SLL). You might add configuration to allow using https. For the
example of [DeleGate](http://www.delegate.org/delegate/) (DeleGate is a
multi-purpose application level gateway, or a proxy server) , you
should add https to REMITTABLE parameter to allow HTTP-Proxy like
this:

```
delegated -Pxxxx ...... REMITTABLE='+,https' ...
```

For the case of Squid, you should allow target ports via https by ACL,
and so on.


### SOCKS5 Servers

- [NEC SOCKS Reference Implementation](http://www.socks.nec.com/refsoftware.html): 
  Reference implementation of SOKCS server and library.

- [Dante](http://www.inet.no/dante/index.html):
  Dante is free implementation of SOKCS server and library. Many enhancements and modulalized.

- [DeleGate](http://www.delegate.org/delegate/):
  DeleGate is multi function proxy service provider. DeleGate 5.x.x
  or earlier can be SOCKS4 server, and 6.x.x can be SOCKS5 and
  SOCKS4 server. and 7.7.0 or later can be SOCKS5 and SOCKS4a
  server.


### Specifications

- [socks4.protocol.txt](http://www.socks.nec.com/protocol/socks4.protocol):
  SOCKS: A protocol for TCP proxy across firewalls 
- [socks4a.protocol.txt](http://www.socks.nec.com/protocol/socks4a.protocol):
 SOCKS 4A: A Simple Extension to SOCKS 4 Protocol 
- [RFC 1928](http://www.socks.nec.com/rfc/rfc1928.txt):
  SOCKS Protocol Version 5 
- [RFC 1929](http://www.socks.nec.com/rfc/rfc1929.txt):
  Username/Password Authentication for SOCKS V5 
- [RFC 2616](http://www.ietf.org/rfc/rfc2616.txt):
 Hypertext Transfer Protocol -- HTTP/1.1 
- [RFC 2617](http://www.ietf.org/rfc/rfc2617.txt):
 HTTP Authentication: Basic and Digest Access Authentication 


### Related Links

- [OpenSSH Home](http://www.openssh.org/)
- [Proprietary SSH](http://www.ssh.com/)
- [Using OpenSSH through a SOCKS compatible PROXY on your LAN](http://www.taiyo.co.jp/~gotoh/ssh/openssh-socks.html) (J. Grant)


### Similars

- [Proxy Tunnel](http://proxytunnel.sourceforge.net/):
  Proxying command using https CONNECT.
- [stunnel](http://www.snurgle.org/~griffon/ssh-https-tunnel):
  Proxy through an https tunnel (Perl script)
