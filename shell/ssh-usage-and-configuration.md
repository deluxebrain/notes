# SSH Usage and Configuration

## SSH configuration via `ssh_config`

### Configuration files
SSH configuration is controlled through two files:

* `/etc/ssh/ssh_config` controls system-wide SSH configuration
* `~/.ssh/config` are user-overrides

It is also possible to specify an alternative configuration file using the `-F` option, e.g:
`ssh -F ~/.ssh/<alternative_config> <host>`

### Per-Host Configuration
SSH configuration for a particular system is controlled using the `Host` keyword. This matches on ip or dns and must be an exact case-sensitive match to what is typed at the command line.

#### Matching groups

The `Host` keyword allows matching of IP or DNS groups, e.g:

* `Host *.example.com`
* `Host 192.168.*.0`

#### Matching mulitple hosts

Match multiple hosts by specifying them on the same line, separated by spaces, e.g:

`Host <some_ip> <some_dns> <some_other_dns>`

#### Specifying default configuration

Defaults are specified at the end of the file and are used by **servers not matched by previous `Host` entries**.

E.g:
```script
Host <some_host>
	Port <some_port>
	...

# defaults for non-matched hosts
Port <some_other_port>
```

## SSH agent forwarding

**tl;dr;** Dont use SSH agent forwarding due to its associated security implications. Nothing in this documentation relies on SSH agent forwarding for its operation.

SSH agent forwarding allows you to access a chain of SSH servers without installing your private key onto any of them by chaining SSH authentication back to the originating SSH client.

With SSH agent forwarding, all authentication requests are forwarded back to the SSH client on your local machine. Agent forwarding creates a socket on each originating SSH server and sets its location in the `$SSH_AUTH_SOCK` environment variable. SSH access from tha SSH server out to other SSH servers is then authenticated via this socket back to the SSH client on your machine.

**Note SSH agent forwarding presents a security risk in that if the remote server is compromised the intruder can use the socket to authenticate across machines. Where possible use the SSH `ProxyCommand` instead.**

### Limiting SSH agent forwarding

Due to the risks associated with SSH agent forwarding, it should only be used when explicitly required and for trusted servers. Best practice is to disabled it system-wide, and then re-enable it per host.

```script
# Within /etc/ssh_config
AllowAgentForwarding no

# Within ~/.ssh/config
Host <my_trusted_host>
	ForwardAgent yes
```

### Using SSH agent forwarding

To allows password-less logins, the public key of the originating user will need to be installed into the `~/.ssh/authorized_keys` of each host in the SSH chain.

#### SSH agent forwarding via the command-line

* Use the `-A` option to enable agent forwarding.
* Use the `-t` option on all `ssh` commands that specify a command to be run to force a pseudo-tty to be attached and prevent it exiting after running the command.

To SSH to **Server3** via **Server1** and then **Server2**:
`$ ssh -A -t Server1 ssh -A -t Server2 ssh -A Server 3` 

#### Simplification using `ssh_config`

```shell
Host Jump
	HostName <some_ip> or <some_dns>
	User <my_user>
	ForwardAgent yes

Host Destination
	HostName <some_ip> or <some_dns>
	User <my_user>

# Use the -t option to prevent the SSH connection to Jump from exiting
$ ssh -t Jump ssh Destination
```

## The SSH `ProxyCommand`

SSH `proxycommand` offers the same functionality as SSH agent forwarding  of connecting to SSH servers via a series of intermediaries. However, it has none of the security risks that comes with agent forwarding associated with the `$SSH_AUTH_SOCK`.

### Using the SSH `ProxyCommand`

The SSH `ProxyCommand` is used alongside the **netcat* utility to pass connections onto secondary SSH servers.

> As of OpenSSH version 5.4, the SSH `proxycommand` option can be run in a **netcat mode** that does not need the explicit use (nor installation) of the `netcat` utility. 

Authentication happens on all SSH servers in the chain and hence the public key asscociated with the user must be installed in the `~/.ssh/authorized_keys` files of each host.

#### Using the SSH `ProxyCommand` from the command-line

* Use the `-o` option to specify the `ProxyCommand`
* This example uses the **netcat mode** to prevent the explcit need to the use of the **netcat** utility

`$ ssh -o ProxyCommand="ssh -W %h:%p JumpServer" DestinationServer`

