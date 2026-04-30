# Networking from First Principles

A primer that builds up from "two computers wanting to talk" to "why is my SSH connection going through Cloudflare". Read this before `remote-access.md` if any of that felt magical.

---

## 1. The starting point: two computers, one wire

Imagine just two computers connected by a single cable. You want to send the message `"hello"` from A to B.

What does the wire actually carry? Voltage. High = 1, low = 0. So `"hello"` becomes bytes (`0x68 0x65 0x6c 0x6c 0x6f`), each byte becomes 8 bits, each bit becomes a voltage pulse. B reads the voltages, reassembles bits → bytes → string. Done.

That's it. That's the whole foundation. Everything else is layers on top of this.

But once you have **more than two computers**, problems pile up:

1. How does A know *which* computer to talk to?
2. What if the wire is noisy and bits get flipped?
3. What if B is busy and can't process the message right now?
4. What if there are 10 computers sharing one wire — how do they take turns?
5. What if A and B are on opposite sides of the world?
6. What if multiple programs on the *same* computer want to talk at the same time?

Each of these is solved by a different **layer**.

---

## 2. The layered model

Networking is built in layers because each problem can be solved independently. The classic mental model is the **OSI 7-layer model**, but in practice everyone uses the simpler **TCP/IP 4-layer model**:

```
┌──────────────────────────────────────┐
│  4. Application                      │  HTTP, SSH, DNS, SMTP — what you write
├──────────────────────────────────────┤
│  3. Transport                        │  TCP, UDP — reliable vs fast delivery
├──────────────────────────────────────┤
│  2. Internet (Network)               │  IP — addressing & routing across networks
├──────────────────────────────────────┤
│  1. Link                             │  Ethernet, Wi-Fi — bits over a wire/airwave
└──────────────────────────────────────┘
```

Each layer **only talks to the layers directly above and below it**. It doesn't know or care what's happening elsewhere.

### Concrete example

When you type `ssh robot@192.168.1.50` in a terminal:

| Layer | What it does |
|---|---|
| **Application (SSH)** | "I want to open a secure shell to that machine. Here's my login + a command." |
| **Transport (TCP)** | "OK, I'll chop your data into packets, number them, retransmit lost ones, ensure they arrive in order." |
| **Internet (IP)** | "I'll figure out the route from your machine to 192.168.1.50, hop by hop." |
| **Link (Ethernet/Wi-Fi)** | "I'll turn each packet into electrical/radio signals to the next physical device." |

On the receiving side, the same stack runs in reverse: signals → packet → assembled data stream → SSH server processes the login.

The key insight: **SSH doesn't know about Wi-Fi, and Wi-Fi doesn't know about SSH.** That's the power of layering.

---

## 3. Layer 1: physical / link

This is the "how do bits actually travel" layer.

- **Ethernet**: copper or fiber cable. Each computer has a **MAC address** (a 48-bit hardware ID like `aa:bb:cc:11:22:33`) burned into its network card.
- **Wi-Fi**: same idea, but radio waves instead of wire.

Within one local network (your home, one university lab), computers find each other by MAC address. Each device shouts (**broadcasts**) "is anyone here `aa:bb:cc:...`?" via a protocol called **ARP** ("Address Resolution Protocol"), and the matching machine replies.

> **Broadcast** = sending a packet to a special "everyone" address (`ff:ff:ff:ff:ff:ff` for Ethernet) that every device on the local segment receives and inspects. It's literally a shout — no targeting, everyone hears it, only the relevant machine answers. The technical term is "broadcast"; "shout" is just the intuition.

This works fine for ~hundreds of devices on one segment. But it doesn't scale to billions globally — you can't shout across the internet, because routers deliberately **don't forward broadcasts** between networks (otherwise one chatty machine would drown the whole internet). So we need a higher layer.

---

## 4. Layer 2: IP addresses — addressing across networks

**IP** (Internet Protocol) gives every machine on the internet a logical address that's independent of physical hardware.

- **IPv4**: 32-bit address, written as 4 dotted numbers, e.g. `192.168.1.50` or `8.8.8.8`. ~4.3 billion possible addresses.
- **IPv6**: 128-bit, e.g. `2606:4700:4700::1111`. Astronomically more, designed to replace IPv4 (we ran out).

