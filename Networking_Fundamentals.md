# Lesson 8: AWS Networking (Part 1) — Fundamentals Explained Simply

*AWS Education Notes · Author: Jayesh Sharma*

## What you will learn

- IP addresses, public/private IPv4 ranges and CIDR calculations.
- DNS, TCP, UDP, ports, HTTP and HTTPS.
- VPCs, subnets, Availability Zones and route tables.
- Internet gateways, NAT gateways, security groups and network ACLs.
- Latency, bandwidth, proxies and load balancers.
- How to follow and troubleshoot a request end to end.

## 1. The big picture: what is networking?

A computer network allows devices and applications to exchange data. When you open https://example.com, several things happen:

1. Your device looks up the IP address associated with the domain name.
2. It establishes a network connection to the destination.
3. For HTTPS, it negotiates a secure TLS session.
4. It sends an HTTP request.
5. The server processes the request and returns an HTTP response.

~~~mermaid
flowchart LR
  B["Browser"] --> D["DNS lookup"]
  D --> IP["Destination IP"]
  IP --> C["TCP connection"]
  C --> T["TLS handshake"]
  T --> H["HTTP request"]
  H --> S["Application server"]
  S --> R["HTTP response"]
~~~

| Concept | Main responsibility |
|---|---|
| IP | Identifies a network interface/address |
| DNS | Resolves names to records such as IP addresses |
| TCP / UDP | Transport data between endpoints |
| Port | Identifies an application endpoint on a host |
| HTTP | Defines web request and response semantics |
| TLS | Protects data in transit and authenticates the server |
| Router / route table | Chooses where packets should be forwarded |
| Firewall | Allows or blocks traffic according to rules |

## 2. Everyday analogy

Imagine sending a parcel:

- **IP address:** the destination's street address.
- **Port:** the apartment or department inside the building.
- **DNS:** a directory that finds an address from a name.
- **Router:** a junction that chooses the next road.
- **Firewall:** a security guard who checks whether entry is allowed.
- **TCP:** a tracked delivery process that detects missing pieces and puts data in order.
- **UDP:** sending individual postcards without built-in delivery confirmation.
- **TLS:** a locked, authenticated package that protects its contents while travelling.

This is only a memory aid. Real networking is packet-based and governed by protocols and routing rules.

## 3. IP addresses

An **IP address** identifies a network interface so packets can be delivered to the right endpoint.

### IPv4

IPv4 uses 32 bits and is usually written as four decimal numbers separated by dots.

Example: **192.168.1.10**

Each part is 8 bits and can range from 0 to 255.

### Public vs private IPv4

- **Public IPv4:** globally unique and Internet-routable, subject to routing and security controls.
- **Private IPv4:** used inside private networks; it is not directly routed across the public Internet.

The three commonly used private IPv4 ranges are:

| Range | CIDR | Typical use |
|---|---|---|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | Large private networks |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | Medium private networks |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | Home and office networks |

**Easy example:** A public-facing website or load balancer may need a public endpoint so customers can reach it. Its application servers and database can use private IP addresses inside the VPC.

A private IP does not automatically make a resource secure. Routing, security groups, network ACLs and application authentication still matter.

### IPv4 vs IPv6

IPv6 uses 128 bits and has a much larger address space. Example: **2001:db8:abcd:12::10**. AWS VPCs can support IPv4, IPv6 or dual-stack designs, depending on service and configuration.

## 4. CIDR notation and subnet calculations

CIDR stands for **Classless Inter-Domain Routing**. It represents a network using an address and a prefix length.

Example: **10.0.1.0/24**

- IPv4 has 32 bits.
- /24 means 24 bits identify the network prefix.
- Remaining bits = 32 - 24 = 8.
- Total addresses = 2^8 = **256**.

### Easy CIDR examples

| CIDR | Host bits | Total IPv4 addresses |
|---|---:|---:|
| 10.0.0.0/16 | 16 | 65,536 |
| 10.0.1.0/24 | 8 | 256 |
| 10.0.1.0/25 | 7 | 128 |
| 10.0.1.0/26 | 6 | 64 |
| 10.0.1.0/27 | 5 | 32 |
| 10.0.1.0/28 | 4 | 16 |

