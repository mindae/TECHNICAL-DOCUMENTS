# Networking for Developers & Architects

> A practical, evolving field guide to networking for Java/Spring Boot developers working with AWS, Docker, Kubernetes, HTTPS, certificates, and application security.
>
> **Learning philosophy:** Understand the concepts and traffic flows; use tools for subnet arithmetic. Learn breadth first, then revisit important topics in more depth. CLI first, followed by application and architecture examples.

**Last updated:** 2026-10-16  
**Status:** Living notes — Lessons 1–4 captured; next lesson planned: routing, Internet Gateway, and NAT Gateway.

## Contents

1. [Learning roadmap](#1-learning-roadmap)
2. [Start with localhost, IP addresses, and ports](#2-start-with-localhost-ip-addresses-and-ports)
3. [CIDR: what `10.0.0.0/16` means](#3-cidr-what-1000016-means)
4. [VPCs and subnets](#4-vpcs-and-subnets)
5. [End-to-end request flow](#5-end-to-end-request-flow)
6. [Networking and application security](#6-networking-and-application-security)
7. [Hands-on Mac networking lab](#7-hands-on-mac-networking-lab)
8. [CIDR calculators and utilities](#8-cidr-calculators-and-utilities)
9. [Common misunderstandings](#9-common-misunderstandings)
10. [Knowledge check](#10-knowledge-check)
11. [Next lessons and progress log](#11-next-lessons-and-progress-log)
12. [Reference links](#12-reference-links)

---

## 1. Learning roadmap

The goal is to confidently interpret an architecture request such as:

> Create a VPC `10.0.0.0/16`, public and private subnets across two Availability Zones, an internet-facing ALB, Spring Boot services on EKS, security groups, Route 53 DNS, and an ACM TLS certificate.

We will progress through these areas:

| Stage | Topics | Outcome |
|---|---|---|
| 1. Foundations | IP address, localhost, ports, CIDR, TCP/UDP | Understand what identifies a host/network endpoint |
| 2. Network movement | Subnets, route tables, gateways, NAT, DNS | Explain how traffic reaches another network |
| 3. AWS | VPC, Availability Zones, internet/public/private networking, security groups, ALB | Understand a typical AWS deployment |
| 4. Docker & Kubernetes | Container/Pod IPs, ClusterIP, NodePort, LoadBalancer, Ingress/Gateway API | Trace requests from ingress to Pods |
| 5. Secure communications | HTTPS, TLS, certificates, CA, truststore/keystore, mTLS, OAuth2/JWT | Separate transport security from API authorization |
| 6. Production troubleshooting | Connection timeout/refused, DNS, routing, 502/503, certificate errors | Diagnose failures layer by layer |

**Approach:** We will **not** spend extensive time converting IPs to binary or calculating network/broadcast addresses by hand. Learn the meaning of CIDR prefixes, then use calculators and CLI tools for the actual values. Recognizing overlapping address ranges is useful, but tools can verify it.

---

## 2. Start with localhost, IP addresses, and ports

A familiar Spring Boot request:

```text
http://localhost:8081/api/v1/employees/101
```

| Part | Meaning |
|---|---|
| `http` | Application protocol |
| `localhost` | Hostname referring to the current host/network namespace |
| `8081` | Listening TCP port in this example |
| `/api/v1/employees/101` | HTTP path |

### Loopback versus LAN address

- `localhost` commonly resolves to `127.0.0.1` (IPv4) or `::1` (IPv6).
- `127.0.0.1` is an IPv4 loopback address: traffic stays within the host's networking environment.
- A Mac may also have a private LAN address such as `192.168.1.10` assigned by its Wi-Fi network.
- IP addresses identify network interfaces/locations; ports identify transport endpoints on them.

Example home network (illustrative values):

```text
                    Home Wi-Fi network
                  Router: 192.168.1.1
                           |
                  +--------+--------+
                  |                 |
            Your MacBook       Other device
            192.168.1.10      192.168.1.20
```

**Critical Docker/Kubernetes point:** `localhost` means *this network namespace*. Inside a container or Pod it does **not** automatically point to your MacBook or to another Pod. Containers in the same Kubernetes Pod usually share a network namespace and can therefore communicate over `localhost`; separate Pods cannot assume this.

### Ports

The same host can run multiple apps on different ports:

```text
192.168.1.10:8081  -> Employee Service
192.168.1.10:8082  -> Department Service
192.168.1.10:9090  -> API Gateway
```

Frequently seen ports:

| Port | Common use |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 5432 | PostgreSQL |
| 3306 | MySQL |
| 8080 / 8081 / 8082 | Common developer-selected web app ports |
| 9090 | Often a custom API Gateway port (not standardized for that purpose) |
| 6443 | Kubernetes API server |

Ports are conventions/configurations, not permanent bindings to particular applications. A listening port does **not** imply internet reachability; routing, bind addresses, firewalls, and policy matter.

---

## 3. CIDR: what `10.0.0.0/16` means

**CIDR** = Classless Inter-Domain Routing. IPv4 addresses have **32 bits**. In `10.0.0.0/16`, the `/16` means the first 16 bits form the fixed **network prefix**, with 16 bits available for addresses within the block.

```text
IPv4 address:     10       . 0        . 0        . 0
                 [8 bits]  [8 bits]   [8 bits]   [8 bits]
                 +---- network ----+  +--- variable ---+
                       16 bits           16 bits

CIDR: 10.0.0.0/16
Address range: 10.0.0.0 through 10.0.255.255
Total IPv4 addresses: 2^(32-16) = 65,536
```

| Property | `10.0.0.0/16` |
|---|---|
| Network address | `10.0.0.0` |
| Subnet mask | `255.255.0.0` |
| Last address (traditional broadcast address) | `10.0.255.255` |
| Total addresses | 65,536 |
| Traditional host range on a conventional IPv4 subnet | `10.0.0.1` – `10.0.255.254` |

**Intuition:** A **smaller CIDR prefix number means a larger block**. A `/16` contains more addresses than a `/24`.

| CIDR | Total addresses | Example range |
|---|---:|---|
| `10.0.0.0/8` | 16,777,216 | `10.0.0.0` – `10.255.255.255` |
| `10.0.0.0/16` | 65,536 | `10.0.0.0` – `10.0.255.255` |
| `10.0.0.0/24` | 256 | `10.0.0.0` – `10.0.0.255` |
| `10.0.0.0/28` | 16 | `10.0.0.0` – `10.0.0.15` |

**Do not confuse counts:** CIDR describes address space, not the number of running servers. In AWS, the first four and last IPv4 addresses in every subnet are reserved: an AWS `/24` therefore offers **251 assignable IPv4 addresses**, even though the block contains 256.

### Private versus public IP addresses

Private IPv4 ranges commonly used inside organizations/cloud networks (RFC 1918):

- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

These addresses are not globally routed on the public internet. They can be reused in separate private networks; connecting those networks may create overlap problems. Public IP addresses are globally routable when allocated and routing/policies permit.

---

## 4. VPCs and subnets

**VPC = Virtual Private Cloud:** a logically isolated virtual network within AWS. If a VPC has a primary IPv4 CIDR `10.0.0.0/16`, you can carve that address space into **non-overlapping subnets**.

```text
+------------------------- AWS VPC 10.0.0.0/16 -------------------------+
|                                                                      |
|  Availability Zone A                   Availability Zone B            |
|  +------------------------+            +------------------------+    |
|  | Public subnet A        |            | Public subnet B        |    |
|  | 10.0.1.0/24            |            | 10.0.2.0/24            |    |
|  +------------------------+            +------------------------+    |
|                                                                      |
|  +------------------------+            +------------------------+    |
|  | Private app subnet A   |            | Private app subnet B   |    |
|  | 10.0.11.0/24           |            | 10.0.12.0/24           |    |
|  +------------------------+            +------------------------+    |
|                                                                      |
+----------------------------------------------------------------------+
```

- A **subnet** is an address range within a VPC, associated with **one Availability Zone**.
- Public and private are **routing properties**, not properties of the IP numbers themselves.
- A typical **public subnet** has an IPv4 route to an **Internet Gateway (IGW)**.
- A typical **private application subnet** has **no direct route to the IGW**; it may route outbound internet access through a **NAT Gateway** in a public subnet.
- Merely being in a public subnet does **not** make a resource public: it also needs appropriate addressing, routes, and access rules.
- AWS security groups and network ACLs further control traffic.

**Containment example:** `10.0.1.25` belongs to subnet `10.0.1.0/24`, which belongs to VPC CIDR `10.0.0.0/16`.

**Architect implication:** When choosing VPC/subnet ranges, leave capacity for growth and avoid overlap with VPN/on-premises/other VPC networks that may later need to connect.

---

## 5. End-to-end request flow

Example request:

```text
GET https://api.example.com/api/v1/employees/101
```

Conceptual deployment (not the only valid EKS setup):

```text
User's browser
     |
     | DNS resolves api.example.com (e.g., Route 53 record)
     v
Public Application Load Balancer (ALB) :443
     | TLS certificate (possibly AWS Certificate Manager)
     | ALB applies listener / target-group rules
     v
Private application network (EKS or EC2)
     |
     +--> Optional ingress controller / application API Gateway :9090
                  |
                  +--> Employee Service :8081
                  |          |
                  |          +--> RDS/PostgreSQL :5432
                  |
                  +--> Department Service :8082
```

**Important precision:** DNS is a *name-resolution step*, not an HTTP forwarding proxy. Also, the ALB can be integrated directly with Kubernetes service/Pod targets (depending on the AWS load balancer controller and target type); an extra gateway is optional, not mandatory.

Request sequence:

1. The client resolves `api.example.com` using DNS.
2. The client establishes a TCP connection to the destination and negotiates HTTPS/TLS, typically on **443**.
3. The ALB receives the request; its listener and target-group rules choose a healthy backend target.
4. Depending on design, traffic reaches an ingress/gateway or directly reaches a Kubernetes service/Pod target.
5. The Spring Boot Employee Service processes the request, optionally calls Department Service and queries a database, then responds.

### Why the external port differs from the app port

```text
Client  -- HTTPS :443 --> ALB
ALB     -- HTTP  :9090 -> API Gateway (optional)
Gateway -- HTTP  :8081 -> Employee Service
```

These are **different connections**, which may use different protocols and ports. Internal connections can also use HTTPS if required.

| Component | Main purpose |
|---|---|
| DNS / Route 53 | Resolve names to destinations |
| ALB | Receive and distribute HTTP(S) requests across targets |
| API Gateway | Apply API-specific routing and policies; may integrate with authentication/rate limiting |
| Kubernetes Service | Stable logical access to changing Pod endpoints |
| Spring Boot app | Process business logic and enforce application authorization |

---

## 6. Networking and application security

Example **intended** connectivity policy:

| Source | Destination | Port | Allow? |
|---|---|---:|---|
| Internet | Public ALB | 443 | Yes |
| ALB | Gateway/backend | 9090 or backend port | Yes, restricted to necessary targets |
| Gateway | Employee Service | 8081 | Yes, if needed |
| Employee Service | Database | 5432 | Yes, constrained |
| Internet | Database | 5432 | No |

Controls at different layers:

- **Security group:** stateful allow rules controlling traffic at supported AWS network interfaces/resources.
- **Network ACL (NACL):** stateless subnet-level allow/deny rules; return traffic must be accounted for explicitly.
- **TLS/HTTPS:** encrypts traffic and authenticates a server endpoint using certificates and trust chains (in standard server-authenticated TLS).
- **JWT / Spring Security:** enforces authentication and authorization for API operations.
- **mTLS:** mutually authenticated TLS, where both endpoints provide certificates, when configured.

For example, a request may *reach* the Employee Service but still be rejected due to missing privileges:

```java
@PreAuthorize("hasAuthority('SCOPE_employees.read')")
@GetMapping("/{id}")
public Employee getEmployee(@PathVariable Long id) {
    return employeeService.findById(id);
}
```

`@PreAuthorize` is a Spring Security method-authorization annotation; the required authority is `SCOPE_employees.read`. It requires method security to be enabled and relevant authentication/authority mapping to be configured.

**Principle:** Network reachability does not imply API authorization. A successful login likewise does not automatically secure every API endpoint.

---

## 7. Hands-on Mac networking lab

Run these at home in macOS Terminal. Outputs vary by host configuration. Commands are meant for observation, not for modifying network settings.

### Lab A — Find your LAN IP

```bash
ipconfig getifaddr en0
```

- `ipconfig`: macOS network utility.
- `getifaddr`: show an interface's IPv4 address.
- `en0`: often Wi-Fi on Mac; verify your interface if this returns nothing.

Illustrative output: `192.168.1.10`.

### Lab B — Test local loopback

```bash
ping -c 3 127.0.0.1
```

- `ping`: sends ICMP echo requests.
- `-c 3`: send three probes.

ICMP does not use the Spring Boot application's TCP port `8081`.

### Lab C — Inspect Spring Boot's listening socket

With an app running on port 8081:

```bash
lsof -nP -iTCP:8081 -sTCP:LISTEN
```

- `lsof`: list open files, including sockets.
- `-n`: no DNS name lookup.
- `-P`: keep port numbers numeric.
- `-iTCP:8081`: filter TCP port 8081.
- `-sTCP:LISTEN`: select listening TCP sockets.

You might see `java` and a listening address.

**Bind address difference:**

```text
127.0.0.1:8081 -> Listen only on loopback IPv4 interface.
0.0.0.0:8081   -> Listen on all local IPv4 interfaces.
```

`0.0.0.0` is a *wildcard bind address* in this context; it is not an internet destination to call. Container publishing/network policy must be correctly configured in addition to the app bind address.

### Lab D — DNS lookup and HTTPS response

```bash
dig +short example.com
curl -I https://example.com
```

- `dig`: query DNS.
- `+short`: show concise DNS results.
- `curl`: HTTP client.
- `-I`: request response headers (HTTP HEAD).

### Optional: compare loopback and LAN connectivity

With your Spring Boot service running and configured to listen on the appropriate address, test from the Mac:

```bash
curl http://127.0.0.1:8081/api/v1/employees/101
curl http://<YOUR_MAC_LAN_IP>:8081/api/v1/employees/101
```

Replace `<YOUR_MAC_LAN_IP>` with your actual LAN address. LAN reachability also depends on local firewall/security settings.

---

## 8. CIDR calculators and utilities

We will **use tools**, not manually derive subnet masks and broadcast addresses every time.

### Option A — Visual subnet calculator

- [Visual Subnet Calculator](https://www.davidc.net/sites/default/subnets/subnets.html) — helpful when planning subnet subdivisions.

### Option B — `ipcalc` in Terminal

```bash
brew install ipcalc
ipcalc 10.0.0.0/16
```

- `brew install`: install a package with Homebrew.
- `ipcalc`: calculate network/subnet details from an address and prefix.

### Option C — Python built-in `ipaddress` (no pip installation)

```python
import ipaddress

network = ipaddress.ip_network("10.0.0.0/16")
print("Network:", network.network_address)
print("Mask:", network.netmask)
print("Broadcast:", network.broadcast_address)
print("Total addresses:", network.num_addresses)

subnet = ipaddress.ip_network("10.0.1.0/24")
address = ipaddress.ip_address("10.0.1.25")
print("Belongs to subnet:", address in subnet)

other_subnet = ipaddress.ip_network("10.0.2.0/24")
print("Overlaps:", subnet.overlaps(other_subnet))
```

This is particularly useful for architecture validation and scripting. Remember that `ipaddress` reports *general IPv4 network properties*; it does not automatically apply AWS's five-reserved-addresses-per-subnet convention to the counts.

### What to know versus what to delegate

| Know conceptually | Let a tool calculate/check |
|---|---|
| `/16` is larger than `/24` | Exact mask and first/last address |
| A subnet is contained in a larger CIDR range | Exact usable host/address counts |
| Overlapping networks can make connectivity difficult | Whether two arbitrary CIDRs overlap |
| Private/public is determined by routing/access | Valid subnet partitioning and free capacity |

---

## 9. Common misunderstandings

| Statement | Correction |
|---|---|
| “`10.0.0.0/16` means 65,536 running servers.” | No, it describes an address block; usable/assignable counts differ. |
| “My app uses `localhost`, so another Pod can call that address.” | No; `localhost` means the caller's own network namespace. |
| “Port 8081 is open, so the internet can reach it.” | Not necessarily; bind addresses, routes, public addressing, firewalls and security rules also matter. |
| “Public subnet means every instance is public.” | No; routing plus addressing and security controls determine connectivity. |
| “The application has to use port 443 because users call HTTPS.” | No; ALB can terminate TLS on 443 and forward on a different backend port. |
| “Certificate validation gives users API permissions.” | No; TLS and API authorization solve different problems. |
| “DNS forwards my HTTP request to the server.” | No; DNS resolves the name, and the client then connects to the resolved destination. |

---

## 10. Knowledge check

Try answering these without looking up the answers:

1. What does `/16` mean in `10.0.0.0/16`?
2. What does port `8081` identify?
3. Does `10.0.1.25` belong to subnet `10.0.1.0/24`?
4. Does a Spring Boot app have to listen on `443` for users to access it over HTTPS?
5. Why doesn't `localhost` inside one Kubernetes Pod normally identify a different Pod?
6. What is the difference between an internet-facing ALB and a private database?

<details>
<summary>Answers</summary>

1. A 16-bit IPv4 network prefix, leaving 16 bits within the address block.
2. A transport port used by a listening application endpoint.
3. Yes.
4. No; a load balancer can listen on 443 and forward to a backend port such as 8081.
5. Loopback points to that Pod's network namespace, not another Pod's namespace.
6. The ALB is reachable using public networking/configured rules, while the database should be on privately routed networks with tightly restricted access.

</details>

---

## 11. Next lessons and progress log

### Planned next lesson

**Subnets, route tables, Internet Gateway, and NAT Gateway**, using the same example VPC `10.0.0.0/16`:

- What happens when a resource in `10.0.11.0/24` calls an internet endpoint?
- What does a route such as `0.0.0.0/0` actually mean?
- Why does a public subnet need an Internet Gateway?
- Why and when would a private subnet use a NAT Gateway?
- How to inspect networking/routing safely with CLI utilities.

### Progress log

| Date | Covered | Follow-up |
|---|---|---|
| 2026-10-16 | IP, loopback, ports, CIDR, VPC, subnets, request flow, security boundaries, Mac lab | Practice commands and basic terminology |
| 2026-10-16 | Agreed to avoid manual binary/subnet arithmetic; use `ipcalc`, Python `ipaddress`, and visual calculators | Apply this principle throughout the course |

### Maintenance convention

When we learn something new in the ChatGPT thread:

1. Add a concise new section or lesson here.
2. Add a practical flow diagram in plain text or Mermaid when useful.
3. Explain each *new* command, flag, class, annotation, or setting on first use.
4. Include one small hands-on lab and a few architecture-level takeaways.
5. Update the progress log and move the planned next lesson forward.
6. Correct earlier notes in place if a later lesson clarifies or supersedes them.

Suggested repository location:

```text
architecture-learning/
└── networking/
    └── networking-for-developers-architects.md
```

---

## 12. Reference links

- [AWS: What is Amazon VPC?](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [AWS: VPC subnet sizing and address reservations](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html)
- [AWS: Route tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [AWS: Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Kubernetes: Services, load balancing, and networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Python: `ipaddress` module](https://docs.python.org/3/library/ipaddress.html)
- [RFC 1918: Private IPv4 address space](https://www.rfc-editor.org/rfc/rfc1918)


## Maintaining This Networking Handbook

This is a **living document** that will evolve as I learn networking, AWS, Kubernetes, microservices, and application security.

### How We'll Maintain It

Whenever we cover a new networking topic in our ChatGPT learning thread, I can simply say:

> **"Update our networking Markdown notes."**

ChatGPT will help consolidate the new material into this handbook while:

1. **Preserving existing lessons** and previously covered concepts.
2. **Adding new topics** in a logical learning sequence.
3. **Updating practical examples**, CLI commands, architecture diagrams, and explanations.
4. **Maintaining a learning progress log** to track completed and upcoming topics.
5. **Avoiding unnecessary duplication** and keeping explanations concise and developer-focused.

### Git Repository Workflow

```text
Learn and experiment in ChatGPT
              |
              v
Update the networking Markdown handbook
              |
              v
Review and update the local Markdown file
              |
              v
Git Add → Commit → Push
              |
              v
Read from GitHub anytime
```

### Long-Term Goal

Build a continuously evolving **Networking Handbook for Java Developers, AWS and Kubernetes Architects** that serves as:

- A practical networking learning reference.
- A quick refresher for development and architecture decisions.
- A troubleshooting guide for real-world networking problems.
- A Git-hosted knowledge base accessible even when ChatGPT is unavailable.

### Guiding Principle

**Understand the concepts → Visualize the architecture → Practice with tools → Apply to real-world engineering decisions.**

Prioritize practical understanding over memorizing theory or performing unnecessary manual IP/subnet calculations.
