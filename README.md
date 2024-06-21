# Inter-AS Option B EVPN/IP-VPN services lab

This example shows EVPN and IPVPN services on Inter-AS option B scenarios.

![](interasoptb.clab.drawio.png)

The diagram shows two different Autonomous Systems (ASes) that are connected via border routers br4, br5 and br6, which are acting as Inter-AS model B ASBRs.
This model allows the operator to extend EVPN and IP-VPN services across different MPLS or Segment Routing MPLS domains, without the need for instantiating services on the border routers. Egress PEs advertise EVPN/IP-VPN routes to the adjacent border routers, and the border routers carry out two functions:

1. Import the routes and, if valid, redistribute the routes to the remote border routers or local PEs, using their own address as next hop and their own locally allocated service mpls labels.
2. Program a label swap operation so that ingress traffic service label is looked up and packets forwarded with a new service label that was previously received from the PE in the local domain.

## Configuration of the default network-instance in PEs and Border Routers

The following output shows the configuration of pe1's default network-instance. ISIS is used as IGP in each of the two domains. Transport tunnels are of type LDP and SR-ISIS. And BGP neighbor br4 is configured for the exchange of EVPN and IP-VPN routes.

<pre>
--{ + candidate shared default }--[ network-instance default ]--
A:pe1# info
    type default
    interface ethernet-1/10.0 {
    }
    interface system0.0 {
    }
    protocols {
        bgp {
            admin-state enable
            autonomous-system 65001
            router-id 100.0.0.1
            bgp-label {
                bgp-ipvpn {
                    next-hop-resolution {
                        ipv4-next-hops {
                            tunnel-resolution {
                                allowed-tunnel-types [
                                    ldp
                                    sr-isis
                                ]
                            }
                        }
                    }
                }
            }
            ebgp-default-policy {
                import-reject-all false
                export-reject-all false
            }
            afi-safi evpn {
                admin-state enable
                evpn {
                    keep-all-routes true
                    rapid-update true
                }
            }
            afi-safi l3vpn-ipv4-unicast {
                admin-state enable
                l3vpn-ipv4-unicast {
                    keep-all-routes true
                    rapid-update true
                }
            }
            group overlay-ibgp {
                peer-as 65001
                afi-safi evpn {
                }
                afi-safi l3vpn-ipv4-unicast {
                }
                timers {
                    connect-retry 1
                    minimum-advertisement-interval 1
                }
                trace-options {
                    flag update {
                        modifier detail
                    }
                }
            }
            neighbor 100.0.0.4 {
                peer-group overlay-ibgp
            }
        }
        ldp {
            admin-state enable
            dynamic-label-block range-1-ldp
            discovery {
                interfaces {
                    interface ethernet-1/10.0 {
                        ipv4 {
                            admin-state enable
                        }
                    }
                }
            }
        }
        isis {
            dynamic-label-block range-3-srgb
            instance i14 {
                admin-state enable
                instance-id 1
                level-capability L2
                iid-tlv true
                net [
                    49.0001.0000.0000.0001.00
                ]
                trace-options {
                    trace [
                        adjacencies
                        interfaces
                        packets-all
                    ]
                }
                segment-routing {
                    mpls {
                        dynamic-adjacency-sids {
                            all-interfaces true
                        }
                    }
                }
                interface ethernet-1/10.0 {
                    circuit-type point-to-point
                    ipv4-unicast {
                        admin-state enable
                    }
                }
                interface system0.0 {
                    passive true
                    ipv4-unicast {
                        admin-state enable
                    }
                }
            }
        }
    }
    segment-routing {
        mpls {
            global-block {
                label-range range-2-srgb
            }
            local-prefix-sid 1 {
                interface system0.0
                ipv4-label-index 1
            }
        }
    }
--{ + candidate shared default }--[ network-instance default ]--
</pre>

Similar configurations exist on the other PEs, with the exception that pe2 is configured as a route reflector on its own domain. The configuration of the default network-instance on pe2 follows.

