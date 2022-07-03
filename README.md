# DNS-to-DNS-Over-TLS-Proxy
Domain Name System (DNS) is a crucial part of Internet infrastructure. It is responsible for translating a human-readable, memorable domain (like google.com) into a numeric IP address (such as 142.250.192.14).
In order to translate a domain into an IP address, your device sends a DNS request to a special DNS server called a resolver (which is most likely managed by your Internet provider i.e. ISP). The DNS requests are sent in plain text so anyone who has access to your traffic stream can see which domains you visit.

DNS over TLS server is used to send DNS queries over an encrypted connection, by default, DNS queries are sent over plain text connection. 

There are two recent Internet standards that have been designed to solve the DNS privacy issue:
DNS over TLS (DoT)
DNS over HTTPS (DoH)
Both of them provide secure and encrypted connections to a DNS server.

We’ll talk about DNS over TLS (DOT).


Below is the Docker image we used:
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

