# User Manual

This User Manual covers usage of the Teleport client tool `tsh`. In this 
document you will learn how to:

* Log into interactive shell on remote cluster nodes.
* Copy files to and from cluster nodes.
* Connect to SSH clusters behind firewalls without any open ports using SSH reverse tunnels.
* Explore a cluster and execute commands on those nodes in a cluster that match your criteria.
* Share interactive shell sessions with colleagues or join someone else's session.
* Replay recorded interactive sessions.

In addition to this document, you can always simply type `tsh` into your terminal for
the CLI reference.

```bash
$ tsh
usage: tsh [<flags>] <command> [<command-args> ...]

Gravitational Teleport SSH tool

Commands:
  help         Show help.
  version      Print the version
  ssh          Run shell or execute a command on a remote SSH node
  join         Join the active SSH session
  play         Replay the recorded SSH session
  scp          Secure file copy
  ls           List remote SSH nodes
  clusters     List available Teleport clusters
  login        Log in to the cluster and store the session certificate to avoid login prompts
  logout       Delete a cluster certificate

# Run `tsh help <command>` to get help for <command> like `tsh help ssh`
```

## User Identities

A user identity in Teleport exists in the scope of a cluster. The member nodes
of a cluster may have multiple OS users on them. A Teleport administrator assigns
allowed logins to every Teleport user account.

When logging into a remote node, you will have to specify both logins. Teleport
identity will have to be passed as `--user` flag, while the node login will be
passed as `login@host`, using syntax compatible with traditional `ssh`.

These examples assume your localhost username is 'joe':

```bash
# Authenticate against cluster 'work' as 'joe' and then login into 'node'
# as root:
$ tsh ssh --proxy=work.example.com --user=joe root@node

# Authenticate against cluster 'work' as 'joe' and then login into 'node'
# as joe (by default tsh uses $USER for both):
$ tsh ssh --proxy=work.example.com node
```

