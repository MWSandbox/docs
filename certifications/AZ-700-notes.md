# AZ 700

- 5 IP addresses of every subnet are reserved: the first 4 + the last one
- VMs can communicate between subnets on the same VNET if there is no additional NSG setup
- Azure Network Encryption: Encrypt traffic within VNet via DTLS tunnel (accelerated networking needs to be enabled in VMs)
- Subnets are not restricted to zones

# Public IP
- SKUs:
  - Basic: 
    - Dynamic & Static for IPv4
    - NSGs optional
    - AZs / routing preference / global tier not supported
  - Standard: 
    - Only static
    - Security via NSGs required
    - Nonzonal / zonal / zone redundant (in regions with 3 AZs)
    - Routing preferences for control how traffic is routed between azure and internet
    - Supports global tier (cross-region load balancer)
  - Config: DNS name can be created, DDoS protection settings (inherit or specific), routing preference
- Public IP prefixes:
  - Pull from predictable list of public IP addresses
  - Choose up to 16 addresses to be reserved that will be next to each other
  - Advantages: Whitelisting, faster creation
  - BYOIP feature:
    - Public IP address need to be registered, but Azure will advertise them in the WAN


# VMs
- VMs can be attached to multiple private and public IP addresses
- Inbound ports can be specified for VMs -> will generate NSG automatically
- Availability set: Logical groupings to reduce chance of correlated failures by assigning different fault domains
- DNS servers can be configured in NIC settings
- Microsoft Azure Network Adapter (MANA)
  - Better performance and availability due to new network interface
  - Requires hardware and software components:
    - For Windows and Linux
    - Accelerated Networking needs to be enabled on NIC -> MANA NIC as a PCI device in the VM
    - Accelerated networking requires VMs with at least 2 vCPUs

# VPN
- Site-to-Site VPN:
  - Via IPsec Tunnel -> connects 2 public IP addresses on each side
  - Network gateways on each side (VPN gateway on Azure side)
  - Gateway Subnet (specific subnet) needs to be created
  - Service: Virtual network gateway (VPN device, handles also encryption/decryption)
    - Configuration
      - VPN types in Azure
        - Route-based: Route tables define where to send packets to, the tunnels themselves allow all traffic -> static or dynamic routing protocols, enables automatic failover
        - Policy-based: Each VPN tunnel defines a policy of which traffic to permit (only those packets will be sent through each tunnel) -> static routing
          - Custom policy need to be configured in order to connect route-basen gateway to onprem policy-based device
      - SKUs:
        - Basic: 10 Tunnels, no P2S OpenVPN Connections, 100 MBit/s, kein BGP, legacy SKU
        - VpnGw1-5: More tunnels, more P2S connection, more bandwidth, AZ suffix for zone-redundancy
          - 1: 30 S2S, 650 MBit/s, 250 OpenVPN P2S
          - 2: 30 S2S, 1 GBit/s or 1,25 GBit/s (Gen2), 500 OpenVPN P2S
          - 3: 30 S2S, 1,25 GBit/s or 2,5 GBit/s (Gen2), 1000 OpenVPN P2S
          - 4: 100 S2S, 5 GBit/s, 5000 OpenVPN P2S
          - 5: 100 S2S, 10 GBit/s, 10000 OpenVPN P2S
      - Generation:
        - 1: Less P2S connections
          - 128 SSTP P2S
        - 2: More P2S connections
          - 128 SSTP P2S
      - Active-active vs active-standby mode: 2 connections in parallel for failover
      - BGP protocol support for route-based VPNs - e.g. for Hub and Spoke models
        - ASN identifies a network
    - Connections can bbe added with Local network gateway as target to establish VPN connection
      - Shared key (PSK) is required on each side, will be exchanged by IKE protocol
      - Types: VNet-to-VNet, Site-to-Site (IPsec), ExpressRoute
  - Virtual WAN should be used if you need more than 30 S2S connections
  - Local Network Gateway:
    - Representation of OnPrem gateway (pointing to OnPrem VPN gateway)
    - Configuration:
      - Endpoint: IP or FQDN
      - Define address space of Onprem network that does not overlap with Azure network
      - BGP support