### Public vs private IPs

Not all IPs are equal. Some ranges are reserved for **private** use — only valid inside a local network:

| Range | Use |
|---|---|
| `10.0.0.0/8` | Private (big networks, corporate) |
| `172.16.0.0/12` | Private |
| `192.168.0.0/16` | Private (most home routers) |
| `127.0.0.0/8` | Loopback (`127.0.0.1` = "this machine") |
| anything else | Public (routable on the internet) |

When you check your home router, you'll see something like `192.168.1.x`. That IP is **not reachable from the internet** — millions of homes have the same `192.168.1.50`. They don't conflict because they live in separate islands.

This is why your PC has no public IP — your university (and home router) put you on a private network and **NAT** (next section) translates between them and the public internet.

### Routing — how packets find their way

When you send a packet to `8.8.8.8`:

1. Your machine checks: "is `8.8.8.8` on my local network?" No.
2. It sends the packet to its **default gateway** (your router).
3. Your router asks the same question, sends it to *its* gateway (your ISP).
4. The ISP routes it through the global backbone, hop by hop, until it lands at Google's network.

Each hop along the way is a router. The internet is just **a giant mesh of routers cooperatively forwarding packets**.

You can see this with `traceroute 8.8.8.8`.

#### Real example: traceroute from this PC to `8.8.8.8`

```
$ traceroute -w 2 -q 1 -m 20 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 20 hops max, 60 byte packets
 1  172.31.0.3              3.458 ms      ← university gateway (private IP)
 2  172.17.0.3              3.417 ms      ← upstream uni router (private IP)
 3  59.145.92.81            3.407 ms      ← uni edge → ISP handoff (public IP)
 4  125.23.204.133          6.557 ms      ← ISP backbone hop 1
 5  116.119.72.108          6.548 ms      ← ISP backbone hop 2
 6  116.119.61.253         22.867 ms      ← ISP peering point (latency jumps — likely longer physical link)
 7  72.14.243.0            22.842 ms      ← entered Google's network (AS15169)
 8  142.251.193.131        19.585 ms      ← Google internal routing
 9  172.253.67.91          19.616 ms      ← Google internal routing
10  dns.google (8.8.8.8)   19.565 ms      ← destination reached
```

Read this top-to-bottom as the journey of a single packet:

1. **Hops 1–2**: still inside the university network. Private IPs (`172.x`), sub-ms latency.
2. **Hop 3**: the packet leaves the university and enters the ISP's network. First public IP appears.
3. **Hops 4–6**: traversing the ISP backbone. Latency creeps up; hop 6 jumps from 6 ms to 23 ms, suggesting a long-distance link (likely cross-city or to an internet exchange point).
4. **Hop 7**: crossed into Google's network (`AS15169`). You can verify this with `whois 72.14.243.0`.
5. **Hops 8–9**: routing inside Google's infrastructure.
6. **Hop 10**: arrived at `dns.google` — Google's public DNS resolver.

**How traceroute actually works** (a fun trick): it doesn't ask routers nicely for their identity. Each IP packet has a **TTL** (Time To Live) field — the max number of hops before it's dropped. Traceroute sends a packet with `TTL=1`. The first router decrements it to 0, drops the packet, and sends back an ICMP "time exceeded" error — revealing its IP. Then traceroute sends `TTL=2`, hears from the second router, and so on. It's reverse-engineering the path by deliberately sending packets that fail one hop further each time.

### NAT — Network Address Translation

This is the magic that lets your private `192.168.1.50` talk to the public internet.

```
You (192.168.1.50)  ──►  Router (public IP 203.0.113.7)  ──►  google.com
                          rewrites source to 203.0.113.7
                          remembers: "203.0.113.7:54321 ↔ 192.168.1.50:54321"
google.com  ──►  Router  ──►  You
                  rewrites dest back to 192.168.1.50
```

Your router has the only public IP. It rewrites packets so all your devices appear to share that one address, multiplexed by **port number**.