<pre>
--{ + candidate shared default }--[ network-instance default protocols bgp ]--
A:pe2# info
    admin-state enable
    autonomous-system 65023
    router-id 100.0.0.2
    bgp-label {
        bgp-ipvpn {
            next-hop-resolution {
                ipv4-next-hops {
                    tunnel-resolution {
                        allowed-tunnel-types [
                            ldp
                            sr-isis
                        ]
                    }
                }
            }
        }
    }
    ebgp-default-policy {
        import-reject-all false
        export-reject-all false
    }
    afi-safi evpn {
        admin-state enable
        evpn {
            keep-all-routes true
            rapid-update true
        }
    }
    afi-safi l3vpn-ipv4-unicast {
        admin-state enable
        l3vpn-ipv4-unicast {
            keep-all-routes true
            rapid-update true
        }
    }
    group overlay-ibgp {
        peer-as 65023
        afi-safi evpn {
        }
        afi-safi l3vpn-ipv4-unicast {
        }
        route-reflector {
            client true
            cluster-id 2.2.2.2
        }
        timers {
            connect-retry 1
            minimum-advertisement-interval 1
        }
        trace-options {
            flag update {
                modifier detail
            }
        }
    }
    neighbor 100.0.0.3 {
        peer-group overlay-ibgp
    }
    neighbor 100.0.0.5 {
        peer-group overlay-ibgp
    }
    neighbor 100.0.0.6 {
        peer-group overlay-ibgp
    }
</pre>

The border routers are configured in a similar way, only that the IGP is not enabled in their interfaces to other border routers. In addition, border routers configured for inter-as option B, need the following two things:

1. The configuration of the label block that the border router, as ASBR, will use to draw EVPN/IP-VPN labels for the service label swap operation. This is configured using the command `protocols.bgp.bgp-label.bgp-vpn.dynamic-label-block <label-block>`.
2. The configuration of `inter-as-vpn true` command for EVPN and IP-VPN, which triggers the router to keep the received EVPN/IP-VPN routes (even without a network-instance that imports the routes) and redistribute them to the adjacent ASN with a next hop and label change.

As an example, br4's configuration of the default network instance follows:

<pre>
--{ + candidate shared default }--[ network-instance default ]--
A:br4# info
    type default
    interface ethernet-1/10.0 {
    }
    interface ethernet-1/11.0 {
    }
    interface ethernet-1/12.0 {
    }
    interface system0.0 {
    }
    protocols {
        bgp {
            autonomous-system 65001
            router-id 100.0.0.4
            bgp-label {
                bgp-vpn {
                    dynamic-label-block range-6-bgp-lu
                }
                bgp-ipvpn {
                    next-hop-resolution {
                        ipv4-next-hops {
                            tunnel-resolution {
                                allowed-tunnel-types [
                                    ldp
                                    sr-isis
                                ]
                            }
                        }
                    }
                }
            }
            ebgp-default-policy {
                import-reject-all false
                export-reject-all false
            }
            afi-safi evpn {
                admin-state enable
                evpn {
                    inter-as-vpn true
                    rapid-update true
                    default-received-encapsulation mpls
                    next-hop-resolution {
                        ipv4-next-hops {
                            tunnel-resolution {
                                allowed-tunnel-types [
                                    bgp
                                    ldp
                                    sr-isis
                                ]
                            }
                        }
                    }
                }
            }
            afi-safi l3vpn-ipv4-unicast {
                admin-state enable
                l3vpn-ipv4-unicast {
                    inter-as-vpn true
                    rapid-update true
                }
            }
            group overlay-ebgp {
                peer-as 65023
                afi-safi evpn {
                }
                timers {
                    connect-retry 1
                    minimum-advertisement-interval 1
                }
                trace-options {
                    flag update {
                        modifier detail
                    }
                }
            }
            group overlay-ibgp {
                peer-as 65001
                afi-safi evpn {
                }
                timers {
                    connect-retry 1
                    minimum-advertisement-interval 1
                }
                trace-options {
                    flag update {
                        modifier detail
                    }
                }
            }
            neighbor 10.4.5.2 {
                peer-group overlay-ebgp
            }
            neighbor 10.4.6.2 {
                peer-group overlay-ebgp
            }
            neighbor 100.0.0.1 {
                peer-group overlay-ibgp
            }
        }
        ldp {
            admin-state enable
            dynamic-label-block range-1-ldp
            discovery {
                interfaces {
                    interface ethernet-1/10.0 {
                        ipv4 {
                            admin-state enable
                        }
                    }
                    interface ethernet-1/11.0 {
                        ipv4 {
                            admin-state enable
                        }
                    }
                    interface ethernet-1/12.0 {
                        ipv4 {
                            admin-state enable
                        }
                    }
                }
            }
        }
        isis {
            dynamic-label-block range-3-srgb
            instance i14 {
                admin-state enable
                instance-id 1
                level-capability L2
                iid-tlv true
                net [
                    49.0001.0000.0000.0004.00
                ]
                segment-routing {
                    mpls {
                        dynamic-adjacency-sids {
                            all-interfaces true
                        }
                    }
                }
                interface ethernet-1/10.0 {
                    circuit-type point-to-point
                    ipv4-unicast {
                        admin-state enable
                    }
                }
                interface system0.0 {
                    passive true
                    ipv4-unicast {
                        admin-state enable
                    }
                }
            }
        }
    }
    segment-routing {
        mpls {
            global-block {
                label-range range-2-srgb
            }
            local-prefix-sid 1 {
                interface system0.0
                ipv4-label-index 4
            }
        }
    }