- Extended networks for azure:
  - Extend Azure network with onprem network (e.g. Server cannot be moved to the cloud)
  - Alternatives should always be preferred (e.g. move server to Azure)
  - Using VXLAN tunnel between 2 Windows Server 2019 (2022 for the one in Azure) VMs that are capable of running nested virtualization
  - In Windows Admin center, there is an extension for extended networks
  - After tunnel has been established, IP addresses need to be defined that shall be extended into Azure
- Point-2-Site VPN:
  - Individuals connecting to Azure
  - A root certificate needs to be created, each client needs a certificate to connect, signed by the root cert
  - Configuration:
    - Address pool assigned to clients dynamically 
    - Root certificate public data
    - Revoked client certificates
  - VPN Client can be downloaded from Azure directly to start a connection from client side
    - If network topology changes, client file needs to be re-downloaded
  - Certificates public/private keys can be exported on Windows using certmgr
  - Export client cert in pfx format with private key
  - Tunnel types:
    - OpenVPN: Supports all auth. options
    - SSTP: Supports Radius + Certificate
    - IKEv2 (alone, with OpenVPn or with SSTP): Supports Radius + Certificate
  - Authentication types:
    - Azure certificate
    - RADIUS auth.: Required RADIUS server, can be used onprem AD and SSO
    - Azure AD -> Add Azure VPN as enterprise application to AD
  - Always On:
    - Maintain VPN connection, will automatically connect and reconnect based on triggers (e.g. user sign-in, device screen active)
    - Device tunnel: Connects before user login
    - User tunnel: Connects after user login
    - Windows built-in VPN solution only with Win 10 and certificate auth
- Diagnostic logs:
  - IKEDiagnosticLog: Verbose debug logging for IKE/IPsec. This is very useful to review when troubleshooting disconnections, or failure to connect VPN scenarios.
  - GatewayDiagnosticLog: Configuration changes auditing
  - TunnelDiagnosticLog: Inspect historical connectivity statuses of the tunnel
  - RouteDiagnosticLog: Traces the activity for statically modified routes or routes received via BGP
  - P2SDiagnosticLog: traces the activity for Point to Site. 
- Client diagnostics for P2S:
  - Status logs
  - sign in information can be cleared (Entra ID auth)
  - Run diagnostics - checks internet access, client credentials, server resolvable and reachable
- IKE / IPsec policy:
  - Encryption settings (phase 1 / 2), IPSec SA lifetime in KB and seconds
- Virtual network gateway provides a health endpoint via https on port 8081
  - Requires at least /27 subnet
- Reset VPN connections if only some VPN tunnels are losing connectivity

# ExpressRoute
- Onprem network connected to network provider edge location with dedicated line
- network provider has a primary/secondary dedicated connection to microsoft edge network
- Pricing based on bandwidth
- Inbound data transfer is free, unlimited data plan is available (includes outbound, but not global reach)
- SKUs:
  - ER Premium to connect to all regions, otherwise only a single region
  - Standard: Connect to all regions within one geography (e.g. West Europe + France Central)
  - Local SKU: Only one Azure region in the same metro, no extra egress data transfer fee
- Global Reach: Multiple ExpressRoute connections can be connected (WAN)
  - Communication between OnPrem networks via ExpressRoute is possible
- ER Direct: No network provider required, network will be connected directly with Azure network, physical isolation (higher security)
- Configuration via Virtual Network Gateway
  - SKUs:
    - Standard / ERGw1Az: ER + VPN, no fast path, 4 circuit connections, 1 Gbps
    - High Performance / ERGw2Az: ER + VPN, no fast path, 8 circuit connections, 2 Gbps
    - Ultra / ERGw3Az: ER + VPN, Fast path, 16 circuit connections, 10 Gbps
  - Can be deployed in 1-3 AZs (increased pricing), AZ SKUs
  - Requires public IP
- ExpressRoute with VPN failover:
  - Only route-based VPN supported with default ASN
  - If express route and VPN are configured for the same VPN gateway, BGP will do failover automatically, Azure prefers ER over VPN when both are available
  - If BGP is not used, failover can be done by manually changing routing