**Interview shortcut:**

**Total IPv4 addresses = 2^(32 - prefix length)**

For example:
- /24: 2^(32 - 24) = 2^8 = **256** addresses.
- /26: 2^(32 - 26) = 2^6 = **64** addresses.
- /28: 2^(32 - 28) = 2^4 = **16** addresses.

Each time the prefix increases by 1, the address count is halved: /24 gives 256, /25 gives 128, /26 gives 64 and /27 gives 32.

**Memory rule:** A larger prefix means a smaller subnet. A /24 is larger than a /26.

### AWS subnet calculation: important difference

For an IPv4 subnet in an AWS VPC, AWS reserves **five IP addresses** in each subnet. The number available to resources is therefore the total minus five.

Example: **10.0.1.0/24**
- Total addresses = 2^8 = 256
- AWS-reserved addresses = 5
- Available to resources = 256 - 5 = **251**

For a /26:
- Total = 2^6 = 64
- Available in an AWS subnet = 64 - 5 = **59**

AWS reserves the first four IPv4 addresses and the last IPv4 address in every IPv4 subnet. For example, in 10.0.1.0/24, the reserved addresses are 10.0.1.0, 10.0.1.1, 10.0.1.2, 10.0.1.3 and 10.0.1.255. These reservations support VPC networking functions such as the network address, VPC router, DNS and future use.

| Subnet size | Total addresses | AWS-reserved | Available to resources |
|---|---:|---:|---:|
| /24 | 256 | 5 | 251 |
| /25 | 128 | 5 | 123 |
| /26 | 64 | 5 | 59 |
| /27 | 32 | 5 | 27 |
| /28 | 16 | 5 | 11 |

AWS IPv4 subnets have a minimum size of /28. Always distinguish the mathematical address count from the addresses available for resource assignment in an AWS subnet.

### Example VPC address plan

Suppose an application needs two Availability Zones, with public subnets for an internet-facing load balancer, private application subnets and a separate private database tier. A VPC such as 10.0.0.0/16 leaves room for growth.

| Subnet | CIDR | Purpose |
|---|---|---|
| Public AZ-A | 10.0.1.0/24 | Internet-facing load balancer / NAT Gateway |
| Public AZ-B | 10.0.2.0/24 | Internet-facing load balancer / NAT Gateway |
| Private App AZ-A | 10.0.11.0/24 | Application servers |
| Private App AZ-B | 10.0.12.0/24 | Application servers |
| Private DB AZ-A | 10.0.21.0/24 | Database tier |
| Private DB AZ-B | 10.0.22.0/24 | Database tier |

~~~mermaid
flowchart TB
  VPC["VPC: 10.0.0.0/16"]
  VPC --> AZA["Availability Zone A"]
  VPC --> AZB["Availability Zone B"]
  AZA --> PA["Public: 10.0.1.0/24"]
  AZA --> AA["Private App: 10.0.11.0/24"]
  AZA --> DA["Private DB: 10.0.21.0/24"]
  AZB --> PB["Public: 10.0.2.0/24"]
  AZB --> AB["Private App: 10.0.12.0/24"]
  AZB --> DB["Private DB: 10.0.22.0/24"]
~~~

**Why separate subnets?**
1. **Security and isolation:** Apply different route tables and network ACL policies to public, application and database tiers.
2. **Availability:** Place resources across multiple AZs so an AZ disruption does not take down the entire application.
3. **Management and growth:** Allocate address space by tier and leave unused ranges for future services.

**VPC planning rules:**
- Every subnet CIDR must be contained within the VPC CIDR.
- Subnet CIDRs in the same VPC must not overlap.
- Estimate resource counts and leave room for scaling, replacement resources and managed services.
- Use multiple AZs for production high availability.
- Keep application and database resources private unless a specific requirement justifies public exposure.
- Avoid overlapping CIDRs with networks you may later connect through VPC peering, Transit Gateway or VPN.

