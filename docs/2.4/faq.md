## FAQ

### Can I use Teleport in production today?

Teleport has completed a security audit from a nationally recognized technology security company. 
So we are comfortable with the use of Teleport from a security perspective. However, Teleport 
is still a relatively young product so you may experience usability issues. We are actively 
supporting Teleport and addressing any issues that are submitted to the [github repo](https://github.com/gravitational/teleport).

### Can I connect to nodes behind a firewall?

Yes, Teleport supports reverse SSH tunnels out of the box. To configure behind-firewall clusters
refer to [Trusted Clusters](admin-guide.md#trusted-clusters) section of the Admin Manual.

### Does Web UI support copy and paste?

Yes. You can copy&paste using the mouse. For working with a keyboard, Teleport employs `tmux`-like
"prefix" mode. To enter prefix mode, press `Ctrl+A`.

While in prefix mode, you can press `Ctrl+V` to paste, or enter text selection mode by pressing `[`.
When in text selection mode, move around using `hjkl`, select text by toggling `space` and copy
it via `Ctrl+C`.

### Can I use OpenSSH with a Teleport cluster?

Yes. Take a look at [Using OpenSSH client](user-manual.md##using-teleport-with-openssh) section in the User Manual
and [Using OpenSSH servers](admin-guide.md) in the Admin Manual.

### What TCP ports does Teleport use?

[Ports](admin-guide.md#ports) section of the Admin Manual covers it.

### Does Teleport support authentication via OAuth, SAML or Active Directory?

Gravitational offers this feature as part of the commercial version for Teleport called
[Teleport Enterprise](enterprise.md#rbac)

## Commercial Teleport Editions


### What is a commercial edition of Teleport?

In addition to the [numerous advanced features](enterprise.md), the commercial Teleport license 
also gives users the following:

* Role-based access control, also known as [RBAC](enterprise#rbac)
* Authentication via SAML and OpenID with providers like Okta, Active Directory, Auth0, etc.
* Commercial support.
* Premium SLA with guaranteed response times.

There are two commercial editions of Teleport: 

* **Teleport Pro** is for start-ups and small companies with up to 100 serers.
  Users can sign up for Teleport Pro subscription [on our web site](https://gravitational.com/teleport/).
  Teleport Pro sends the anonymized usage data to Gravitational (see below).
  You can cancel your Teleport Pro subscription any time.

* **Teleport Enterprise** works best for larger companies with 100+ servers and
  comes with substantial volume discounts. Teleport Enterprise does not send
  any usage data to Gravitaitonal and requires an annual contract.

We also offer implementation Services, when our team can help you integrate
Teleport with your existing systems and processes.

### Does Teleport send any data to Gravitational?

The open source edition of Teleport and Teleport Enterprise do not send any information
to Gravitational and can be used on servers without internet access. _Teleport Pro_, our
entry level commercial edition, sends the following anonymized information to
Gravitational on every login event, which contains:

* Anonymized user ID: SHA256 hash of a username with a randomly generated prefix.
* Anonymized server ID: SHA256 hash of a server IP with a randomly generated prefix.
* Anonymized cluster ID: SHA256 hash of a cluster name with a randomly generated prefix.

This allows Teleport Pro to print a warning if users are exceeding the usage limits
encoded in their license. The reporting library code is [on Github](https://github.com/gravitational/reporting).

### Will Teleport stop working if my license expires?

No. Teleport never stops working even if you exceed usage limits as set in the
license.  Teleport will print a warning and will continue to work as usual.

Reach out to `sales@gravitational.com` if you have questions about commercial
edition of Teleport.