- Circuits: Represent logical conenction between OnPrem and Azure
  - Each circuit can be in the same or different regions
  - Circuit connections can be shared accross 10 subscriptions and depending on the SKU multiple regions
  - S-key: Unique key
  - Peering types:
    - Microsoft: 365 and Azure PaaS services not deployed in a VNet over public IPs owned by company
      - To enable:
        - Either contact provider (if managed Layer 3 services)
        - Or: Primary / secondary subnet (/30), SNAT, VLAN ID, ASN, Advertised prefixes (public)
    - Private: Azure IaaS and PaaS solutions deployed in a VNet, uses private IP addresses of company network, supports more IP addresses
      - To enable: ASN, IPv4 primary/secondary BGP peer, VLAN ID, Shared Key
    - Can all be enabled at once
  - To create a circuit:
    - Create circuit
    - Exchange service key with connectivity provider
    - Configure private peering
    - Create connection from VNet Gateway to ExpressRoute circuit (resilience settings)
- Encryption:
  - Point-to-point by MACsec (Layer 2): Encrypt physical links between network devices and Microsoft's network devices (ER direct only)
    - Enabled on ER Direct ports by default (key stored in keyvault)
  - End-to-end by IPsec (Layer 3)
    - Can be enabled in addition to MACsec
    - VPN connection over ER
    - P2S Users can use ER S2S tunnel
- Bidirectional Forwarding Detection (BFD):
  - Speed up link failure detection (< 1 sec instead of 3 min.)
  - Configured by default on al newly created ER peering interfaces -> Enable BFD on own primary, secondard devices
    1. Enable BFD on the interface
    2. Link it to the BGP session

# Security
- Firewall vs WAF on Application Gateway:
  - WAF only filters incoming HTTPS traffic, handles Cross Site Scripting, etc.
  - Firewall also outgoing and non HTTPS traffic, more logic
  - Route table needs to be configured to route traffic through the firewall subnet - can also be setup the same way for App gateway
  - if both are configured: route inbound traffic through app gateway subnet and outbound traffic through firewall subnet
  - Azure Firewall requires its own dedicated subnet

## Azure Firewall
- Microsoft Threat Intelligence: Knows malicious IPs and FQDNs
- Needs to be placed in its own subnet which needs to be called "AzureFirewallSubnet"
- Tiers:
  - Basic (very low bandwidth, no content filtering)
  - Standard
  - Premium: 
    - TLS inspection (TLS termination / 2 separate TLS connections from source to target)
    - URL filtering
    - IDPS (intrusion detection and prevention system, check byte sequencey in network traffic, etc.)
    - Web categories: Categorizes URLS using the complete URL not only FQDN (standard) - e.g.: google.com/news -> News
- Firewall management:
  - via firewall policies
    - Policiy config:
      - Can be added via collections or single rules
      - Priorities will be used
      - Allow/Deny
      - Define Source and Destination (IP/FQDN) as well as ports
      - Rule groups to collect rules / rule collections
    - Types:
      - Parent policy: Inheritance of rules
      - Rule collections: Set of rules with same priorities and action
      - DNAT rules: TCP/UDP protocols + ports + translated address and port
        - E.g. use public ip of firewall and RDP port -> forward it to private VM + RDP port
      - Network rules: TCP/UDP/ICMP/Any + ports
        - Can be used to Allow DNS lookup (UDP port 53) and add IP addresses of DNS servers, filter base on IP, ports
      - Application rules: http/https/etc. protocols + ports
        - Filter based on FQDNs, URLs, etc.
  - via firewall rules (classic)
- Requires public IP
- Can act as a DNS proxy (e.g. to forward DNS requests to Azure internal DNS)
- Traffic needs to be routed through the firewall network (use private IP address of firewall)
- RG of firewall needs to be the same as the one of the VNET
- Forced tunneling can be enabled when VPN tunneling is used
- IP groups can be used for easier rule management
- Firewall manager:
  - Allows management of firewall policies across network
  - Types of networks that can be secured that way: Virtual WAN, Hub + Spoke
  - Detect VNets / WAN that do not have firewall attached to it
  - Policies can be associated with other networks / hubs