## 5. OSI and TCP/IP models

Networking is easier to debug when you understand the layers.

### OSI model: seven layers

| Layer | Name | Simple responsibility | Examples |
|---:|---|---|---|
| 7 | Application | Application-level communication | HTTP, DNS |
| 6 | Presentation | Data representation and encryption concepts | Encoding, TLS concepts |
| 5 | Session | Managing communication sessions | Session concepts |
| 4 | Transport | Delivery between processes | TCP, UDP |
| 3 | Network | Addressing and routing packets | IP |
| 2 | Data Link | Communication on the local link | Ethernet, MAC |
| 1 | Physical | Signals over a medium | Cable, radio |

The OSI model is a conceptual learning model. Real protocol stacks do not always map one-to-one to these seven layers.

### TCP/IP model

| Layer | Examples |
|---|---|
| Application | HTTP, DNS, SSH |
| Transport | TCP, UDP |
| Internet | IPv4, IPv6 |
| Link / Network access | Ethernet, Wi-Fi |

**Memory trick:** Application → Transport → Internet → Link.

## 6. DNS: converting a name into an address

DNS stands for **Domain Name System**. It lets applications look up records for names such as api.example.com.

| Record | Purpose | Example use |
|---|---|---|
| A | Maps a name to an IPv4 address | app.example.com |
| AAAA | Maps a name to an IPv6 address | IPv6 endpoint |
| CNAME | Alias to another DNS name | www alias |
| MX | Mail server routing | Email delivery |
| TXT | Text data, often for verification | Domain ownership |
| NS | Name servers for a zone | DNS delegation |

~~~mermaid
sequenceDiagram
  participant B as Browser
  participant R as Recursive resolver
  participant D as DNS hierarchy
  participant A as Authoritative DNS
  B->>R: Resolve api.example.com
  R->>D: Find authoritative zone
  D-->>R: Referral to authoritative DNS
  R->>A: Query record
  A-->>R: Return DNS answer
  R-->>B: Return answer (may be cached)
~~~

Caches may satisfy a lookup without repeating the full hierarchy. DNS resolution is not the same as connecting to the returned address: the TCP/TLS connection can still fail.

### DNS in AWS

Amazon Route 53 is AWS's DNS service. It supports public and private hosted zones and routing policies such as simple, weighted, latency-based and failover routing.

A **private hosted zone** is associated with VPCs and resolves names within the associated private DNS context. A public hosted zone serves DNS records for Internet-facing domain names.

## 7. TCP, UDP and ports

### TCP

TCP is a connection-oriented transport protocol. It provides reliable delivery through acknowledgements and retransmission, ordered delivery of bytes, flow control and congestion control.

A simplified TCP three-way handshake:

~~~mermaid
sequenceDiagram
  participant C as Client
  participant S as Server
  C->>S: SYN
  S-->>C: SYN-ACK
  C->>S: ACK
  Note over C,S: TCP connection established
~~~

TCP provides a **byte stream**, not application message boundaries. Applications must define how messages are framed.

### UDP

UDP is connectionless and sends independent datagrams. It does not provide built-in retransmission, ordering or delivery guarantees. It is useful when low overhead or application-controlled recovery is important. Examples include DNS queries, voice/video traffic and some real-time protocols. Some DNS traffic also uses TCP.

### TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| Connection setup | Yes | No |
| Built-in reliability | Yes | No |
| Ordering | Yes | No |
| Data model | Byte stream | Datagrams |
| Typical examples | HTTPS, SSH, database connections | DNS queries, voice/video, real-time traffic |

### What is a port?

An IP address identifies the host/interface; a port identifies the application endpoint on that host.

| Port | Protocol / service |
|---:|---|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 8080 | Common alternate HTTP application port |

A port number alone does not guarantee that a service is listening or reachable. The application, host firewall, AWS security group and network route must all permit the connection.

## 8. HTTP vs HTTPS

