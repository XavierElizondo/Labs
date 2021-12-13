# Home Lab to emulate On-Premises (IPSec S2S) VPN to Azure

**Contents**

[Background](#Background)

[Components](#Components)

[Setup](#Setup)

[Functionality checks and validation](#Functionality-checks-and-validation)

[Acknowledgments](#Acknowledgments)

## Background

In my role as Cloud Solution Architect (CSA) at Microsoft I had the need to test scenarios and architectures of my own research and also validate customer architectures that involved a connection from On-Premises to Azure via [IPSec IKE Site to Site (S2S) VPN](https://docs.microsoft.com/azure/vpn-gateway/design#s2smulti). While you can also emulate this by creating a [VPN between VNets](https://docs.microsoft.com/azure/vpn-gateway/vpn-gateway-howto-vnet-vnet-resource-manager-portal), I wanted to really test this outside of the Azure fabric. I also wanted to avoid a major redo in my home network and perform the least amount of change and expense, so this lab worked perfectly for my needs.

I plan to update this article at a later time with a way for you to deploy as much as possible a similar environment in a more automated way but for now, this first round is meant to document my lab and also help you to build a lab environment similar to mine to give you the capability to demonstrate and validate VPN concepts.

It is also expected that you have some medium to advanced experience already with these network concepts.
  
### Architecture

![Environment](./media/home-vpn-lab-diagram.png)

**Note:** There's a sub-diagram for reference on how the the Surface laptop is "wired"  

## Components

This is a summary of the components:

1. **Surface Laptop.** I initially started with Windows 10 Enterprise, but recently upgraded to Windows 11 Enterprise without any issues or changes.
    1. **Network adapters.** Both, wireless and wired network adapters in my laptop are used. I rely on a Surface Dock for the wired adapter because this laptop does not come with one. Also there's a dependency to stay docked for this lab to work.
    2. **Hyper-V.** Hyper-V role is enabled.
    3. **Network adapters.** 2 virtual switches created, one External Switch (mapped to the wired adapter) and the Default Internal Switch (mapped wireless adapter)
2. **Windows 10 Hyper-V VM.** Used for connectivity testing. This VM is connected to the Hyper-V internal network. You could use a Linux or any other supported OS as long as you have tooling installed for testing (curl, ping, route etc.)
3. **pfsense firewall Hyper-V VM.** Used to establish IPSec VPN with Azure. I've used pfsense for several years and got used to it although sooner or later I will switch to OpenSense which it is very similar and a lot more investments nowadays than pfsense.
    1. **Network adapters.** Uses 2 network interfaces, connected to the Hyper-V Internal and External Switch respectively
    2. **Dynamic Routing Package** You will need to install an additional package in order to use BGP dynamic routing for the VPN tunnel. You can also do static routing, if desired. In my lab I'm using FRR although I originally started with OpenBGPD but went out of support
4. **AT&T Gateway.** My ISP is AT&T and they provide a Gateway/Router that has built-in PAT/NAT features
5. **Azure Subscription**
    1. **[Hub and Spoke network topology](https://docs.microsoft.com/azure/architecture/reference-architectures/hybrid-networking/hub-spoke?tabs=cli).** An operational Hub VNet with peering to a spoke VNet
    2. **VPN Gateway.** An Azure Virtual Network Gateway deployed in the Hub VNet. For [routed-based (dynamic routing)](https://docs.microsoft.com/azure/vpn-gateway/vpn-gateway-vpn-faq#what-is-a-route-based-dynamic-routing-gateway) with BGP the [minimum SKU supported is VpnGw1](https://docs.microsoft.com/azure/vpn-gateway/vpn-gateway-about-vpngateways#benchmark).

## Setup

### Surface Laptop

1. **Network Adapters in Windows.** These is how the 2 adapters are shown from Network Connections
![Surface laptop - network connections](./media/surface-net-connections.png)

2. **Hyper-V Virtual Switches.** These is how the 2 adapters are shown Hyper-V Virtual Switch Manager. The Default virtual switch is created automatically when enabling the Hyper-V role. The External virtual switch was created after.
![Hyper-V - (Default) vSwitch Internal ](./media/hyper-v-switch-internal.png)
![Hyper-V - vSwitch External](./media/hyper-v-switch-external.png)

3. **Hyper-V VMs network mappings** A summary view of both VMs
![Hyper-V - VMs](./media/hyper-v-vms.png)
    1. **pfsense firewall VM.** The pfsense VM has the 2 adapters configured as below in Hyper-V
    ![pfsense firewall VM - Internal NIC](./media/pfsense-hyper-v-nic-internal.png)
    ![pfsense firewall VM - External NIC](./media/pfsense-hyper-v-nic-external.png)
    2. **Windows  10 VM.** This VM NIC is configured the same as pfsense Internal NIC
4. **pfsense firewall configuration.**
    1. **Interfaces.** These are the configured interfaces. Note that the OPT1 interface is a Virtual Tunnel Interface (VTI) that is created when the IPSec tunnel is configured.
        ![pfsense - internaces](./media/pfsense-interfaces.png)
    2. **FRR package.** This is the installed package for handling BGP
        1. **FRR package info**
        ![FRR package - info](./media/pfsense-bgp-package-info.png)
        2. **Global Settings.** Here you enable FRR globally and set the Router ID which is the pfsense LAN interface IP.
        ![FRR - Global Settings](./media/pfsense-bgp-package-global-settings.png)
        3. **Prefix lists.** Here we create a prefix list that is required to allow inbound and outbound communications between the BGP peers. This list will be used later in step 6. Without this list the BGP message updates would be discarded despite the BGP peers established a connection.
        ![FRR - prefix-lists](./media/pfsense-bgp-package-prefix-lists.png)
        4. **BGP Router Options.** Here we set the AS used by this BGP peer instance. The Router ID is inherited from the Global settings already configured in step #2.
        ![FRR - Router Options](./media/pfsense-bgp-package-router-options.png)
        5. **BGP Network Distribution.** Here we configured the neworks that will be advertised by this BGP instance. If you wanted to advertise pfsense static and kernel routes including 0.0.0.0/0 for force tunneling you would enable IP4 in the Redistribute Kernel routing table/pfSense static routes drop down option and leave the Networks to Distribute field blank. Otherwise we specify the specific network(s) we want to advertized like the image below
        ![BGP - Network Distribution](./media/pfsense-bgp-package-network-distribution.png)
        6. **BGP Neighbors.** Here you add a neighbor and configure its settings. Name/Address: The Azure VNet Gateway BGP peer IP address.
        ![FRR - BGP Neighbors General options](./media/pfsense-bgp-package-neighbors-general-options.png)
        Remote AS: The Azure VNet Gateway ASN
        ![FRR - BGP Neighbors Basic options](./media/pfsense-bgp-package-neighbors-basic-options.png)
        Prefix List Filter: The previously created Prefix lists in step #3.
        ![FRR - BGP Neighbors Peer filtering](./media/pfsense-bgp-package-peer-filtering.png)


    3. **Firewall rules.** For testing purposes overall it is very much open and not locked down
        1. **Rules - WAN** Key traffic to be allowed to the external interface IP is BGP (TCP & UDP:179) as well IPSec NAT-T (UDP:4500)
        ![Rules - WAN](./media/pfsense-rules-wan.png)
        2. **Rules - LAN**  Key traffic to be allowed is BGP (TCP & UDP:179)
        ![Rules - LAN](./media/pfsense-rules-lan.png)
        3. **Rules - IPSec**
        ![Rules - IPSec](./media/pfsense-rules-ipsec.png)
    4. **IPSec config.**
        1. **IPSec - Summary**
        ![IPSec - Summary](./media/pfsense-ipsec-summary.png)
        2. **IPSec - Phase 1.** 
        Remote Gateway. The Azure VNet Gateway Public IP
        My Identifier. The Public IP associated to my Home Router
        ![IPSec - Phase 1](./media/pfsense-ipsec-p1-1.png)
        3. **IPSec - Phase 2**
        Mode: Routed as the VNet Gateway type is also route-based.
        Local Network: 172.31.248.61, because the BGP peer IP from Azure is 172.31.248.62 and it is a route-based VPN we need an /30 transit network (2 IP addresses) as the networks will be learned by BGP. If we did not use BGP, we would then specify manually remote networks here.
        Remote Network: The BGP peer IP from Azure, 172.31.248.62
        ![IPSec - Phase 2](./media/pfsense-ipsec-p2-1.png)

5. **AT&T Gateway.** Here is where you create custom services to configure the required PAT/NAT (AKA port forwarding) for the VPN, then assign to a device in your network, in this case the pfsense firewall VM (external IP, 192.168.11.10). In this way the Gateway does *not* need to be configured in bridge mode/pass through to allocate the public IP to the pfsense firewall VM external adapter. Only the 2 ports below are required to forward or allow the pfsense firewall terminate the VPN connection.
![AT&T Router - Custom Services](./media/ATT-GW-NAT-1.png)
![AT&T Router - Assigning custom services](./media/ATT-GW-NAT-2.png)

6. **Azure Hub Spoke network.**
    1. Hub VNet address space and peering with the spoke VNet
    ![Hub VNet- address space](./media/vnet-hub-eastus-address-space.png)
    ![Hub VNet- peerings](./media/vnet-hub-eastus-peerings.png)
    2. VPN Gateway configuration
    ![Hub VNet- address space](./media/vpg-gw-config.png)
    3. Local Network Gateway configuration
    ![Hub VNet- address space](./media/lng-gw-config.png)
    4. VPN Connection configuration
    ![Hub VNet- address space](./media/vpn-connection-config.png)

## Functionality checks and validation

### IPSec tunnel review

1. **pfsense IPSec Tunnel - Status**
We can see below the tunnel is established successfully and the IPSec security associations (SAs) between the VPN GW public IP (20.120.24.138) and the pfsense external IP (192.168.11.10)
![IPSec - Status 1](./media/pfsense-ipsec-status-1.png)

2. **Azure VPN gateway connection - Status**
We can see below the IPSec connection is established successfully
![Connection - Status](./media/vpn-connection-status.png)

### Routes review

1. **pfsense BGP dynamic routing - status.** We can see below that pfsense successfully retrieves dynamic routing from the Azure VNet Gateway (AS 65515, BGP Peer IP 172.31.248.62); the **Hub VNet address space** (172.31.248.0/22), and  the **Spoke VNet** (10.50.0.0/24).

![BGP status - 1](./media/pfsense-bgp-status-1.png)
![BGP status - 2](./media/pfsense-bgp-status-2.png)

2. **Azure VPN gateway connection - Status.** We can see below that the Azure VNet Gateway  successfully retrieves dynamic routing from pfsense (AS 65505, BGP Peer IP 192.168.1.1); the **On-prem network** 192.168.1.0/24.

![VPN BGP - Status](./media/vpn-bgp-status.png)

**Alternative Azure VPN Gateway BGP check.** Use [Portal](https://docs.microsoft.com/en-us/azure/vpn-gateway/bgp-diagnostics), PowerShell or CLI. You can also find a PowerShell script where you can dump BGP routes for both VPN and ExpressRoute Gateways, consult: [Verify BGP information on Azure VPN and ExpressRoute Gateways](https://github.com/dmauser/Lab/tree/master/VNG-BGP-Info).

## Acknowledgments

Special thanks to [Nehali Neogi](https://github.com/nehalineogi/), [Heather Sze](https://github.com/hsze/) for their insights in this lab and [Daniel Mauser](https://github.com/dmauser) for creating great documentation that we can reuse and repurpose