--{ + candidate shared default }--[ network-instance default ]--
</pre>

And br4's label allocation for transport and services is configured as follows:

<pre>
--{ + candidate shared default }--[ system mpls ]--
A:br4# info
    label-ranges {
        static range-2-srgb {
            shared true
            start-label 100001
            end-label 120000
        }
        static range-5-static-services {
            shared false
            start-label 3000
            end-label 4000
        }
        dynamic range-1-ldp {
            start-label 100
            end-label 200
        }
        dynamic range-3-srgb {
            start-label 120001
            end-label 120999
        }
        dynamic range-4-evpn {
            start-label 500
            end-label 699
        }
        dynamic range-5-services {
            start-label 1000
            end-label 2000
        }
        dynamic range-6-bgp-lu {
            start-label 122001
            end-label 122201
        }
    }
    services {
        evpn {
            dynamic-label-block range-4-evpn
        }
        network-instance {
            dynamic-label-block range-5-services
        }
    }
</pre>

Similar configurations exist in the other border routers. The configuration of br6 follows:

<pre>
--{ + candidate shared default }--[ network-instance default ]--
A:br6# info
    type default
    interface ethernet-1/10.0 {
    }
    interface ethernet-1/11.0 {
    }
    interface ethernet-1/12.0 {
    }
    interface system0.0 {
    }
    protocols {
        bgp {
            autonomous-system 65023
            router-id 100.0.0.6
            bgp-label {
                bgp-vpn {
                    dynamic-label-block range-6-bgp-lu
                }
                bgp-ipvpn {
                    next-hop-resolution {
                        ipv4-next-hops {
                            tunnel-resolution {
                                allowed-tunnel-types [
                                    ldp
                                    sr-isis
                                ]
                            }
                        }
                    }
                }
            }
            ebgp-default-policy {
                import-reject-all false
                export-reject-all false
            }
            afi-safi evpn {
                admin-state enable
                evpn {
                    inter-as-vpn true
                    rapid-update true
                    default-received-encapsulation mpls
                    next-hop-resolution {
                        ipv4-next-hops {
                            tunnel-resolution {
                                allowed-tunnel-types [
                                    bgp
                                    ldp
                                    sr-isis
                                ]
                            }
                        }
                    }
                }
            }
            afi-safi l3vpn-ipv4-unicast {
                admin-state enable
                l3vpn-ipv4-unicast {
                    inter-as-vpn true
                    rapid-update true
                }
            }
            group overlay-ebgp {
                peer-as 65001
                afi-safi evpn {
                }
                timers {
                    connect-retry 1
                    minimum-advertisement-interval 1
                }
                trace-options {
                    flag update {
                        modifier detail
                    }
                }
            }
            group overlay-ibgp {
                peer-as 65023
                afi-safi evpn {
                }
                timers {
                    connect-retry 1
                    minimum-advertisement-interval 1
                }
                trace-options {
                    flag update {
                        modifier detail
                    }
                }
            }
            neighbor 10.4.6.1 {
                peer-group overlay-ebgp
            }
            neighbor 100.0.0.2 {
                peer-group overlay-ibgp
            }
        }
        ldp {
            admin-state enable
            dynamic-label-block range-1-ldp
            discovery {
                interfaces {
                    interface ethernet-1/10.0 {
                        ipv4 {
                            admin-state enable
                        }
                    }
                    interface ethernet-1/11.0 {
                        ipv4 {
                            admin-state enable
                        }
                    }
                    interface ethernet-1/12.0 {
                        ipv4 {
                            admin-state enable
                        }
                    }
                }
            }
        }
        isis {
            dynamic-label-block range-3-srgb
            instance i2356 {
                admin-state enable
                instance-id 3
                level-capability L2
                max-ecmp-paths 64
                iid-tlv true
                net [
                    49.0003.0000.0000.0006.00
                ]
                segment-routing {
                    mpls {
                        dynamic-adjacency-sids {
                            all-interfaces true
                        }
                    }
                }
                interface ethernet-1/11.0 {
                    circuit-type point-to-point
                    ipv4-unicast {
                        admin-state enable
                    }
                }
                interface ethernet-1/12.0 {
                    circuit-type point-to-point
                    ipv4-unicast {
                        admin-state enable
                    }
                }
                interface system0.0 {
                    passive true
                    ipv4-unicast {
                        admin-state enable
                    }
                }
            }
        }
    }
    segment-routing {
        mpls {
            global-block {
                label-range range-2-srgb
            }
            local-prefix-sid 1 {
                interface system0.0
                ipv4-label-index 6
            }
        }
    }