**Why your PC can't be reached from the internet**: NAT only forwards return traffic for connections *you* initiated. Random incoming packets get dropped. There's no rule saying "send unsolicited traffic to 192.168.1.50". This is the exact problem `remote-access.md` is about.

#### Real example: NAT on this PC

Three commands, three different views of the same machine:

```sh
$ ip -4 addr show | grep "inet "
    inet 10.21.0.147/21        ← private IP on wired interface (eno1)
    inet 172.31.12.237/18      ← private IP on Wi-Fi (wlp2s0) — what's actually in use
    inet 192.168.100.1/24      ← Docker's internal bridge (yet another private network!)

$ ip route | grep default
default via 172.31.0.1 dev wlp2s0    ← the Wi-Fi router is our gateway out
default via 10.21.0.1 dev eno1       ← (the wired router as backup)

$ curl -s https://ifconfig.me
59.145.92.86                          ← what the internet sees us as
```

Stop and look at this. **This one PC has three different IPs simultaneously**: `172.31.12.237` (Wi-Fi), `10.21.0.147` (Ethernet), `192.168.100.1` (Docker). All are private. None are routable on the internet. Yet `ifconfig.me` says we're `59.145.92.86`.

That `59.145.92.86` is the *university's edge router*. Every device in the university's Wi-Fi network shares that public IP — hundreds or thousands of laptops appearing as one address to the outside world. Notice it's the same `59.145.x.x` block we saw at hop 3 of the traceroute earlier — same boundary device.

#### NAT in action: live connections

This PC is currently making 6+ outbound connections. Here's a snapshot:

```
$ ss -tn state established
Local Address:Port       Peer Address:Port
172.31.12.237:34890  →   34.149.66.154:443       ← some HTTPS server
172.31.12.237:33050  →   104.26.2.186:443        ← Cloudflare
172.31.12.237:59916  →   34.8.250.101:443        ← Google
172.31.12.237:58570  →   160.79.104.10:443       ← Anthropic (Claude)
172.31.12.237:33936  →   204.80.128.1:443
172.31.12.237:41968  →   58.231.59.90:22         ← an SSH session
```

Look at the **left column**: every outgoing connection uses the *same* local IP (`172.31.12.237`) but a *different* random source port (`34890`, `33050`, `59916`, ...). Those high ports are **ephemeral ports** — picked by the OS for each new connection.

When these packets leave the university gateway, NAT rewrites them:

```
What this PC sends:                What the internet sees:
172.31.12.237:34890 → 34.149.66.154:443    →    59.145.92.86:51000 → 34.149.66.154:443
172.31.12.237:33050 → 104.26.2.186:443     →    59.145.92.86:51001 → 104.26.2.186:443
172.31.12.237:59916 → 34.8.250.101:443     →    59.145.92.86:51002 → 34.8.250.101:443
                                                    ↑                      ↑
                          uni router uses its own public IP +
                          a new ephemeral port per connection
```

The university's NAT table holds entries like:

| Public side | ↔ | Private side |
|---|---|---|
| `59.145.92.86:51000` | ↔ | `172.31.12.237:34890` |
| `59.145.92.86:51001` | ↔ | `172.31.12.237:33050` |
| ... | | ... |

When `34.149.66.154` sends a reply to `59.145.92.86:51000`, the gateway looks up that table, sees it belongs to `172.31.12.237:34890`, rewrites the destination, and forwards the packet back to this PC. **Hundreds of devices behind one public IP, multiplexed entirely by port number.**

Now the punchline: notice there's *no entry* in that NAT table for unsolicited inbound traffic. If someone on the internet tries to send a packet to `59.145.92.86:22` hoping to reach this PC's SSH server, the gateway has no idea where it's supposed to go and drops it. **That's the wall every remote-access tool has to climb over.**

---

## 5. Layer 3: ports — multiple programs on one machine

OK, so an IP address gets you to the right *machine*. But a modern computer is running dozens of network programs at once: web browser, SSH client, Spotify, Slack, etc. How does an incoming packet know which program to go to?

**Ports.** A 16-bit number (0–65535) that identifies a specific *socket* (connection endpoint) on a machine.

The full address of a network connection isn't just an IP — it's `IP:port`. For example:

- `192.168.1.50:22` → SSH server on that machine
- `192.168.1.50:80` → web server on that machine
- `8.8.8.8:53` → Google's DNS resolver

A connection is actually identified by **four things together**: `(source IP, source port, dest IP, dest port)`. That tuple uniquely identifies one conversation. Your laptop can have 50 simultaneous tabs open to `google.com:443` because each tab uses a different *source* port.

### Are port numbers conventions or rules?

**They're conventions, enforced socially, not technically.** A port number is just a number — nothing about TCP says "port 80 must be HTTP". You can run an SSH server on port 80, or a web server on port 22. It'll work, it'll just confuse every client that expects defaults.

To prevent chaos, **IANA** (Internet Assigned Numbers Authority) maintains an official registry of port assignments. Three categories:

| Range | Name | Notes |
|---|---|---|
| **0 – 1023** | Well-known / system ports | Assigned by IANA. On Unix, requires root to bind. |
| **1024 – 49151** | Registered ports | Assigned by IANA on request (e.g., 5432 = PostgreSQL). |
| **49152 – 65535** | Dynamic / ephemeral | Free-for-all, used by clients for outgoing connections. |

### Common ports you should know

| Port | Protocol | What |
|---|---|---|
| 20, 21 | FTP | File transfer (legacy) |
| **22** | SSH | Secure shell |
| 23 | Telnet | Insecure remote shell (don't use) |
| 25 | SMTP | Email sending |
| 53 | DNS | Domain name lookups |
| **80** | HTTP | Web (unencrypted) |
| 110 | POP3 | Email retrieval (legacy) |
| 143 | IMAP | Email retrieval |
| **443** | HTTPS | Web (encrypted) |
| 465, 587 | SMTPS | Email sending (encrypted) |
| 993 | IMAPS | Email retrieval (encrypted) |
| 3306 | MySQL | Database |
| 3389 | RDP | Windows Remote Desktop |
| 5432 | PostgreSQL | Database |
| 6379 | Redis | Cache / DB |
| 8080 | HTTP-alt | Common dev/proxy port |
| 8443 | HTTPS-alt | Common dev/proxy port |
| 27017 | MongoDB | Database |

### Why port 22 for SSH? Why 80 for HTTP?

Largely **historical accident, then locked in by convention**. SSH was assigned 22 in 1995 by IANA when its author requested an unused number near Telnet (23) and FTP (21). HTTP got 80 because Tim Berners-Lee asked IANA for one in 1990. Nothing inherent about the numbers — but now billions of clients assume them.

### Ephemeral ports — what your client uses

When *you* connect out to a server, your OS assigns you a random high port. So a client connection looks like:

```
your laptop         remote server
192.168.1.20:51847  ──►  142.250.80.46:443    (you visiting google.com)
192.168.1.20:51848  ──►  140.82.114.4:443     (you visiting github.com)
```

The 51847/51848 are picked by your OS each time. The server's port is fixed (443); yours is throwaway. This is why hundreds of simultaneous outbound connections work fine.

#### Real example: ports listening on this PC right now

```sh
$ ss -tlnp
State   Local Address:Port   Process
LISTEN  0.0.0.0:22           sshd                ← SSH server (well-known port)
LISTEN  0.0.0.0:9993         zerotier-one        ← ZeroTier daemon (registered, UDP normally)
LISTEN  0.0.0.0:5900         x11vnc              ← VNC remote desktop
LISTEN  127.0.0.1:631        cupsd               ← printing service (loopback only)
LISTEN  127.0.0.1:11434      ollama              ← local LLM server (loopback only)
LISTEN  127.0.0.1:35992      code (VS Code)      ← editor's internal RPC (loopback)
LISTEN  127.0.0.1:5345       ?                   ← random local service
LISTEN  0.0.0.0:2049         nfs                 ← network file system
LISTEN  0.0.0.0:111          rpcbind             ← portmapper for RPC services
LISTEN  127.0.0.53:53        systemd-resolved    ← local DNS stub resolver
```

Three things to notice:

**1. The "address" half of `IP:port` matters as much as the port.**
- `0.0.0.0:22` → "listen on this port from *any* interface". SSH is reachable from the LAN.
- `127.0.0.1:11434` → "listen *only* on loopback". Ollama is reachable only by programs on this same machine. The internet cannot reach it even if you opened the firewall — the OS won't accept the connection.
- `127.0.0.53:53` → systemd-resolved listens on a special loopback alias for DNS.

This is a real security boundary. Many local services bind to `127.0.0.1` deliberately so you can't accidentally expose your database / dev tools.

**2. Mix of well-known, registered, and "what is that" ports.**
- `22` (SSH) and `53` (DNS) — well-known, IANA-assigned.
- `631` (CUPS), `2049` (NFS), `5900` (VNC), `9993` (ZeroTier), `11434` (Ollama) — registered ports for specific software.
- `35992`, `5345`, `38971` — random high ports picked by apps that don't need a fixed port (like VS Code's internal RPC). These are *listening* but on ephemeral-range numbers.