#### Simplification using the `ssh_config`

```script
# Jump server
Host JumpServer
	HostName <some_ip> or <some_dns>
	User <my_user>
	Port <my_port> # if not 22
	IdentityFile ~/.ssh/<my_key> # if not id_rsa

# Destination server
Host DestinationServer
	HostName ...
	User ...
	ProxyCommand ssh -W %h:%p JumpServer

# Usage:
# $ ssh DestinationServer
```

## SSH port forwarding

Port forwarding allows SSH to serve as an encrypted wrapper around any TCP traffic, proving a secure transport between the client and server machines.

### Local port forwarding

Local port forwarding redirects all traffic from a local port on the client machine to a remote port on the server. Typical uses are as follows:

* Encrpyt otherwise unsecured network traffic. For example, wrap http traffic to a remote web server.
* Tunnel access to remote services through a firewall. For example, access a remote database server behind a firewall by tunnelling the database network traffic within an SSH connection.

> On Unix-like systems, TCP ports below 1024 are reserved and can only be binded to with `root` access. 

#### Local port forwarding from the command-line

* Use the `-L` flag to activate local forwarding

`$ ssh -L <local_ip>:<local_port>:<remote_ip>:<remote_port> <user>@<hostname>`

SSH attaches to `127.0.0.1` by default, allowing the simplified form:

`$ ssh -L <local_port>:<remote_ip>:<remote_port> <user>@<hostname>`

#### Simplification using the `ssh_config`

In usage:

* Use the `-N` flag to state that no command will be executed
* background using the `&` keyword

```shell
Host JumpServer
	HostName <host_address>
	User <my_user>
	IdentityFile ~/.ssh/<my_key> # if not id_rsa
	
Host DestinationServer
	HostName <host_address>
	User <my_user>
	IdentityFile ~/.ssh/<my_key> # if not id_rsa
	LocalForward <local_port> <remote_host>:<remote_port>

# Usage
$ ssh -N DestinationServer &
```

### Remote port forwarding

Remote port forwarding redirects all traffic on a remote port on the server to a local port on the client. Typical uses are as follows:

* Access a client machine behind a firewall from a publicly facing server. 

#### Remote port forwarding from the command-line

* Use the `-R` flag to activate remote port forwarding

`$ ssh -R <remote_ip>:<remote_port>:<local_ip>:<local_port> <user>@<hostname>`

As with local forwarding, ssh attaches to `127.0.0.1` by default, allowing the cimplified form:

`$ ssh -R <remote_ip>:<remote_port>:<local_port> <user>@<hostname>`

#### Simplification using the `ssh_config`

In usage:

* Use the -N flag to state that no command will be executed
* Background using the & keyword

```shell
Host PublicServer
	HostName <host_address>
	User <my_user>
	IdentityFile ~/.ssh/<my_key> # if not id_rsa
	
Host LocalClient
	HostName <host_address>
	User <my_user>
	IdentityFile ~/.ssh/<my_key> # if not id_rsa
	RemoteForward <remote_host>:<remote_port> <local_host>:<local_port>

# Usage (from client)
$ ssh -N PublicServer &
```

### Dynamic port forwarding

Dynamic port forwarding creates a **SOCKS** proxy on the SSH client and tunnels **all inbound network traffic on the proxy port** to the SSH server. Typical uses are as follows:

* To create a generic network gateway to a remote network. E.g. to connect a local and remote network via a gateway (and client software that supports specification of a proxy).

#### Dynamic port forwarding from the command-line

* Use the `-D` flag to activate dynamic port forwarding

`$ ssh -D <local_ip>:<local_port> <user>@<hostname>`

By default SSH attaches to `127.0.0.1`, allowing the simplified form:

`$ ssh -D <local_port> <user>@<hostname>`

#### Simplification using the `ssh_config`

In usage:

* Use the `-N` flag to state that no command will be executed
* Background using the `&` keyword

```shell
Host JumpServer
	HostName <host_address>
	User <my_user>
	IdentityFile ~/.ssh/<my_key> # if not id_rsa
	
Host ProxyServer
	HostName <host_address>
	User <my_user>
	IdentityFile ~/.ssh/<my_key> # if not id_rsa
	DynamicForward <local_host>:<local_port> JumpServer

# Usage (from client)
$ ssh -N PublicServer &
```