--{ + candidate shared default }--[ network-instance default ]--
</pre>

Once the default network-instance is configured on all the routers, reachability and the setup of tunnels can be checked as follows:

<pre>
// route-table and tunnel-table in pe1

A:pe1# show route-table
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+------------------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------------+----------------------+----------------------+---------------------------------------------+
|               Prefix               |  ID   | Route Type |     Route Owner      |  Active  |  Origin  | Metric  |    Pref    |   Next-hop (Type)    |  Next-hop Interface  |   Backup Next-hop    |          Backup Next-hop Interface          |
|                                    |       |            |                      |          | Network  |         |            |                      |                      |        (Type)        |                                             |
|                                    |       |            |                      |          | Instance |         |            |                      |                      |                      |                                             |
+====================================+=======+============+======================+==========+==========+=========+============+======================+======================+======================+=============================================+
| 10.1.4.0/30                        | 4     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 10.1.4.1 (direct)    | ethernet-1/10.0      |                      |                                             |
| 10.1.4.1/32                        | 4     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None (extract)       | None                 |                      |                                             |
| 10.1.4.3/32                        | 4     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None (broadcast)     |                      |                      |                                             |
| 100.0.0.1/32                       | 6     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None (extract)       | None                 |                      |                                             |
| 100.0.0.4/32                       | 1     | isis       | isis_mgr             | True     | default  | 10      | 18         | 10.1.4.2 (direct)    | ethernet-1/10.0      |                      |                                             |
+------------------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------------+----------------------+----------------------+---------------------------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 routes total                    : 5
IPv4 prefixes with active routes     : 5
IPv4 prefixes with active ECMP routes: 0
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--{ + candidate shared default }--[ network-instance default ]--
A:pe1# show tunnel-table
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 tunnel table of network-instance "default"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+---------------------------------------------+--------+----------------+----------+-----+----------+---------+--------------------------+---------------------------+---------------------------+
|                 IPv4 Prefix                 | Encaps |  Tunnel Type   |  Tunnel  | FIB |  Metric  | Prefere |       Last Update        |      Next-hop (Type)      |         Next-hop          |
|                                             |  Type  |                |    ID    |     |          |   nce   |                          |                           |                           |
+=============================================+========+================+==========+=====+==========+=========+==========================+===========================+===========================+
| 10.1.4.2/32                                 | mpls   | sr-isis        | 120001   | Y   | 0        | 11      | 2024-06-21T10:39:09.687Z | 10.1.4.2 (mpls)           | ethernet-1/10.0           |
| 100.0.0.4/32                                | mpls   | ldp            | 65537    | Y   | 10       | 9       | 2024-06-21T10:39:15.491Z | 10.1.4.2 (mpls)           | ethernet-1/10.0           |
| 100.0.0.4/32                                | mpls   | sr-isis        | 100005   | Y   | 10       | 11      | 2024-06-21T10:39:15.097Z | 10.1.4.2 (mpls)           | ethernet-1/10.0           |
+---------------------------------------------+--------+----------------+----------+-----+----------+---------+--------------------------+---------------------------+---------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
2 SR-ISIS tunnels, 2 active, 0 inactive
1 LDP tunnels, 1 active, 0 inactive
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv6 tunnel table of network-instance "default"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
<no_entries>
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--{ + candidate shared default }--[ network-instance default ]--