## WAF
- Core rule set which protects against OWASP (supports different versions)
- Integrations to application gateway, front door, CDN
- WAF global policy with managed rules for OWASP
- Custom rules can be added and can override default rules:
  - Priority
  - Match Type (e.g. string)
  - Match Variable (e.g. RequestUri)
  - Operation (is / is not)
  - Operator (e.g. contains)
  - Transformations
  - Match Values
  - Action (Allow / deny)
## NSGs
- Default rules cannot be altered: 
  - Inbound: Allow within Vnet traffic, load balancer traffic and deny everything else
  - Outound: Allow towards Vnet, Internet, deny anything else (other private IPs)
- Can be attached to NICs or subnets
- Applications (https, etc.) and protocols (TCP, UDP, ICMP) configurable
- Source / targets: IPs / Service Tags / Application Security Group
- Service Tags: Internet, etc. -> A group of IP address prefixes from a given Azure service
- Application security group: Group together resources with the samen security / network requirements

# Service endpoints
- Need to be setup on the subnet side and on the service side that should be connected (add exception in network settings for subnet that should be conencted)
- Better performance and latency compared to public / Azure managed network
- Increased security
- Service does not need to be in the same region and subscription as the subnet (except SQL Database)

# DNS
- Azure DNS Private Resolver
  - Azure private DNS records can be used OnPrem
  - Provides DNS services between On-Prem and Azure
  - Conditional forwarding via rules -> e.g. if internet needs to be resolved, the Azure DNS Private Resolver can forward those requests to an internet DNS server
  - Inbound (from on prem to azure)/outbound (from azure to on prem) endpoints, latter will be forwarded to onprem DNS
- Public DNS zone
  - You can register any domain name -> Multiple Name servers of azure will be displayed
  - Azure name servers need to be configured in a domain name registrar
- Records:
  - A = Point to IPv4 address
  - CNAME = Link subdomain to another record
  - NS = Authorative name server for that domain
  - SOA = Start of Authority - stores metadata like the email address of the admin
  - Use @ sign in DNS record to refer to the standard domain name without subdomain
- Private DNS Zone:
  - No NS record
  - Needs to be linked to VNETs
  - Auto registration can be enabled -> All VMs will automatically receive a nicename
    - Static IP address changes need to be manually changed in DNS entrys as well
- DNS settings can be configured for VNet (default = Azure, but custom servers possible)
  - Azure DNS private Zones
  - Azure-provided name resolution
  - Custom DNS server
  - Azure DNS Private Resolver
- NIC settings can override VNet settings

# Connecting VNETs
- VNET peering
  - Types:
    - Standard: Within the same region
    - Global: Across Azure regions
  - Traffic over Microsoft backbone network, not public internet
  - Pricing for outbound and inbound traffic even for standard VNET peering (charged twice)
  - IP ranges should not be overlapping
  - Configuration:
    - Block traffic from one direction to the other
    - Allow forwarding of traffic from one direction to the other (VNet in between)
    - Gateway transit: Use the Virtual Network Gateway of another VNet and route server
  - Sensitive data should be encrypted -> Use network gateway
  - In a globally peered Vnet, resources in one Vnet can communicate/interact with the front-end IP addresses of a Basic internal load balancer.
  - Service chaining:
    - Direct traffic to a resource in another VNet via user-defined routes (next hop IP address in different VNet - e.g. Gateway)
    - Cannot be used when there is a user-defined route that specifies ER gateway as next hop type
  - VPN peering is non-transitive: To make it transitive a VNET Gateway needs to be used

# WAN
- Hub & Spoke:
  - All network connect to one hub, instead of connecting all networks to each other