**3. SSH on `0.0.0.0:22` is exactly why this whole remote-access exercise exists.** The server is sitting there, listening, ready to accept connections. The problem is purely network reachability — NAT and the firewall stop external traffic from ever reaching this socket. The application is fine; the path to it isn't.

**Useful related commands:**

```sh
ss -tlnp           # TCP listening sockets, with process names
ss -ulnp           # UDP listening sockets
ss -tn established # TCP connections currently active
lsof -i :22        # which process owns port 22?
sudo lsof -iTCP -sTCP:LISTEN -P -n   # everything listening, with PIDs
```

---

## 6. Layer 3.5: TCP vs UDP

Now you have IPs and ports. But raw IP packets are unreliable — they can arrive out of order, get duplicated, or vanish. You need a transport layer to handle that. Two options:

### TCP — Transmission Control Protocol

**Reliable, ordered, connection-oriented.** Like a phone call.

1. Client and server **handshake** (3-way: SYN → SYN-ACK → ACK) to set up a connection.
2. Data is split into segments, each numbered.
3. Receiver acknowledges each segment. Lost segments get retransmitted.
4. Receiver reassembles in order.
5. Connection is closed cleanly (FIN).

Used by: HTTP/HTTPS, SSH, email, databases — anything where you can't tolerate missing data.

**Cost:** more overhead, more latency. The handshake takes a round trip before any data flows.

### UDP — User Datagram Protocol

**Fire-and-forget.** Like sending a postcard.

1. Just send the packet. No handshake.
2. No retransmission, no ordering, no acknowledgement.
3. If it gets lost, too bad — the application has to handle it (or not care).

Used by: DNS, video calls, gaming, VPNs (WireGuard), QUIC. Anything where speed matters more than perfect delivery, or where the application does its own reliability layer.

### Visual comparison

```
TCP:  CLIENT ──SYN──►       (1)   "want to talk?"
             ◄──SYN-ACK──   (2)   "sure, ready"
             ──ACK──►       (3)   "great, here we go"
             ──data──►      (4+)  ... reliable stream ...

UDP:  CLIENT ──packet──►          (just send it)
             ──packet──►          (and another)
             ◄──maybe a reply──   (if the app feels like it)
```

### Why this matters for firewalls

Firewalls treat TCP and UDP very differently. TCP has explicit connection state — easy to inspect, easy to track. UDP has no state — every packet is independent. Restrictive firewalls often block UDP entirely (except for DNS) because it's harder to monitor and is a common VPN carrier.

**This is why ZeroTier (UDP/9993) was blocked but Cloudflare Tunnel (TCP/443) wasn't.**

---

## 7. Layer 4: application protocols

On top of TCP or UDP, you have application protocols. These define the *shape* of the conversation.

### HTTP — HyperText Transfer Protocol

The web. Runs on TCP, default port 80. Conversation is **request/response** in plain text:

```
GET /index.html HTTP/1.1
Host: example.com
User-Agent: curl/8.0

```