// route-table and tunnel-table in pe2

--{ + candidate shared default }--[ network-instance default ]--
A:pe2# show route-table
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+---------------------------------------+-------+------------+----------------------+----------+----------+---------+------------+------------------------+------------------------+------------------------+------------------------------------------------+
|                Prefix                 |  ID   | Route Type |     Route Owner      |  Active  |  Origin  | Metric  |    Pref    |    Next-hop (Type)     |   Next-hop Interface   | Backup Next-hop (Type) |           Backup Next-hop Interface            |
|                                       |       |            |                      |          | Network  |         |            |                        |                        |                        |                                                |
|                                       |       |            |                      |          | Instance |         |            |                        |                        |                        |                                                |
+=======================================+=======+============+======================+==========+==========+=========+============+========================+========================+========================+================================================+
| 10.2.3.0/30                           | 5     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 10.2.3.1 (direct)      | ethernet-1/11.0        |                        |                                                |
| 10.2.3.1/32                           | 5     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None (extract)         | None                   |                        |                                                |
| 10.2.3.3/32                           | 5     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None (broadcast)       |                        |                        |                                                |
| 10.2.5.0/30                           | 4     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 10.2.5.1 (direct)      | ethernet-1/10.0        |                        |                                                |
| 10.2.5.1/32                           | 4     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None (extract)         | None                   |                        |                                                |
| 10.2.5.3/32                           | 4     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None (broadcast)       |                        |                        |                                                |
| 10.3.6.0/30                           | 3     | isis       | isis_mgr             | True     | default  | 20      | 18         | 10.2.3.2 (direct)      | ethernet-1/11.0        |                        |                                                |
| 10.5.6.0/30                           | 3     | isis       | isis_mgr             | True     | default  | 20      | 18         | 10.2.5.2 (direct)      | ethernet-1/10.0        |                        |                                                |
| 100.0.0.2/32                          | 7     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None (extract)         | None                   |                        |                                                |
| 100.0.0.3/32                          | 3     | isis       | isis_mgr             | True     | default  | 10      | 18         | 10.2.3.2 (direct)      | ethernet-1/11.0        |                        |                                                |
| 100.0.0.5/32                          | 3     | isis       | isis_mgr             | True     | default  | 10      | 18         | 10.2.5.2 (direct)      | ethernet-1/10.0        |                        |                                                |
| 100.0.0.6/32                          | 3     | isis       | isis_mgr             | True     | default  | 20      | 18         | 10.2.3.2 (direct)      | ethernet-1/11.0        |                        |                                                |
|                                       |       |            |                      |          |          |         |            | 10.2.5.2 (direct)      | ethernet-1/10.0        |                        |                                                |
+---------------------------------------+-------+------------+----------------------+----------+----------+---------+------------+------------------------+------------------------+------------------------+------------------------------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 routes total                    : 12
IPv4 prefixes with active routes     : 12
IPv4 prefixes with active ECMP routes: 1
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--{ + candidate shared default }--[ network-instance default ]--
A:pe2# show tunnel-table
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 tunnel table of network-instance "default"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+------------------------------------------------+--------+----------------+----------+-----+----------+---------+--------------------------+-----------------------------+-----------------------------+
|                  IPv4 Prefix                   | Encaps |  Tunnel Type   |  Tunnel  | FIB |  Metric  | Prefere |       Last Update        |       Next-hop (Type)       |          Next-hop           |
|                                                |  Type  |                |    ID    |     |          |   nce   |                          |                             |                             |
+================================================+========+================+==========+=====+==========+=========+==========================+=============================+=============================+
| 10.2.3.2/32                                    | mpls   | sr-isis        | 120001   | Y   | 0        | 11      | 2024-06-21T10:39:09.125Z | 10.2.3.2 (mpls)             | ethernet-1/11.0             |
| 10.2.5.2/32                                    | mpls   | sr-isis        | 120002   | Y   | 0        | 11      | 2024-06-21T10:39:09.895Z | 10.2.5.2 (mpls)             | ethernet-1/10.0             |
| 100.0.0.3/32                                   | mpls   | ldp            | 65537    | Y   | 10       | 9       | 2024-06-21T10:39:11.546Z | 10.2.3.2 (mpls)             | ethernet-1/11.0             |
| 100.0.0.3/32                                   | mpls   | sr-isis        | 100004   | Y   | 10       | 11      | 2024-06-21T10:39:10.769Z | 10.2.3.2 (mpls)             | ethernet-1/11.0             |
| 100.0.0.5/32                                   | mpls   | ldp            | 65538    | Y   | 10       | 9       | 2024-06-21T10:39:11.977Z | 10.2.5.2 (mpls)             | ethernet-1/10.0             |
| 100.0.0.5/32                                   | mpls   | sr-isis        | 100006   | Y   | 10       | 11      | 2024-06-21T10:39:10.769Z | 10.2.5.2 (mpls)             | ethernet-1/10.0             |
| 100.0.0.6/32                                   | mpls   | ldp            | 65539    | Y   | 20       | 9       | 2024-06-21T10:39:13.315Z | 10.2.3.2 (mpls)             | ethernet-1/11.0             |
| 100.0.0.6/32                                   | mpls   | sr-isis        | 100007   | Y   | 20       | 11      | 2024-06-21T10:39:10.769Z | 10.2.3.2 (mpls)             | ethernet-1/11.0             |
|                                                |        |                |          |     |          |         |                          | 10.2.5.2 (mpls)             | ethernet-1/10.0             |
+------------------------------------------------+--------+----------------+----------+-----+----------+---------+--------------------------+-----------------------------+-----------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
5 SR-ISIS tunnels, 5 active, 0 inactive
3 LDP tunnels, 3 active, 0 inactive
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv6 tunnel table of network-instance "default"
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
<no_entries>
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--{ + candidate shared default }--[ network-instance default ]--
A:pe2#