HTTP is an application protocol used for web requests and responses. HTTPS is HTTP protected by TLS.

~~~mermaid
sequenceDiagram
  participant C as Client
  participant W as Web server
  C->>W: GET /products
  W-->>C: 200 OK + response body
~~~

Common HTTP status codes:

- **200 OK:** request succeeded.
- **301 / 302:** redirect.
- **400 Bad Request:** request is invalid.
- **401 Unauthorized:** authentication is required or invalid.
- **403 Forbidden:** request is understood but not allowed.
- **404 Not Found:** resource was not found.
- **500 Internal Server Error:** server-side failure.
- **503 Service Unavailable:** service is temporarily unable to handle the request.

### What HTTPS adds

TLS provides:
1. **Encryption:** helps prevent others from reading traffic in transit.
2. **Integrity:** helps detect tampering.
3. **Server authentication:** certificates help the client verify the server's identity.

HTTPS does not automatically make an application secure. Authentication, authorization, input validation, secure configuration and safe handling of secrets are still necessary.

## 9. Routing: how packets find their next hop

A router examines a packet's destination IP and uses a routing table to decide where to forward it. The most specific matching route is generally selected (longest-prefix match).

In AWS, a VPC route table contains destination CIDR blocks and targets such as:
- Local VPC route
- Internet Gateway
- NAT Gateway
- Transit Gateway
- Virtual private gateway

Example route table for a public subnet:

| Destination | Target | Meaning |
|---|---|---|
| 10.0.0.0/16 | local | Route within the VPC |
| 0.0.0.0/0 | Internet Gateway | Default route toward the Internet |

A route to an Internet Gateway alone does not make a private instance publicly reachable. The instance also needs suitable public IPv4 addressing (or IPv6 setup), and security rules must allow the traffic.

## 10. Latency, bandwidth and throughput

- **Latency:** time taken for data to travel between endpoints; often measured as round-trip time (RTT).
- **Bandwidth:** maximum theoretical data-carrying capacity of a link, often measured in bits per second.
- **Throughput:** actual rate of successful data transfer.
- **Jitter:** variation in packet delay.
- **Packet loss:** packets that fail to reach their destination.

### Easy numerical example

Suppose a network link has a bandwidth of **100 Mbps**.

- 100 Mbps means up to 100 million bits per second under ideal conditions.
- 1 byte = 8 bits.
- The theoretical byte rate is 100 / 8 = **12.5 MB/s** (decimal units).
- A 100 MB file would take at least 100 / 12.5 = **8 seconds** in this simplified ideal calculation.

Real transfer time is usually longer because of protocol overhead, congestion, latency, server limits and other users sharing the link.

**Interview point:** High bandwidth does not guarantee low latency. A link can transfer a lot of data per second and still have a long round-trip delay.

## 11. AWS VPC: your private network in AWS

A **Virtual Private Cloud (VPC)** is a logically isolated virtual network in an AWS Region. You choose its IP address range and configure subnets, routing and network controls.

### Key building blocks

- **VPC:** the overall virtual network.
- **Subnet:** a range of IP addresses inside a VPC.
- **Availability Zone (AZ):** an isolated location within a Region.
- **Route table:** determines where subnet traffic is sent.
- **Internet Gateway (IGW):** enables Internet connectivity for eligible VPC resources with suitable routes and addressing.
- **NAT Gateway:** allows resources in private subnets to initiate connections to external destinations without accepting unsolicited inbound connections through that NAT path.
- **Security Group:** stateful virtual firewall associated with supported resources/network interfaces.
- **Network ACL (NACL):** stateless subnet-level traffic filter.

### VPC and subnet architecture