- Virtual WAN:
  - Types:
    - Basic: Only VPN S2S
    - Standard: ExpressrRoute, all VPN options, VNET-to-VNET, etc.
  - Pricing:
    - Per hour, per GB of use, bandwidth dependent, per connection
    - Scale units: Throughput of gateways in WAN (1 - 200)
      - S2S: 1 SU = 500 Mbps
      - P2S: 1 SU = 500 Mbps, supports 500 clients
      - ER: 1 SU = 2 Gbps
    - SKUs:
      - Basic: S2S only
      - Standard: All integrations
  - Hub:
    - Has its own network private address space
    - Can create gateways for VPN S2S/P2S and ExpressRoute
    - Options for traffic to tracel accross Microsoft network as long as possible before it enters public internet
  - VPN Sites:
    - Similar to local network gateway for the Virtual WAN
    - Links can be added for the actual VPN Gateway OnPrem: IP address, BPG address, ASN number, Link speed etc. need to be entered
    - After creating the link, the site needs to be connected:
      - Pre-shared key can be entered, protocol and routing options can be selected
  - Virtual network connections:
    - VNETs can be selected and route table assigned
  - Transit VNet connectivity can be established by connecting a NVA in a connected VNet and peer VNets with this Vnet
    - Add custom routes to the outer VNets to point to the NVA
    - Routes will be synced automatically through BGP peering from Virtual WAN router
  - Routing
    - Route tables can be created manually and associated to VNets and VPN / ER
    - Propagation of routes from connections towards the route table can be enabled
    - Routing preference can be set for hub (ER / VPN / AS Path)
    - Routing policies (internet / private) of a hub can be used to forward traffic through a firewall -> will advertise new default route to all connections
  - Private endpoints deployed to one connected VNet will be available for all other connected VNets

# Routing
- System routes: Default set of routes
  - Routing within the VNET
  - 0.0.0.0/0 -> Internet
- VNet peering, Virtual network gateway, Service Endpoints -> Will add routes to the system route table
- Route tables can be created / assigned per subnet
- IP forwarding needs to be enabled on the NIC and within the OS to allow the device receiving traffic that is sent to a different IP 
- Forced tunnel: Go through VPN tunnel instead of going directly to the internet
  - Frontend still can route directly to the internet, but mid-tier and backend should be routed through OnPrem for increased security
  - Can be configured for an existing local network gateway
  - Configuration options:
    - BGP: VPN Gateway with BGP -> advertise 0.0.0.0/0 via BGP from OnPrem to Azure
    - Default Site: Can be configured in VPN gateway settings, OnPrem device needs to use 0.0.0.0/0 as traffic selectors
    - User-defined routes: To enable forced tunneling only for specific subnets
- Route Server:
  - Prefer either ER / VPN / ASPath (shortest BGP AS Path)
  - Needs to be deployed in its own subnet
  - Requires public IP
  - Routing information can be exchanged dynamically between azure networks (use routes automatically in ER / VPN Gateways instead of configuring manually) and onprem
    - Routes are automatically configured once advertised via BGP
- Debug:
  - NSG rules are visible in VM settings
  - Effective NSG rules and effective routes are visible in NIC settings with next hop type / IP
  - In network watcher using next hop diagnostic tool

# Load Balancing

![Decision tree](./resources/load-balancing-decision-tree.png)

## Load Balancer
- Basic tier is free
  - Up to 300 instances
  - VMs in a single availability set or a VMSS
  - Health probes (TCP, HTTP)
  - Does NOT support: Multiple AZs, high Availability, HTTPS health probes, optional NSG, no SLAs, no cross-region traffic over peered networks
- Standard tier:
  - Up to 1.000 instances
  - Any VM, any VMSS in a single VNet
  - HTTPS health probe, multi AZ, high availability (for internal LB), 99,99% SLA supported
  - NSG is required
- Gateway tier: For 3rd party NVAs 
- Public vs. internal/private LBs
  - Public: Outbound connections of VMs through NAT
- Layer 4 (transport)
- High availability through global load balancer that can balance regional LBs
- Multiple IPs can be connected to the same LB
- Health probes need to be added manually:
  - HTTP GET to a specific path (check for 200 response) or TCP with interval and unhealthy threshold
- Only supports balancing traffic inside a single VNet
- Load balancing rules:
  - Specify protocol, port and FE IP
  - Port can be changed while routing
  - Session persistence can be set (always route to the same server within the same session), idle timeout for sessions can be set
  - Floating IP: To reach the same port on the same VM multiple times (requires multiple NICs)