</pre>

In addition, the following outputs show that the BGP sessions for EVPN and IP-VPN families are established.

<pre>
// BGP neighbors on pe1
--{ + candidate shared default }--[ network-instance default ]--
A:pe1# show protocols bgp neighbor
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+--------------------------+--------------------------------------+--------------------------+---------+--------------+---------------------+---------------------+------------------+--------------------------------------+
|         Net-Inst         |                 Peer                 |          Group           |  Flags  |   Peer-AS    |        State        |       Uptime        |     AFI/SAFI     |            [Rx/Active/Tx]            |
+==========================+======================================+==========================+=========+==============+=====================+=====================+==================+======================================+
| default                  | 100.0.0.4                            | overlay-ibgp             | S       | 65001        | established         | 0d:0h:41m:35s       | evpn             | [16/12/3]                            |
|                          |                                      |                          |         |              |                     |                     | l3vpn-           | [0/0/0]                              |
|                          |                                      |                          |         |              |                     |                     | ipv4-unicast     |                                      |
+--------------------------+--------------------------------------+--------------------------+---------+--------------+---------------------+---------------------+------------------+--------------------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
1 configured neighbors, 1 configured sessions are established, 0 disabled peers
0 dynamic peers
--{ + candidate shared default }--[ network-instance default ]--

