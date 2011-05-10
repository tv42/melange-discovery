======================================================
 OpenStack Melange DHCP/VM configuration architecture
======================================================

:Author: Tommi Virtanen <tommi.virtanen@dreamhost.com>

I'm calling this component ``melange-discovery``, for now.

Definitions
===========

discovery packet
  Any network packet related to network address discovery for a VM,
  including DHCP, IPv6 Router Advertisement, DHCPv6, etc.

Requirements
============

discovery/dhcp
--------------

VMs MUST be able to configure their networking with DHCP (IPv4)

discovery/ipv6-radv
-------------------

VMs MUST be able to configure their networking with IPv6 Router
Advertisements.

discovery/dhcpv6
----------------

VMs MUST/SHOULD(TODO which) be able to configure their networking with
DHCPv6.

TODO: This is still unclear, I don't have extensive hands-on
experience with IPv6. Need DNS resolver from somewhere, what's best
current practice?

discovery/custom
----------------

VMs SHOULD be able to configure their networking via a custom
mechanism. This implies being able to fetch the configuration to use,
somehow.

IDEA: use a virtio 9p filesystem to provide the information.

secure/no-mitm
--------------

VMs MUST NOT be vulnerable to MITM attacks on DHCP.

This is a special case of the generic "no ethernet-level MITM"
requirement. TODO find a link.

secure/egress-filter
--------------------

VMs MUST NOT be able to send packets with an IP address they are not
allowed to use.

If a lease on an IP address expires, the VM MUST NOT be able to
continue using that address.

secure/origin
-------------

DHCP requests from VMs MUST be non-ambiguously attributed to the
originating VM.

floating-ip
-----------

The system MUST be able to use migrate an IP address from one VM to
another. The migration MUST complete in less than 1 second.

TODO: given what limitations, e.g. same L2 broadcast zone?  how is
this different from just explicit DHCP lease termination and a lease
taken up elsewhere? what component triggers the migration? can this
level treat this as lease expiry & new lease elsewhere? break into
multiple requirements! OR just say you must use an LB.

TODO: ping to ensure IP is free before using, like legacy DHCP does?
IPAM is supposed to be absolute truth, though.


High-level Design
=================

Every Nova host machine runs a `Melange discovery service` (TODO
actual code may belong in the Nova project). This replaces the current
``dnsmasq`` mechanism in setups where Melange is used.

Outgoing `discovery packets` from Nova instances are passed to the
discovery service.

The discovery service consults the Melange IPAM service, with
information on what Nova instance the request originates from. Melange
may at that time e.g. allocate a new IP address from a pool, or just
returned the statically configured IP address.

The Melange IPAM response is used to both provide a response to the
VM, and to configure egress filtering. IP address lease information is
recorded locally into non-volatile storage.

Existing leases are kept track of, and upon expiry, egress filtering
is configured appropriately.


Proposed implementation
=======================

Discovery service
-----------------

Needs to receive discovery packets from all VMs, and inject response
packets to a given VM.

Receiving discovery packets
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Assumption: VM networking is provided via a kernel network interface,
and not a purely userspace API. (If not true, userspace API must
provide receive, send, and egress filtering functionality.)

Receiving could be done using a raw listening socket. Requirement
`secure/no-mitm`_ means packets that would be received by the raw
socket are also subject to netfilter rules. I am personally wary of
mixing the two, nothing seems to guarantee which one gets the packets
first.

Discovery packets can be handed off to desired userspace process
reliably, even in the presence of netfilter ``DROP`` rules, by using
the ``NFQUEUE`` netfilter target. The userspace process can easily
read the packets matched by netfilter, with metadata such as what
interface it was sent from. Python bindings are available from the
`nfqueue-bindings` project at
http://www.nufw.org/projects/nfqueue-bindings/wiki/Wiki .

TODO: data source need: mapping ``ifindex`` -> VM identity.


Processing discovery packets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The discovery service can process the discovery packets using Scapy
(see http://www.secdev.org/projects/scapy/ ), a Python library for
low-level TCP/IP packet manipulation. For an example of a very similar
implementation, see the Clusto project's (http://clusto.org/ ) DHCP
server, at
https://github.com/clusto/clusto/blob/master/src/clusto/services/dhcp.py

At this stage, the discovery service will typically need to do a
single-roundtrip request-response communication to the Melange IPAM
service. This protocol will be discussed in `IPAM protocol`_.

At this stage, the discovery service will also typically need to do a
single-roundtrip request-response communication to the component
responsible for egress filtering of VM traffic. This protocol will be
discussed in `Firewall protocol`_.


Sending a response packet
~~~~~~~~~~~~~~~~~~~~~~~~~

Assumption: VM networking is provided via a kernel network interface,
and not a purely userspace API. (If not true, userspace API must
provide receive, send, and egress filtering functionality.)

Sending responses to the VM can be done via raw sockets. Scapy's
``sendp`` method should work nicely.

TODO: data source need: mapping VM identity -> ``ifindex``.


Lease expiration
~~~~~~~~~~~~~~~~

The discovery service keeps an in-memory list of active leases. This
list is loaded from non-volatile storage on startup.

When any lease expires, the following actions are performed, in this
order:

- egress filter is notified to prevent any communication with that
  IP address from the previous lease holder

- Melange IPAM is notified to confirm that the lease has truly
  expired.

If one of the actions fails, it will be retried, with exponential
backoff. The lease is forgotten only after successful completion.

This MUST handle IP address being assigned elsewhere while previous
lease expiry is still in progress, at any stage of the above actions,
even within one host machine.

This MUST handle lease being renewed by the same VM before the
previous expiration was completed, at any stage of the above actions.


IPAM protocol
=============

Request:

- VM identity
- address requested (if any)

Response:

- ok / report failure to VM / internal error
- address
- lease expiry time

TODO

Firewall protocol
=================

Request:

- VM identity
- acceptable egress IP addresses (IPv4, IPv6)
  (TODO link-level etc stuff?)

Overwrites the acceptable address list with the given list.

Response:

- ok / internal error

This should run over a UNIX domain socket? This is about the compute
node itself, no need to ever cross the network.

TODO

Security considerations
=======================

Avoiding running as ``root``
----------------------------

(or with net admin capabilities, which is almost about as bad, just
more obscure)

Send
~~~~

Opening raw sockets requires ``root`` privileges. As VMs can be
created and reconfigured at any time, and raw sockets must be bound to
the VMs network interface, this must be possible at runtime.

This can be solved by having a helper process run as ``root``, with a
unidirectional pipe from the main process to the helper process. The
pipe carries fully formed and ready to send IP packets, including the
outgoing ``ifindex``, encapsulated in e.g. JSON.

The privileged helper process can avoid all protocol parsing.

Receive
~~~~~~~

``NFQUEUE`` requires ``root`` access at least at open
time. ``nfqueue-bindings``, the Python library, seems to require
``root`` access at packet receive time; it is currently unclear
whether this is a limitation of ``nfqueue-bindings`` or netfilter.

In either case, this can be moved into a helper process, with a
unidirectional pipe from the helper to the main process, carrying the
raw packets with metadata, encapsulated in e.g. JSON.

The privileged helper process can avoid all protocol parsing.

(The desired ``NFQUEUE`` *verdict* is here always ``DROP``; this is
what allows the pipe to be unidirectional.)

TODO
====

- ip address, gateway, subnet, dns server, ntp, etc
- authenticating to Melange IPAM
- write more about virtio 9p fs