- Inbound NAT rules: To route traffic to specific VMs
  - v1: Single machine
  - v2: Whole BE pool
- Outbound rules:
  - SNAT -> Use public IP of LB to provide outbound internet connectivity for BE
- Good use case in front of IaaS (VMs)
- HA ports: 1 load balancing rule for multiple protocols

## Application Gateway
  - Layer 7 (application) -> URLs, paths, etc.
  - Limited to HTTP/web traffic
  - Requires a subnet that can only host App GWs
  - Optionally includes WAF
  - Supports load balancing across different AZs and VNets within a single region
  - Can be used with AKS as Application GW Ingress Controller
  - SKUs: Standard or with WAF
  - Autoscaling options for the Gateway itself
  - Supports public/private or both IP configs for Frontend
  - Supports other services in addition to azure VMs (compared to Load Balancer): App Services, IP addresses, FQDNs
  - Each backend pool needs its own listener and routing rules
  - Routing rules:
    - HTTP/HTTPS traffic only
    - Multi-site to register more web applications on the same port (e.g. with different FQDNs)
    - Cookie-based affinity to root based on cookies to the same target server
    - Connection draining: When server gets shutdown, wait until there is no more traffic 
    - Backend paths and host names can be overriden
    - Path-based routing rules vs basic routing rules (forward all requests on listener to be pool)
  - HTTP headers can be rewritten
    - Associated with routing rules
    - Conditions and priorities can be added
    - Can modify request and response headers
    - URLs can be also rewritten, e.g. to respond with a different API path than the actual server
  - Manual and autoscaling options (min. 2 instances) for app gateway itself
  - WAF modes: Detection (without any actions) vs. Prevention

## Traffic Manager
  - DNS based global routing (layer 7) between regions -> e.g. closed geographically routing
  - Automatic failover when one region goes down
  - Not only limited to web traffic
  - Similar to Front Door, but without caching, WAF, etc. -> only routing
  - Config:
    - DNS name with suffix trafficmanager.net (CNAME can be used)
    - Routing methods: Performance, Weighted (% based), Priority (All traffic to one location as long as available), Geographic (based on DNS query location - e.g. for compliance reasons regarding data sovereignity), MultiValue (Return all IPs for a given DNS), Subnet (CIDR)
  - Possible endpoints: Azure (either choose service or public ip), External, Nested (another traffic manager (e.g. first geographic, then performance))
  - Traffic manager monitoring settings for different endspoints requires nested traffic manager setup

## Azure Front Door
  - Global load balancing
  - SKUs:
    - Standard
    - Premium: More security features
  - Frontend config:
    - host name with azurefd.net suffic -> CNAME to redirect own DNS
  - Backend config:
    - Supports also Storage Accounts, Application Gateway, API Management, Traffic Manager
    - Health probes (HTTP or HTTPS)
  - Routing:
    - Path-based, URL rewrite
    - Forward vs. Redirect (change URL)
  - Caching feature for static content and dynamic compression to enhance performance
    - Cache can be purged on demand
  - Offers session affinity and WAF features
  - Layer 7 (application)
  - Optionally includes WAF, caching and app acceleration features
  - Support SSL offloading
  - Good use case in front of PaaS (Functions, App Service, etc.)

# NAT Gateway
- Idle timeout until connections get removed
- Autom. sets up routing for the subnet to route everything through the gateway
- NAT only support IPv4, not IPv6 and can span only one VNet
- VMs in private subnet can connect outbound to the internet while staying private
- Subnet placement:
  - Not in subnets where basic SKUs are being used
  - Not in GW subnet
  - Only one NAT GW per subnet
- Can be used together with LB for dual stack outbound connectivity

# Monitor
- Dashboard for network health including network health, connectivity and traffic
- Network health:
  - View metrics like throughput, response status, failed requests
  - Diagnostics need to be enabled in order to receive metrics
    - View a list of which resources have diagnostics enabled within Azure Monitor
    - Collected data will be stored in storage account, log analytics workspace or streamed to an event hub or stored in an external solution
    - Storage account is cheaper and logs can be kept indefinetely, log analytics workspace offers query options
    - Specify which logs and metrics to collect and the retention period
  - DDoS logs can be collected for public IP addresses
