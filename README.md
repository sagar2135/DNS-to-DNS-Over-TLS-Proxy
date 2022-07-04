# DNS-to-DNS-Over-TLS-Proxy
Domain Name System (DNS) is a crucial part of Internet infrastructure. It is responsible for translating a human-readable, memorable domain (like google.com) into a numeric IP address (such as 142.250.192.14).
In order to translate a domain into an IP address, your device sends a DNS request to a special DNS server called a resolver (which is most likely managed by your Internet provider i.e. ISP). The DNS requests are sent in plain text so anyone who has access to your traffic stream can see which domains you visit.


**DNS Spoofing Architecture Diagram**

<img width="437" alt="image" src="https://user-images.githubusercontent.com/28102334/177039575-020e6893-97d8-41fe-b1c1-fcb738a9bc6d.png">



DNS over TLS server is used to send DNS queries over an encrypted connection, by default, DNS queries are sent over plain text connection. 

There are two recent Internet standards that have been designed to solve the DNS privacy issue:
DNS over TLS (DoT)
DNS over HTTPS (DoH)
Both of them provide secure and encrypted connections to a DNS server.

We’ll talk about DNS over TLS (DOT).

**DNS Architecture Diagram:**

![DNS_Architecture](https://user-images.githubusercontent.com/28102334/177097334-870d2bc8-39c1-425d-964d-9129e89fb7f3.png)

**What's going on in the architecture:**

1. This is the Amazon-provided default DNS server for the central DNS VPC, which we’ll refer to as the DNS-VPC. This is the second IP address in the VPC CIDR range (as illustrated, this is 172.27.0.2). This default DNS server will be the primary domain resolver for all workloads running in participating AWS accounts.
2. This shows the Route 53 Resolver endpoints. The inbound endpoint will receive queries forwarded from on-premises DNS servers and from workloads running in participating AWS accounts. The outbound endpoint will be used to forward domain queries from AWS to on-premises DNS.
3. This shows conditional forwarding rules. For this architecture, we need two rules, one to forward domain queries for onprem.private zone to the on-premises DNS server through the outbound gateway, and a second rule to forward domain queries for awscloud.private to the resolver inbound endpoint in DNS-VPC.
4. This indicates that these two forwarding rules are shared with all other AWS accounts through AWS Resource Access Manager and are associated with all VPCs in these accounts.
5. This shows the private hosted zone created in each account with a unique subdomain of awscloud.private. We have created our application server in of the account and used docker to take care of DNS over TLS instead of plaintext.
6. This shows the on-premises DNS server with conditional forwarders configured to forward queries to the awscloud.private zone to the IP addresses of the Resolver inbound endpoint.

**Below is the Docker image we used:**
___________________________________________________________________________________________________________________________________________________________________

version: "3"

**# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/**

services:
  
  pihole:

        container_name: pihole
        image: pihole/pihole:latest
        # For DHCP it is recommended to remove these ports and instead add: network_mode: "host"
        ports:
          - "53:53/tcp"
          - "53:53/udp"
          - "80:80/tcp"
        environment:
          TZ: 'America/Chicago'
          # WEBPASSWORD: 'set a secure password here or it will be random'
        # Volumes store your data between container upgrades
        volumes:
          - './etc-pihole:/etc/pihole'
          - './etc-dnsmasq.d:/etc/dnsmasq.d'    
        #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
        cap_add:
          - NET_ADMIN # Recommended but not required (DHCP needs NET_ADMIN)      
        restart: unless-stopped
____________________________________________________________________________________________________________________________________________________________________


**Dnsdist Configuration:**

Create dnsdist configuration file /etc/dnsdist/dnsdist.conf with the following content:
___________________________________________________________________________________________________________________________________________________________________
addACL('0.0.0.0/0')

-- path for certs and listen address for DoT ipv4,
-- by default listens on port 853.
-- Set X(int) for tcp fast open queue size.
addTLSLocal("0.0.0.0", "/etc/letsencrypt/live/dns.example.com/fullchain.pem", "/etc/letsencrypt/live/dns.example.com/privkey.pem", { doTCP=true, reusePort=true, tcpFastOpenSize=64 })


-- set X(int) number of queries to be allowed per second from a IP
addAction(MaxQPSIPRule(50), DropAction())

--  drop ANY queries sent over udp
addAction(AndRule({QTypeRule(DNSQType.ANY), TCPRule(false)}), DropAction())

-- set X number of entries to be in dnsdist cache by default
-- memory will be preallocated based on the X number
pc = newPacketCache(10000, {maxTTL=86400})
getPool(""):setCache(pc)

-- server policy to choose the downstream servers for recursion
setServerPolicy(leastOutstanding)

-- Here we define our backend, the pihole dns server
newServer({address="127.0.1.53:5300", name="127.0.1.53:5300"})

setMaxTCPConnectionsPerClient(1000)    -- set X(int) for number of tcp connections from a single client. Useful for rate limiting the concurrent connections.
setMaxTCPQueriesPerConnection(100)    -- set X(int) , similiar to addAction(MaxQPSIPRule(X), DropAction())
______________________________________________________________________________________________________________________________________________________________________

**Check if DoT works using kdig program from the knot-dnsutils package:**

apt install knot-dnsutils
kdig -d @dns.example.com +tls-ca leaseweb.com