Server replies:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...
```

You can literally `telnet example.com 80` and type this by hand. HTTP is just text over a TCP socket.

#### Real example: a live HTTP exchange to `example.com`

```
$ curl -v http://example.com
*   Trying 104.20.23.154:80...                ← TCP connect to port 80
* Connected to example.com (104.20.23.154) port 80
> GET / HTTP/1.1                              ← request line (method, path, version)
> Host: example.com                           ← which site (one IP can host many)
> User-Agent: curl/7.81.0                     ← who's asking
> Accept: */*                                 ← what response types we'll take
>                                             ← blank line = end of headers
< HTTP/1.1 200 OK                             ← response line (version, status)
< Date: Wed, 29 Apr 2026 23:31:39 GMT
< Content-Type: text/html                     ← what's in the body
< Transfer-Encoding: chunked
< Server: cloudflare                          ← interesting — example.com is on Cloudflare
< cf-cache-status: HIT                        ← Cloudflare served this from edge cache
< CF-RAY: 9f4219fc3c48b2cb-DEL                ← Cloudflare request ID (DEL = Delhi datacenter)
<                                             ← blank line = end of headers, body follows
<!doctype html><html lang="en"><head><title>Example Domain</title>...
```

Notice three things:

1. **It's all plaintext.** Lines starting with `>` are sent by us, `<` are received. No encryption. Anyone between us and Cloudflare (the university, our ISP, anyone on a coffee-shop Wi-Fi) can read every byte.
2. **The `Host: example.com` header is essential.** The server at `104.20.23.154` likely hosts thousands of sites — this header is how it knows which one to serve. Without it, it'd guess.
3. **Cloudflare served us a cached copy** (`cf-cache-status: HIT`) from their Delhi datacenter (`-DEL`). We never actually talked to example.com's "real" server. This is what a CDN does.

### HTTPS — HTTP Secure

HTTP wrapped in **TLS** (Transport Layer Security, formerly SSL). Default port 443.

The TLS handshake:

1. Client says hello, lists ciphers it supports
2. Server picks a cipher, sends a **certificate** (proves it's really `example.com`, signed by a trusted Certificate Authority like Let's Encrypt)
3. Client verifies the cert against its list of trusted CAs
4. They negotiate a shared symmetric key
5. All further HTTP traffic is encrypted with that key

After the handshake, it's just regular HTTP — but encrypted. This is why HTTPS is "the same as HTTP, just secure."

#### Real example: peeking inside TLS

```
$ openssl s_client -connect example.com:443 -servername example.com
subject=CN = example.com                                         ← who the cert is for
issuer=C = US, O = "CLOUDFLARE, INC.",
       CN = Cloudflare TLS Issuing ECC CA 1                      ← who signed it
Verification: OK                                                 ← our system trusts this CA
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384                   ← protocol & cipher chosen
```

What this tells us:

- **`subject=CN = example.com`** — the certificate is issued for the domain `example.com`. If we were lied to and sent to a fake server, the cert wouldn't match this hostname and our client would refuse to connect.
- **`issuer = Cloudflare TLS Issuing ECC CA 1`** — Cloudflare runs its own Certificate Authority, which itself is trusted by browsers because it chains up to a root CA they trust. This is the **chain of trust**: example.com cert → signed by Cloudflare's intermediate CA → signed by a root CA already in your OS's trust store.
- **`Verification: OK`** — our machine successfully walked that chain.
- **`TLSv1.3`** — modern version of TLS (1.3 is the current standard; 1.2 still common; 1.0/1.1 deprecated).
- **`TLS_AES_256_GCM_SHA384`** — the cipher suite they negotiated: AES-256 for encryption, GCM mode for authenticated encryption, SHA-384 for hashing.

Once this handshake completes, every byte after that is encrypted. If you ran `tcpdump` on this connection, you'd see TCP packets to port 443 — but their contents would be opaque ciphertext. Even your university firewall, sitting in the middle, can't read what you're actually requesting (it can only see the destination IP and the SNI hostname during the handshake).

**Servers can host multiple sites on one IP** (Server Name Indication / SNI). The `-servername example.com` flag is what tells the server which certificate to send back. Modern HTTPS reveals the SNI in cleartext during the handshake — this is exactly how the university's firewall identifies and blocks `controlplane.tailscale.com` even though everything is "encrypted."

### SSH — Secure Shell

Remote login + command execution + file transfer + tunneling, all in one protocol. Runs on TCP, default port 22. Has its own crypto (not TLS).

SSH supports things like:
- **Password auth** — you type a password
- **Public key auth** — your client proves it has a private key matching a public key in `~/.ssh/authorized_keys`
- **Port forwarding** — tunnel arbitrary TCP traffic over the SSH connection

SSH reverse tunnels (mentioned in `remote-access.md`) use that last feature.

### DNS — Domain Name System

How `google.com` becomes `142.250.80.46`. Runs primarily on UDP/53 (small queries) with TCP/53 fallback for big responses.

When you type `google.com`:

1. OS asks its configured **resolver** (often your router or `8.8.8.8` or `1.1.1.1`)
2. Resolver asks the **root nameservers** ("who handles `.com`?")
3. Then the **`.com` nameservers** ("who handles `google.com`?")
4. Then **Google's nameservers** ("what's the IP for `google.com`?")
5. Answer cached locally for next time (TTL controls how long)

You can poke at this manually:

```sh
dig google.com
nslookup google.com
host google.com
```

#### Real example: a live DNS lookup for `example.com`

```
$ dig example.com

;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 52231         ← query metadata
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 13

;; QUESTION SECTION:
;example.com.          IN  A                                     ← we asked: A record for example.com

;; ANSWER SECTION:
example.com.    127    IN  A    104.20.23.154                    ← here are the answers
example.com.    127    IN  A    172.66.147.243                   ← (TTL 127s left in cache)

;; AUTHORITY SECTION:
example.com.    127    IN  NS   elliott.ns.cloudflare.com.       ← who's authoritative for this domain
example.com.    127    IN  NS   hera.ns.cloudflare.com.

;; ADDITIONAL SECTION:
elliott.ns.cloudflare.com.  IN  A    172.64.35.228                ← bonus: IPs of those nameservers
hera.ns.cloudflare.com.     IN  A    173.245.58.162
```

What's happening:

- We asked: "what's the **A record** (IPv4 address) for `example.com`?"
- We got back **two IPs**: `104.20.23.154` and `172.66.147.243`. Many sites return multiple IPs for redundancy and load balancing — your browser will pick one (typically the first, with retry on the second).
- The **TTL of 127 seconds** means: we can cache this answer for 127 more seconds before having to ask again.
- The **AUTHORITY section** reveals that example.com's nameservers are at Cloudflare (`*.ns.cloudflare.com`) — Cloudflare manages this domain.
- The **ADDITIONAL section** gives us the IPs of those nameservers as a freebie, so we don't need a separate lookup if we want to talk to them directly.

DNS record types worth knowing:

| Type | What | Example |
|---|---|---|
| **A** | IPv4 address | `example.com → 104.20.23.154` |
| **AAAA** | IPv6 address | `example.com → 2606:2800:21f:cb07:...` |
| **CNAME** | Alias to another name | `www.example.com → example.com` |
| **MX** | Mail server | `example.com → mail.example.com` |
| **NS** | Authoritative nameserver | `example.com → ns1.cloudflare.com` |
| **TXT** | Arbitrary text (SPF, DKIM, domain verification) | `example.com → "v=spf1 ..."` |
| **SOA** | Zone metadata (admin email, refresh interval) | |

Try `dig example.com MX` or `dig google.com TXT` to see other types in action.

### Other application protocols worth knowing

- **SMTP** (sending email), **IMAP/POP3** (receiving email)
- **FTP / SFTP** (file transfer)
- **WebSocket** (bidirectional streaming over HTTP)
- **gRPC** (binary RPC over HTTP/2)
- **MQTT** (lightweight pub/sub for IoT)

---

## 8. Putting it together: what happens when you visit `https://example.com`

```
1. DNS lookup
   Your laptop: "what's example.com?"
   1.1.1.1: "93.184.216.34"

2. TCP connection
   Laptop → 93.184.216.34:443
   3-way handshake (SYN, SYN-ACK, ACK)

3. TLS handshake
   Server presents cert; laptop verifies via trusted CA
   They derive a shared symmetric key

4. HTTP request (encrypted)
   GET / HTTP/1.1
   Host: example.com

5. HTTP response (encrypted)
   200 OK
   <html>...

6. TCP connection close (FIN)
```

Six layers of machinery for "open a web page." Each layer is unaware of the others — that's why this works at all.

---

## 9. Tools to explore this yourself

Run these on your own machine. They make all this concrete.

| Command | What it shows |
|---|---|
| `ip addr` (Linux) / `ifconfig` | Your machine's IPs and MAC addresses |
| `ip route` / `route -n` | Your routing table — where packets go |
| `ping <host>` | Are you reachable to that machine? Round-trip time? |
| `traceroute <host>` | Each hop your packets take across the internet |
| `dig <domain>` | DNS lookup with full detail |
| `nslookup <domain>` | Simpler DNS lookup |
| `curl -v <url>` | Make an HTTP request and see all the headers |
| `openssl s_client -connect host:443` | Inspect a TLS handshake by hand |
| `ss -tunlp` (Linux) / `netstat -an` | What ports are open / connections active on this machine |
| `lsof -i` | Which programs own which sockets |
| `nmap <host>` | Scan a host for open ports (use only on machines you own!) |
| `tcpdump -i any port 443` | Capture raw packets crossing your interface |
| `wireshark` | GUI version of tcpdump — see protocol details visually |

### A fun exercise

```sh
# what's actually flowing when you visit a site
sudo tcpdump -i any -n -s0 'host example.com' &
curl -s https://example.com > /dev/null
```

You'll see the DNS query, the TCP handshake, the TLS handshake, then encrypted application data. All the layers, in order, in real time.

---

## 10. Things to explore next

Roughly in order of usefulness/depth:

### Practical
- **TLS / certificates** — how does HTTPS actually trust certificates? What's a CA? What's Let's Encrypt? What's mutual TLS?
- **HTTP/2 and HTTP/3 (QUIC)** — how the web evolved past plain HTTP/1.1, and why HTTP/3 runs on UDP
- **Reverse proxies** — nginx, Caddy, Traefik. The thing in front of your web app.
- **Load balancers** — how a single hostname can route to thousands of backend servers
- **CDNs** — how Cloudflare/Akamai cache content close to users worldwide

### Networking depth
- **CIDR notation** — `192.168.1.0/24` and what subnet masks really mean
- **Routing protocols** — BGP (the protocol that actually runs the internet), OSPF
- **VLANs** — splitting one physical network into many logical ones
- **NAT traversal techniques** — STUN, TURN, ICE, hole-punching (how WebRTC, Tailscale, ZeroTier really work)
- **VPN protocols** — IPsec, WireGuard, OpenVPN, and how they differ

### Security
- **TLS deep dive** — handshake variants, cipher suites, perfect forward secrecy
- **Certificate transparency** — how the world catches misissued certs
- **Public key cryptography basics** — RSA, ECDSA, Ed25519
- **Firewalls** — iptables, nftables, how stateful inspection works
- **DPI (Deep Packet Inspection)** — what your university actually does to your traffic

### Hands-on rabbit holes
- Run your own DNS server (e.g., dnsmasq, Pi-hole)
- Set up a personal VPN with WireGuard, expose your home network
- Build a tiny TCP server in your favorite language (it's ~10 lines)
- Read [High Performance Browser Networking](https://hpbn.co/) — free online, the best deep dive
- Try [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) — classic intro to socket programming

---

## 11. The mental ladder

If you remember nothing else from this doc:

1. Computers talk by sending **bits over wires/radio**.
2. **MAC addresses** identify machines on a local network.
3. **IP addresses** identify machines anywhere on the internet.
4. **Ports** identify *programs* on a machine. `IP:port` is the full address.
5. **TCP vs UDP** is reliable-stream vs fire-and-forget.
6. **Application protocols** (HTTP, SSH, DNS, etc.) define the conversation shape on top of TCP/UDP.
7. **NAT** is why your PC has no public IP — and why `remote-access.md` exists.
8. **Firewalls** filter traffic by IP, port, protocol, or content. They're why we have to be clever about which port/protocol to use to escape.

Everything else — TLS, HTTP/3, BGP, VPNs, Cloudflare — is layered on top of these eight ideas.
