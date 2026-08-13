# Unnumbered BGP Peering

A `BGPPeer` normally identifies its neighbor by an IPv4 address. In an **unnumbered**
(interface-based) peering, the neighbor is identified by the *interface* the session runs
over instead. Both routers discover each other through the IPv6 link-local addresses that
they announce in Router Advertisements (RAs) on that link, and both IPv4 and IPv6 routes
are then exchanged with an IPv6 link-local next hop.

This removes the need to plan and manage a /31 or /127 per fabric link, which is why it is
the common design for IPv6 leaf/spine underlays, popularized by FRR and Cumulus Linux.

## Modeling an unnumbered peering

An unnumbered peering consists of two resources: an `Interface` prepared for link-local
peering, and a `BGPPeer` that references it.

```yaml
apiVersion: networking.metal.ironcore.dev/v1alpha1
kind: Interface
metadata:
  name: leaf-01-eth1-1
spec:
  deviceRef:
    name: leaf-01
  name: Ethernet1/1
  type: Physical
  adminState: Up
  ipv4:
    # Route IPv4 traffic over the link although it carries no IPv4 address.
    forwarding: true
  ipv6:
    # Use the automatically generated IPv6 link-local address only.
    linkLocalOnly: true
    routerAdvertisement:
      # Peers discover each other through Router Advertisements.
      enabled: true
---
apiVersion: networking.metal.ironcore.dev/v1alpha1
kind: BGPPeer
metadata:
  name: leaf-01-to-spine-01
spec:
  deviceRef:
    name: leaf-01
  bgpRef:
    name: leaf-01-bgp
  # Peer over the interface instead of an address.
  interfaceRef:
    name: leaf-01-eth1-1
  # Accept any AS number that differs from the local one.
  asNumber: external
  addressFamilies:
    ipv4Unicast:
      enabled: true
    ipv6Unicast:
      enabled: true
```

`spec.address` and `spec.interfaceRef` are mutually exclusive: exactly one of both must be
specified. The referenced `Interface` must belong to the same `Device` as the peer.

Because an unnumbered session sources its packets from the peering interface itself,
`spec.localAddress` must not be set on an interface-based peer.

## Dynamic AS numbers

Setting `spec.asNumber` to the string `external` establishes the session with any AS number
that differs from the local one (`remote-as external`), so that leaves do not need to know
the AS number of every spine up front. A fixed `asNumber` can still be configured for an
interface-based peer if the remote AS number is known.

`external` is only valid together with `spec.interfaceRef`.

## Router Advertisements

Router Advertisements are what makes the peer's link-local address discoverable, so a peering
interface that does not send them never establishes a session. Devices differ in their
default: Cisco NX-OS, for example, suppresses RAs, which is why `spec.ipv6.routerAdvertisement`
has to be set explicitly. When the field is omitted, the operator leaves the device default
in place.

The BGPPeer controller emits a warning event on the `BGPPeer` when the referenced interface
is not prepared for unnumbered peering, i.e. when link-local addressing or Router
Advertisements are missing.

## Provider support

| Provider          | Unnumbered peering |
| ----------------- | ------------------ |
| `cisco-nxos-gnmi` | supported          |
| `cisco-iosxr`     | not supported      |

On Cisco NX-OS the peer is realized as an interface neighbor
(`System/bgp-items/inst-items/dom-items/Dom-list/peerif-items`), which corresponds to the
following CLI configuration:

```
interface Ethernet1/1
  no switchport
  ipv6 address use-link-local-only
  ip forward
  no ipv6 nd suppress-ra
router bgp 65010
  neighbor Ethernet1/1
    remote-as external
    address-family ipv4 unicast
    address-family ipv6 unicast
```