~~~mermaid
flowchart TB
  INTERNET["Internet"]
  IGW["Internet Gateway"]
  subgraph VPC["VPC: 10.0.0.0/16"]
    subgraph AZA["Availability Zone A"]
      PA["Public subnet 10.0.1.0/24"]
      PRA["Private subnet 10.0.11.0/24"]
      LB["Public load balancer node"]
      APPA["Private application"]
      PA --> LB
      PRA --> APPA
    end
    subgraph AZB["Availability Zone B"]
      PB["Public subnet 10.0.2.0/24"]
      PRB["Private subnet 10.0.12.0/24"]
      APPB["Private application"]
      PRB --> APPB
    end
    NAT["NAT Gateway (public subnet)"]
  end
  INTERNET <--> IGW
  IGW <--> PA
  IGW <--> PB
  PRA --> NAT
  PRB --> NAT
~~~

This is a conceptual layout. A production design should consider NAT gateway placement per AZ, resilience, routing, cost and the specific load balancer architecture.

### Public subnet vs private subnet

A subnet is commonly called **public** when its route table has a route to an Internet Gateway. A subnet is commonly called **private** when it does not have a direct route to an Internet Gateway.

| Public subnet | Private subnet |
|---|---|
| Route to IGW | No direct default route to IGW |
| Often hosts internet-facing load balancer nodes or NAT gateway | Often hosts application servers and databases |
| Resources need suitable public addressing and security rules for inbound Internet access | May use NAT or VPC endpoints for outbound access |

The subnet label alone does not control reachability; route tables, addressing and security controls work together.

### Internet Gateway vs NAT Gateway

| Feature | Internet Gateway | NAT Gateway |
|---|---|---|
| Main purpose | Internet connectivity for eligible VPC resources | Outbound connectivity from private subnet resources |
| Typical placement | Attached to VPC | Created in a subnet, commonly public |
| Inbound Internet connections | Possible when routing, addressing and security rules allow | Not through the NAT gateway's outbound translation path |
| Common use | Public-facing endpoint | Private instance downloads updates or calls external APIs |

A NAT Gateway is not a firewall replacement. Security groups and NACLs still control traffic.

### Security Group vs Network ACL

| Feature | Security Group | Network ACL |
|---|---|---|
| Scope | Resource/network interface | Subnet |
| Stateful? | Yes | No (stateless) |
| Rules | Allow rules | Allow and deny rules |
| Rule evaluation | All applicable rules are considered | Rules evaluated in ascending rule-number order |
| Return traffic | Automatically allowed for tracked connections | Must be explicitly allowed in the opposite direction |

**Example:** An application server security group might allow inbound TCP 8080 only from the load balancer security group. It should not expose port 8080 to the whole Internet without a clear reason.

## 12. NAT, proxies and load balancers

### NAT (Network Address Translation)

NAT translates network addresses, often allowing many private hosts to share an external address for outbound connectivity.

In AWS, a NAT Gateway is commonly used so instances in private subnets can initiate IPv4 connections to the Internet while remaining unreachable through unsolicited inbound Internet connections via that NAT gateway.

### Forward proxy vs reverse proxy

- **Forward proxy:** acts on behalf of clients when they access external services. It can provide policy enforcement, filtering or controlled egress.
- **Reverse proxy:** sits in front of servers and receives requests on their behalf. It can route requests, terminate TLS and provide caching or other application-layer features.

### Load balancer

A load balancer distributes incoming requests across healthy targets.

~~~mermaid
flowchart LR
  CLIENT["Clients"] --> LB["Application Load Balancer"]
  LB --> A["Application instance A"]
  LB --> B["Application instance B"]
  LB --> C["Application instance C"]
  A --> DB["Database"]
  B --> DB
  C --> DB
~~~

In AWS:
- **Application Load Balancer (ALB):** Layer 7 HTTP/HTTPS routing, including host- and path-based routing.
- **Network Load Balancer (NLB):** Layer 4 load balancing for TCP, UDP and TLS use cases.
- **Gateway Load Balancer (GWLB):** helps deploy, scale and manage virtual network appliances.

Choose based on protocol, routing needs, performance, client behavior, availability and cost.

## 13. How a request reaches an AWS application

Consider a user opening **https://shop.example.com**.

