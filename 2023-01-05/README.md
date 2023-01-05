# 1. Practical usage of interface 'type' and the association to IETF -> IANA types

## Summary

Since the initial publish of the `openconfig-interfaces` modeling, the
following mandatory nodes have existed

```
module: openconfig-interfaces
  +--rw interfaces
     +--rw interface* [name]
        +--rw config
        |  +--rw type    identityref
        +--ro state
           +--ro type    identityref
```

Which is defined as the following 2-stage identityref back to `iana-if-type`

```
leaf type {
  type identityref {
    base ietf-if:interface-type;
  }
  mandatory true;
  description
    "The type of the interface.

    When an interface entry is created, a server MAY
    initialize the type leaf with a valid value, e.g., if it
    is possible to derive the type from the name of the
    interface.

    If a client tries to set the type of an interface to a
    value that can never be used by the system, e.g., if the
    type is not supported or if the type does not match the
    name of the interface, the server MUST reject the request.
    A NETCONF server MUST reply with an rpc-error with the
    error-tag 'invalid-value' in this case.";
  reference
    "RFC 2863: The Interfaces Group MIB - ifType";
}
```

For a number of years, there has been initiative to remove external
dependencies from OpenConfig modeling.  While on one aspect, leveraging others
work for reuse is good practice, however it also creates dependencies which can
prove difficult to control over time (ref: Redefinition of relevant IETF types
modules) especially if the intent of the 2 projects diverge.

The `iana-if-type` module version `2017-01-19` currently defines 285 possible
identities for use while OpenConfig related use-cases only have necessity for
around 5 true interface "types".  Deviations cannot convey which identities are
supported/not-supported thus each implementation needs to implement custom
guardrails to start that can only be signaled out-of-band or via error
handling to the consumer/client.

Aside from the vast list of potential types, implementations generally have not
supported specifying the "type" as a r/w object but rather the interface
nomenclature drives the _type_ followed by potential encapsulations where
applicable. (You cannot for instance dictate that an interface named `et-0/0/0`
can be of type `softwareLoopback`)

Since the model node is marked as `mandatory` means the payload _MUST_ include
this leaf when provisioning an interface (which in the case of physical
interfaces are often already provisioned at time of hardware initialization).
In most cases, what you will find is that this node does not translate to
anything meaningful within the implementation from a configuration perspective
(e.g. no-op).  Essentially these cases create what one would consider a bad
API.  You are giving the perception that you instructed the server to do
something that it is blindly accepting and the client is re-iterating for no
good reason.

There are a number of cases of leaf nodes within the OpenConfig model sets that
result in no-ops for various implementations but when tied to a leaf node that
is mandatory as such means that a client must define _something_ and that a
server will likely do nothing with that value calls into question the existance
and purpose in the first case.

For r/o objects, this can still be an attractive attribute to understand which
interfaces are of which type dictated by the system but under most
circumstances this is an object that is not editable.

# 2. tunnel/routed-vlan interface structures and design divergence implications

## Summary

Currently, `openconfig-interfaces` presents 2 design patterns from a
hierarchical perspective in relation to the existance of where L2 and L3
containers and attributes exist.

The primary design pattern is that for _most_ interfaces, there is a hierarchy
of `interface->subinterface` - L2 related attributes are generally covered at
the interface while L3 related at the subinterface levels.

```
module: openconfig-interfaces
  +--rw interfaces
     +--rw interface* [name]
        +--rw subinterfaces
           +--rw subinterface* [index]
              +--rw oc-ip:ipv4
                 +--rw oc-ip:addresses
                 |  +--rw oc-ip:address* [ip]
                 |     +--rw oc-ip:ip        -> ../config/ip
                 |     +--rw oc-ip:config
                 |     |  +--rw oc-ip:ip?              oc-inet:ipv4-address
                 |     |  +--rw oc-ip:prefix-length?   uint8
                 |     +--ro oc-ip:state
                 |     |  +--ro oc-ip:ip?              oc-inet:ipv4-address
                 |     |  +--ro oc-ip:prefix-length?   uint8
                 |     |  +--ro oc-ip:origin?          ip-address-origin
                 ...
```