// bgp neighbors on pe2

--{ + candidate shared default }--[ network-instance default ]--
A:pe2#  show protocols bgp neighbor
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+---------------------------+----------------------------------------+---------------------------+---------+--------------+----------------------+----------------------+-------------------+----------------------------------------+
|         Net-Inst          |                  Peer                  |           Group           |  Flags  |   Peer-AS    |        State         |        Uptime        |     AFI/SAFI      |             [Rx/Active/Tx]             |
+===========================+========================================+===========================+=========+==============+======================+======================+===================+========================================+
| default                   | 100.0.0.3                              | overlay-ibgp              | S       | 65023        | established          | 0d:0h:46m:29s        | evpn              | [8/8/11]                               |
|                           |                                        |                           |         |              |                      |                      | l3vpn-            | [0/0/0]                                |
|                           |                                        |                           |         |              |                      |                      | ipv4-unicast      |                                        |
| default                   | 100.0.0.5                              | overlay-ibgp              | S       | 65023        | established          | 0d:0h:40m:27s        | evpn              | [3/3/16]                               |
|                           |                                        |                           |         |              |                      |                      | l3vpn-            | [0/0/0]                                |
|                           |                                        |                           |         |              |                      |                      | ipv4-unicast      |                                        |
| default                   | 100.0.0.6                              | overlay-ibgp              | S       | 65023        | established          | 0d:0h:40m:47s        | evpn              | [3/0/19]                               |
|                           |                                        |                           |         |              |                      |                      | l3vpn-            | [0/0/0]                                |
|                           |                                        |                           |         |              |                      |                      | ipv4-unicast      |                                        |
+---------------------------+----------------------------------------+---------------------------+---------+--------------+----------------------+----------------------+-------------------+----------------------------------------+
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
3 configured neighbors, 3 configured sessions are established, 0 disabled peers
0 dynamic peers
--{ + candidate shared default }--[ network-instance default ]--

// bgp neighbors on br4

--{ + candidate shared default }--[ network-instance default ]--
A:br4#  show protocols bgp neighbor
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+-----------------------+---------------------------------+-----------------------+--------+------------+------------------+------------------+----------------+---------------------------------+
|       Net-Inst        |              Peer               |         Group         | Flags  |  Peer-AS   |      State       |      Uptime      |    AFI/SAFI    |         [Rx/Active/Tx]          |
+=======================+=================================+=======================+========+============+==================+==================+================+=================================+
| default               | 10.4.5.2                        | overlay-ebgp          | S      | 65023      | established      | 0d:0h:40m:29s    | evpn           | [16/0/3]                        |
|                       |                                 |                       |        |            |                  |                  | l3vpn-         | [0/0/0]                         |
|                       |                                 |                       |        |            |                  |                  | ipv4-unicast   |                                 |
| default               | 10.4.6.2                        | overlay-ebgp          | S      | 65023      | established      | 0d:0h:40m:49s    | evpn           | [16/0/19]                       |
|                       |                                 |                       |        |            |                  |                  | l3vpn-         | [0/0/0]                         |
|                       |                                 |                       |        |            |                  |                  | ipv4-unicast   |                                 |
| default               | 100.0.0.1                       | overlay-ibgp          | S      | 65001      | established      | 0d:0h:41m:43s    | evpn           | [3/0/16]                        |
|                       |                                 |                       |        |            |                  |                  | l3vpn-         | [0/0/0]                         |
|                       |                                 |                       |        |            |                  |                  | ipv4-unicast   |                                 |
+-----------------------+---------------------------------+-----------------------+--------+------------+------------------+------------------+----------------+---------------------------------+
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
3 configured neighbors, 3 configured sessions are established, 0 disabled peers
0 dynamic peers
--{ + candidate shared default }--[ network-instance default ]--

</pre>

## Configuration of EVPN and IP-VPN services on the PEs

After the IGP, tunnels and BGP sessions are properly configured and operating, services are configured on the pe routers. This 