- Connectivity:
  - Connection monitor needs to be created within th network watcher
  - Network watcher agent needs to be installed on VMs
  - Can be added within VM configuration at extension menu
    - Configuration:
      - Can create its own workspace or choose your own
      - Test groups can be created to define tests between source and destination pair
        - Protocol (HTTP, TCP, ICMP), port, frequency, header, endpoint path, expected status code need to be defined
        - Thresholds for failed tests and time to receive successful tests
        - Alerts can be created for the test groups
- Traffic:
  - Flow logs and traffic analytics
  - Flow logs need to be enabled on NSG level - will be stored in storage account
    - Traffic analytics need to be enabled during that process and will be stored in log analytics workspace
  - Traffic analytics can be viewed within Network watcher - Azure monitor only provides status overview
    - Traffic distribution, NSG hits, ports being used, traffic visualization
    - Click on malicious request will forward to log analytics workspace to view details about those requests
- Action groups (what to do in case of an alert): E-Mail, 

## Microsoft Defender for Cloud
- Needs to be enabled - different plans:
  - Cloud Security Posture Management (CSPM): Agentless
  - Cloud Workload Protection (CWP): Specify different services (Databases, VMs, DNS, etc.)
    - Can be enabled per service
    - DNS protection:
      - Monitors Azure DNS resolution
      - Data exfiltration through DNS tunneling (Malware sendet sensible Daten über Subdomains an Angriffsserver - DNS traffic geht of an Firewalls vorbei)
      - Malware communications with command & control servers
      - Malicious DNS resolvers
      - Domains used for malicious activities
      - Anomalies

## DDoS Protection
- Basic protection is free
- DDoS protection plans
  - Includes monitoring & Alerts
  - Plans
    - Network protection (VNET): Supports Basic + Standard SKUs, cheaper and little more features
    - IP protection (pay per protected IP model): Only supports standard SKU
  - Automatic monitoring and attack mitigation (L3/L4)
  - Application policies can be setup for attack mitigation (e.g. expect only specific ports)
  - Mitigation flow logs
  - Integration with firewall manager
  - Can protect: VNET, Firewall, App GW, bastion, load balancer, NICs, VMSS, VNET Gateways
- DDoS network protection needs to be enabled for VNETs in configuration
- Azure policy can be setup to require DDoS Protection for all networks
- Public IP protection can be used as an alternative to network DDoS Protection in the settings of a public ip (Standard SKU only)

# Private Access to Azure Services
- Private Link Service provides private access to an otherwise not accessible network
  - Requires a load balancer to connect to
  - Access can be requested based on RBAC / anybody within subscription / anyone with alias
  - Approval needs to be given for connections from different subscriptions
  - Performs NATing to avoid address conflicts (NAT IPs can be configured)
- Private Endpoints can connect to the private link service, connection will work over the microsoft network
  - PEs can be placed in different subscription and region
  - PEs can connect to Azure PaaS solutions by default or to a private link service
  - NIC will be created, connected to PEs
  - Can be integrated with private DNS zones
  - Network policies
    - Two options: Route tables, NSGs -> Can be enabled on subnet level
    - To change routing for PEs (e.g. if routing should be done through firewall subnet)
    - Ability to use custom routes, NSGs to override system generated ones when creating PE
- Service endpoints extend virtual network to azure PaaS
  - Traffic is routed through Azure backbone
  - However service still requires a public IP address
  - Only supports services in the same region as the subnet
  - Service endpoint policies:
    - Restrict egress traffic to specific PaaS (e.g. a specific Storage account / all in a RG / all in a subscription)
    - Aliases can be defined
- App service VNet integration:
  - Can be integrated into VNet in the same region
  - Gateway-required VNet integration: Integration of a service into a VNet if the VNet is in a different region, requires VPN P2S
    - Route-based, SSTP, certificate upload is not required
  - Only a single service plan can use the subnet
  - App Service will use a private IP for communication within the VNet for outbound traffic
  - To receive also private inbound traffic, a private endpoint needs to be deployed