It is also worth noting that various other models have a concept to reference
back to this concept breaking apart references to the interface + subinterface
pair.

e.g.

```
module: openconfig-network-instance
  +--rw network-instances
     +--rw network-instance* [name]
        +--rw interfaces
           +--rw interface* [id]
              +--rw id        -> ../config/id
              +--rw config
              |  +--rw id?                            string
              |  +--rw interface?                     -> /oc-if:interfaces/interface/name
              |  +--rw subinterface?                  -> /oc-if:interfaces/interface[oc-if:name=current()/../interface]/subinterfaces/subinterface/index
```

However for `routed-vlan` and `tunnel` interfaces, the design pattern changes
where there is no concept of "subinterfaces".  All attributes sit at the root
of an interface.

e.g. `routed-vlan` interface

```
module: openconfig-interfaces
  +--rw interfaces
     +--rw interface* [name]
        +--rw oc-vlan:routed-vlan
           +--rw oc-ip:ipv4
              +--rw oc-ip:addresses
              |  +--rw oc-ip:address* [ip]
              |     +--rw oc-ip:ip        -> ../config/ip
              |     +--rw oc-ip:config
              |     |  +--rw oc-ip:ip?              oc-inet:ipv4-address
              |     |  +--rw oc-ip:prefix-length?   uint8
              |     +--ro oc-ip:state
              |     |  +--ro oc-ip:ip?              oc-inet:ipv4-address
              |     |  +--ro oc-ip:prefix-length?   uint8
              |     |  +--ro oc-ip:origin?          ip-address-origin
              ...
```

e.g. `tunnel` interface
```
module: openconfig-interfaces
  +--rw interfaces
     +--rw interface* [name]
        +--rw oc-tun:tunnel
           +--rw oc-tun:ipv4
              +--rw oc-tun:addresses
              |  +--rw oc-tun:address* [ip]
              |     +--rw oc-tun:ip        -> ../config/ip
              |     +--rw oc-tun:config
              |     |  +--rw oc-tun:ip?              oc-inet:ipv4-address
              |     |  +--rw oc-tun:prefix-length?   uint8
              |     +--ro oc-tun:state
              |        +--ro oc-tun:ip?              oc-inet:ipv4-address
              |        +--ro oc-tun:prefix-length?   uint8
              |        +--ro oc-tun:origin?          ip-address-origin
              ...
```

Now only _some_ interfaces have a different design hierarchy.  It is not a
physical vs. logical split since the normal design pattern also accomodates
logical interfaces (e.g. loopbacks)

The schema itself also allows for the full hierarchy in either circumstance
since there is no gating by way of YANG language statements.  What this really
means is that an implementation needs to prevent (cannot deviate effectively)
subinterface configuration for the `routed-vlan` and `tunnel` variants.  In
theory the model gives access to configuration of an IP address under both
hierarchies for these interfaces.

Effectively we have inconsistent hybrid design patterns likely being driven by
separate implementations at some point in time.  What this creates is then a
proprietary encoding scheme that some implementations need to come up w/ for
interface names (e.g. concatenating interface.subinterface) for portions of the
model which obviously conflict with the true underlying representation and thus
difficulties when referencing from other domains (native schema), logs, CLI,
etc..

## Impact on path targeting

For any API operations that target a specific path for interesting data (e.g.
Give me all interfaces with IPv4 or IPv6 addresses), the divergence in model
design for only _some_ interfaces means that a client cannot target a single
specific path

e.g.

```
/interfaces/interface[name=*]/subinterfaces/subinterface/ipv4/addresses
```

Since the path hierarchy is different, this means a client needs to target
multiple paths for consumption or programmability of like objects.