1. DNS resolves the domain to the public endpoint.
2. The client connects to the endpoint using TCP (for typical HTTPS over TCP).
3. TLS establishes a secure session.
4. The load balancer accepts the request.
5. The load balancer forwards it to a healthy application target.
6. The application communicates with its database or other dependencies.
7. The response returns through the established connections.

~~~mermaid
sequenceDiagram
  participant U as User browser
  participant DNS as Route 53 / DNS
  participant LB as Public load balancer
  participant APP as Private application
  participant DB as Database
  U->>DNS: Resolve shop.example.com
  DNS-->>U: Endpoint address
  U->>LB: HTTPS request
  LB->>APP: Forward request to healthy target
  APP->>DB: Query required data
  DB-->>APP: Return data
  APP-->>LB: Application response
  LB-->>U: HTTPS response
~~~

The exact DNS target and network path depend on the load balancer type, VPC configuration and whether the application is public or private.

## 14. Practical exercise: reason about a VPC design

Before creating resources, draw and review this small design:

- One VPC: **10.0.0.0/16**.
- Two Availability Zones.
- One public, one private application and one private database subnet per AZ.
- A public load balancer across the public subnets.
- Application instances in the private application subnets.
- A database tier in private database subnets, not publicly accessible.
- Explicit security group rules for each application-to-application connection.

### Suggested traffic rules

| Source | Destination | Port | Reason |
|---|---|---:|---|
| Internet clients | Public load balancer | TCP 443 | HTTPS entry point |
| Load balancer SG | Application SG | TCP 8080 | Forward application requests |
| Application SG | Database SG | TCP 5432 | PostgreSQL connection |
| Other Internet sources | Database | None | No direct public database access |

The ports are examples. Use the actual ports and protocols configured by your application.

### Safe lab checklist

1. Use a sandbox AWS account and enable MFA.
2. Review current pricing for every resource before deployment.
3. Create a small VPC and subnets only if you understand the costs.
4. Inspect route tables, security groups and NACLs.
5. Test reachability from the intended source and verify that unintended paths are blocked.
6. Delete resources when finished and confirm that billable resources are gone.

**Cost warning:** NAT Gateways, load balancers, public IPv4 addresses, data transfer and other networking resources can incur charges. A short lab can still cost money. Check current AWS pricing and Free Tier terms first.

## 15. Troubleshooting network connectivity

When an application cannot connect, debug from the outside inward rather than changing many settings at once.

1. **Name resolution:** Does DNS return the expected record?
2. **Destination:** Is the IP address and port correct?
3. **Route:** Is there a route from source to destination and a valid return path?
4. **Security Group:** Does the destination allow the required inbound traffic? Does the source allow required outbound traffic?
5. **NACL:** Are inbound and outbound flows permitted, including return/ephemeral ports where required?
6. **Listening service:** Is the application actually listening on the expected address and port?
7. **Application layer:** Is the server returning an HTTP error, TLS error or application exception?
8. **Evidence:** Inspect application logs, load balancer health checks, VPC Flow Logs and relevant service metrics.

~~~mermaid
flowchart TD
  START["Connection failed"] --> DNSQ{"DNS resolves correctly?"}
  DNSQ -->|No| DNSFIX["Check DNS record, zone and cache"]
  DNSQ -->|Yes| ROUTEQ{"Route and return path valid?"}
  ROUTEQ -->|No| ROUTEFIX["Check route tables, IGW, NAT and gateways"]
  ROUTEQ -->|Yes| SECQ{"Security rules allow traffic?"}
  SECQ -->|No| SECFIX["Check SG and NACL rules"]
  SECQ -->|Yes| APPQ{"Service listening and healthy?"}
  APPQ -->|No| APPFIX["Check process, port, target health and logs"]
  APPQ -->|Yes| TLSQ["Inspect TLS and application response"]
~~~

### Useful AWS troubleshooting tools

- **VPC Flow Logs:** capture information about IP traffic to and from network interfaces, subnets or VPCs.
- **Reachability Analyzer:** helps analyze whether a path is possible between supported network resources.
- **CloudWatch:** provides metrics and logs for supported AWS services.
- **Load balancer target health:** helps identify unhealthy application targets.
- **Route 53 query logging and DNS tools:** help investigate DNS behavior.