`tsh login` log a user into a cluster and stores the user certificate 
in `~/.tsh` directory as well as in the [ssh agent](https://en.wikipedia.org/wiki/Ssh-agent), 
if there is one running:

```
$ tsh login --proxy=work.example.com
```

This allows you authenticate just once, maybe at the beginning of the day. 
Subsequent `tsh ssh` commands will run without asking for credentials
until the temporary certificate expires. By default Teleport issues user
certificates with a TTL (time to live) of 23 hours.


!!! tip "Tip": 
    It is recommended to always use `tsh login` before using any other `tsh` commands.
    This allows users to omit `--proxy` flag in subsequent tsh commands. For example 
    `tsh ssh user@host` will work.

### SSH Agent Support

If there is an [ssh agent](https://en.wikipedia.org/wiki/Ssh-agent) running, 
`tsh login` will store the user certificate in the agent. 

This can be verified via:

```bash
$ ssh-add -L
```

SSH agent can be used to feed the certificate to other SSH clients, for example
to OpenSSH `ssh`.

### Identity Files

`tsh login` can also save the user certificate into a file:

```bash
# Authenticate user against proxy.example.com and save the user 
# certificate into joe.pem file
$ tsh login --proxy=proxy.example.com --out=joe

# Use joe.pem to login into a server 'db'
$ tsh ssh --proxy=proxy.example.com -i joe joe@db
```

By default `--out` flag will create an identity file suitable for `tsh -i` but
if compatibility with OpenSSH is needed, `--format=openssh` must be specified. 
In this case the identity will be saved into two files: `joe` and `joe-cert.pub`:

```bash
$ tsh login --proxy=proxy.example.com --out=joe --format=openssh
$ ls -lh
total 8.0K
-rw------- 1 joe staff 1.7K Aug 10 16:16 joe
-rw------- 1 joe staff 1.5K Aug 10 16:16 joe-cert.pub
```

### SSH Certificates for Automation

Regular users of Teleport must request an auto-expiring SSH certificate,
usually every day. This doesn't work for non-intractive scripts, like cron jobs
or CI/CD pipeline.

For such automation, it is recommended to create a separate Teleport user for
bots and request a certificate for them with a long time to live (TTL).

In this example we're creating a certificate with a TTL of 10 years for the
jenkins user and storing it in jenkins.pem file, which can be later used with
`-i` (identity) flag for `tsh`.

```bash
# to be executed on a Teleport auth server
$ tctl auth sign --ttl=87600h --user=jenkins --out=jenkins.pem
```

Now `jenkins.pem` can be copied to the jenkins server and passed to `-i`
(identity file) flag of `tsh`. Essentially `tctl auth sign` is an admin's
equivalent of `tsh login --out` and allows for unrestricted certificate TTL
values.

## Exploring the Cluster

In a Teleport cluster all nodes periodically ping the cluster's auth server and
update their status. This allows Teleport users to see which nodes are online with the `tsh ls` command:

```bash
# This command lists all nodes in the cluster which you previously logged in via "tsh login":
$ tsh ls

# Output:
Node Name     Node ID                Address            Labels
---------     -------                -------            ------
turing        11111111-dddd-4132     10.1.0.5:3022     os:linux
turing        22222222-cccc-8274     10.1.0.6:3022     os:linux
graviton      33333333-aaaa-1284     10.1.0.7:3022     os:osx
```

`tsh ls` can apply a filter based on the node labels. 

```bash
# only show nodes with os label set to 'osx':
$ tsh ls os=osx

Node Name     Node ID                Address            Labels
---------     -------                -------            ------
graviton      33333333-aaaa-1284     10.1.0.7:3022     os:osx
```

## Interactive Shell

To launch an interactive shell on a remote node or to execute a command, use `tsh ssh` 
command:

```bash
$ tsh ssh --help

usage: t ssh [<flags>] <[user@]host> [<command>...]
Run shell or execute a command on a remote SSH node.

Flags:
      --user      SSH proxy user [alice]
      --proxy     SSH proxy host or IP address, for example --proxy=host:ssh_port,https_port
      --ttl       Minutes to live for a SSH session 
      --insecure  Do not verify server certificate and host name. Use only in test environments
  -d, --debug     Verbose logging to stdout
  -p, --port      SSH port on a remote host
  -l, --login     Remote host login
  -L, --forward   Forward localhost connections to remote server
      --local     Execute command on localhost after connecting to SSH node

Args:
  <[user@]host>  Remote hostname and the login to use
  [<command>]    Command to execute on a remote host
```

`tsh` tries to mimic the `ssh` experience as much as possible, so it supports the most popular `ssh`
flags like `-p`, `-l` or `-L`. For example if you have the following alias defined in your 
`~/.bashrc`: `alias ssh="tsh --proxy=work.example.com --user=myname"` then you can continue
using familiar SSH syntax:

```bash
$ ssh root@host
$ ssh -p 6122 root@host ls
```

### Proxy Ports

A Teleport proxy uses two ports: `3080` for HTTPS and `3023` for proxying SSH connections.
The HTTPS port is used to serve Web UI and also to implement 2nd factor auth for `tsh` client.

If a Teleport proxy is configured to listen on non-default ports, they must be specified
via `--proxy` flag as shown:

```
tsh --proxy=proxy.example.com:5000,5001
```

This means _use port `5000` for HTTPS and `5001` for SSH_

### Port Forwarding

`tsh ssh` supports OpenSSH `-L` flag which allows forwarding incoming connections from localhost
to the specified remote host:port. The syntax of `-L` flag is:

```bash
-L [bind_ip]:listen_port:remote_host:remote_port
```

where "bind_ip" defaults to `127.0.0.1`.

Example:

```bash
$ tsh ssh -L 5000:web.remote:80 node
```

Will connect to remote server `node` via `proxy.example.com`, then it will open
a listening socket on `localhost:5000` and will forward all incoming connections 
to `web.remote:80` via this SSH tunnel.

It is often convenient to establish port forwarding, execute a local command which uses such 
connection and disconnect. You can do this via `--local` flag.

Example:

```bash
$ tsh ssh -L 5000:google.com:80 --local node curl http://localhost:5000
```

This command:

1. Connects to `node`
2. Binds the local port 5000 to port 80 on google.com
3. Executes `curl` command locally, which results in `curl` hitting google.com:80 via `node`

### Resolving Node Names

`tsh` supports multiple methods to resolve remote node names. 

1. Traditional: by IP address or via DNS.
2. Nodename setting: teleport daemon supports `nodename` flag, which allows Teleport administrators to assign alternative node names.
3. Labels: you can address a node by `name=value` pair.

In the example above, we have two nodes with `os:linux` label and one node with `os:osx`.
Lets login into the OSX node:

```bash
$ tsh ssh os=osx
```

This only works if there is only one remote node with `os:osx` label, but you can still execute
commands via SSH on multiple nodes using labels as a selector. This command will update all
system packages on machines that run Linux:

```bash
$ tsh ssh os=ubuntu apt-get update -y
```

### Short-lived Sessions

Suppose you are borrowing someone else's computer to login into a cluster. You probably don't 
want to stay authenticated on this computer for 23 hours (Teleport default). This is where the `--ttl`
flag can help.

This command logs you into the cluster with a very short-lived (1 minute) temporary certificate:

```bash
$ tsh --ttl=1 login
```

You will be logged out after one minute, but if you want to log out immediately, you can 
always do:

```bash
$ tsh logout
```

## Copying Files

To securely copy files to and from cluster nodes use `tsh scp` command. It is designed to mimic
traditional `scp` as much as possible:

```bash
$ tsh scp --help

usage: tsh scp [<flags>] <from, to>...
Secure file copy

Flags:
      --user       SSH proxy user [alice]
      --proxy      SSH proxy host or IP address
      --ttl        Minutes to live for a SSH session
      --insecure   Do not verify server certificate and host name. Use only in test environments
  -P, --debug      Verbose logging to stdout
  -d, --debug      Verbose logging to stdout
  -r, --recursive  Recursive copy of subdirectories

Args:
  <from, to>       Source and the destination
```

Examples:

```bash
$ tsh scp example.txt root@node:/path/to/dest
```

Again, you may want to create a bash alias like `alias scp="tsh --proxy=work scp"` and use
the familiar sytanx:

```bash
$ scp -P 61122 -r files root@node:/path/to/dest
```

## Sharing Sessions

Suppose you are trying to troubleshoot a problem on a remote server. Sometimes it makes sense 
to ask another team member for help. Traditionally this could be done by letting them know which 
node you're on, having them SSH in, start a terminal multiplexer like `screen` and join a 
session there.

Teleport makes this a bit more convenient. Let's log into "luna" and ask Teleport for your 
current session status:

```bash
$ tsh ssh luna
>luna $ teleport status

User ID    : joe, logged in as joe from 10.0.10.1 43026 3022
Session ID : 7645d523-60cb-436d-b732-99c5df14b7c4
Session URL: https://work:3080/web/sessions/7645d523-60cb-436d-b732-99c5df14b7c4
```

Now you can invite another user account in the "work" cluster. You can share the URL for access through a web browser. 
Or you can share the session ID and she can join you through her terminal by typing:

```bash
$ tsh join 7645d523-60cb-436d-b732-99c5df14b7c4
```

## Connecting to SSH Clusters behind Firewalls

Teleport supports creating clusters of servers located behind firewalls **without
any open listening TCP ports**.  This works by creating reverse SSH tunnels from
behind-firewall environments into a Teleport proxy you have access to. This
feature is called "Trusted Clusters". 

This chapter explains how to a user may connect to a trusted cluster. 
Refer to [the admin manual](admin-guide/#trusted-clusters) to learn how a trusted 
cluster can be configured.

Assuming the "work" Teleport proxy server is configured with a few trusted
clusters, a user may use `tsh clusters` command to see a list of them:

```bash
$ tsh --proxy=work clusters

Cluster Name     Status
------------     ------
staging          online
production       offline
```

Now you can use `--cluster` flag with any `tsh` command. For example, to list SSH nodes that
are members of "production" cluster, simply do:

```bash
$ tsh --proxy=work --cluster=production ls
Node Name     Node ID       Address            Labels
---------     -------       -------            ------
db-1          xxxxxxxxx     10.0.20.31:3022    kernel:4.4
db-2          xxxxxxxxx     10.0.20.41:3022    kernel:4.2
```

Similarly, if you want to SSH into `db-1` inside "production" cluster:

```bash
$ tsh --proxy=work --cluster=production ssh db-1
```

This is possible even if nodes of the "production" cluster are located behind a firewall
without open ports. This works because "production" cluster establishes a reverse
SSH tunnel back into "work" proxy, and this tunnels is used to establish inbound SSH
connections.

## Web UI

Teleport proxy serves the web UI on `https://proxyhost:3080`. The UI allows you to see the list of 
online nodes in a cluster, open web-based Terminal to them, see recorded sessions and replay them.
You can also join other users via active sessions.

You can copy & paste using the mouse. For working with a keyboard, Teleport employs `tmux`-like
"prefix" mode. To enter prefix mode, press `Ctrl+A`.

While in prefix mode, you can press `Ctrl+V` to paste, or enter text selection mode by pressing `[`.
When in text selection mode, move around using `hjkl`, select text by toggling `space` and copy
it via `Ctrl+C`.


## Using OpenSSH Client

There are a few differences between Teleport's `tsh` and OpenSSH's `ssh` but most
of them can be made completely invisible.

1. `tsh` always requires `--proxy` flag because `tsh` needs to know which cluster
    you are connecting to. But if you execute `tsh --proxy=xxx login`, the
    current proxy will be saved in your `~/.tsh` profile and won't be needed
    for other `tsh` commands.

* `tsh ssh` operates _two_ usernames: one for the cluster and another for the node you
  are trying to login into. See [User Identities](#user-identities) section below. 
  For convenience, `tsh` assumes `$USER` for both by default. But again, if you
  use `tsh login` before `tsh ssh`, your Teleport username will be stored in
  `~/.tsh`

!!! tip "Tip": 
    To avoid typing `tsh ssh user@host` when loggin inton servers, you can
    create a symlink `ssh -> tsh` and execute the symlink. It will behave exactly 
    like a standard `ssh` command, i.e. `ssh login@host`. This is helpful with other 
    tools that expect `ssh` to just work.

Teleport is built using standard SSH constructs: keys, certificates, protocols.
This means that a Teleport system is 100% compatible with both OpenSSH clients and servers.

For OpenSSH client (`ssh`) to work with a Teleport proxy, two conditions must be met:

1. `ssh` must be configured to connect via a Teleport proxy.
2. `ssh` needs to be given the SSH certificate issued by `tsh login` command.

### SSH Proxy Configuration

To configure `ssh` to use a Teleport proxy on `proxy.example.com`, a user must 
update the `/etc/ssh/ssh_config` or `~/.ssh/config`. A few examples are shown below:

```bash
# When "ssh db" is executed, OpenSSH will connect to proxy.example.com on port 3023
# and will request a proxied connection to "db" on port 3022 (default Teleport SSH port)
Host db
    Port 3022
    ProxyCommand ssh -p 3023 %r@proxy.example.com -s proxy:%h:%p

# When connecting to a node behind a trusted cluster named "remote-cluster",
# the name of the trusted cluster must be appended to the proxy subsystem 
# after '@':
Host *.trusted-cluster.example.com
   Port 3022
   ProxyCommand ssh -p 3023 %r@proxy.example.com -s proxy:%h:%p@trusted-cluster
```

The configuration above is all one will need to `ssh root@db` if there's an
SSH agent running on a client computer. You can verify it by executing `ssh-add -L` 
right after `tsh login`. If the SSH agent is running the cluster certificates will 
be printed to stdout.

If there is no ssh-agent available, the certificate must be passed to the
OpenSSH client explicitly.

### Passing Teleport SSH Certificate to OpenSSH Client

If a user does not want to use an SSH agent or if the agent is not available, the certificate
must be passed to `ssh` via `IdentityFile` option (see `man ssh_config`). Consider this example: 
the Teleport user "joe" wants to login into the proxy named "lab.example.com". 
He executes `tsh login` command:

```bash
$ tsh --proxy=lab.example.com login --user=joe
```

His identity is now stored in `~/.tsh/keys/lab.example.com`, so his `~/.ssh/config` needs to look like this:

```bash
# ~/.ssh/config file:
Host *.lab.example.com
    Port 3022
    IdentityFile ~/.tsh/keys/lab.example.com/joe
    ProxyCommand ssh -i ~/.tsh/keys/lab.example.com/joe -p 3023 %r@lab.example.com -s proxy:%h:%p
```

Now he can SSH into any machine behind `lab.example.com` using the OpenSSH client:

```bash
$ ssh jenkins.lab.example.com
```


## Troubleshooting

If you encounter strange behaviour, you may want to try to solve it by enabling
the verbose logging by specifying `-d` flag when launching `tsh`.

Also you may want to reset it to a clean state by deleting temporary keys and 
other data from `~/.tsh`

## Getting Help

Please open an [issue on Github](https://github.com/gravitational/teleport/issues).
Alternatively, you can reach through the contact form on our [website](https://gravitational.com/).

For commercial support, custom features or to try our [Enterprise edition of Teleport](/enterprise/),
please reach out to us: `sales@gravitational.com`.  