Flow Logs showing rejected traffic are useful evidence, but Flow Logs do not capture every detail of application behavior or prove that an application is healthy.

## 16. Common mistakes to avoid

| Mistake | Correct understanding |
|---|---|
| A public subnet makes every resource public | Public reachability also depends on addressing, routes and security rules |
| A private IP guarantees security | Private addressing is not a substitute for access controls |
| DNS success means the server is reachable | DNS lookup and network connection are separate steps |
| Security groups and NACLs are identical | Security groups are stateful and resource-associated; NACLs are stateless and subnet-associated |
| NAT Gateway allows inbound Internet access to private instances | NAT Gateway is primarily for outbound-initiated connectivity |
| More bandwidth always means lower latency | Bandwidth and latency measure different things |
| HTTPS means the application is fully secure | TLS protects traffic in transit; application security is still required |

## 17. Interview questions

1. What is an IP address? How are public and private IP addresses different?
2. What does 10.0.0.0/16 mean? How many IPv4 addresses does it contain?
3. How do you calculate the number of addresses in a CIDR block?
4. How many addresses are available for resources in AWS /24, /26 and /28 subnets?
4. What is the difference between a VPC and a subnet?
5. What makes a subnet public or private in AWS?
6. What is the role of an Internet Gateway?
7. Why would an application in a private subnet need a NAT Gateway?
8. What is the difference between a NAT Gateway and an Internet Gateway?
9. How are security groups different from network ACLs?
10. Explain the TCP three-way handshake.
11. What is the difference between TCP and UDP?
12. What does DNS do? What happens after DNS resolution?
13. What does HTTPS add over HTTP?
14. Explain latency, bandwidth and throughput with an example.
15. What is the difference between a forward proxy and a reverse proxy?
16. When would you choose an ALB instead of an NLB?
17. How would you troubleshoot an application that is not reachable?

## 18. Quick revision sheet

| Term | Remember it as |
|---|---|
| IP address | Address of a network interface |
| CIDR | Network prefix and address range; total IPv4 addresses = 2^(32 - prefix) |
| /24 | 256 total IPv4 addresses; 251 available in an AWS subnet |
| /26 | 64 total IPv4 addresses; 59 available in an AWS subnet |
| AWS subnet reservation | Five IPv4 addresses per subnet |
| Subnet | Smaller IP range inside a VPC |
| DNS | Name-to-record lookup |
| TCP | Reliable, ordered byte stream |
| UDP | Datagrams without built-in delivery guarantee |
| Port | Application endpoint on a host |
| HTTP | Web request/response protocol |
| HTTPS | HTTP protected by TLS |
| Route table | Where packets should go next |
| Internet Gateway | VPC connection to the Internet |
| NAT Gateway | Outbound IPv4 access for private subnet resources |
| Security Group | Stateful resource-level virtual firewall |
| Network ACL | Stateless subnet-level traffic filter |
| Latency | Time delay |
| Bandwidth | Potential data capacity per second |
| Throughput | Actual data transfer rate |
| Load balancer | Distributes traffic across targets |

### One-minute explanation

Computer networking enables applications to exchange data using addresses, protocols and routes. DNS resolves a name to a record; IP provides addressing; TCP or UDP transports data; ports identify application endpoints; and HTTP/HTTPS defines web communication. In AWS, a VPC provides an isolated network, subnets divide its address space, route tables control forwarding, gateways provide connectivity, and security groups and NACLs filter traffic. To troubleshoot a request, check DNS, addressing, routes, security rules, listening services and application logs in that order.

## Useful references

- [AWS VPC documentation](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [VPC subnet sizing and reserved IPv4 addresses](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html)
- [Internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [AWS from Basics to Advanced course playlist](https://www.youtube.com/playlist?list=PL6W8uoQQ2c60g3PAgzVPpi0YWwxITD43p)
