# Network Stack Design (Simplified TCP/IP)

Simplified network implementation for basic connectivity.

## Network Layer Stack

```
Application Layer
  ├── HTTP Client
  ├── DNS Resolver
  └── Custom protocols
       ↓
Transport Layer (TCP/UDP)
  ├── TCP State Machine
  ├── UDP Datagrams
  └── Port Management
       ↓
Internet Layer (IP)
  ├── IP Header Processing
  ├── Routing Table
  └── Fragmentation/Reassembly
       ↓
Link Layer (Ethernet)
  ├── MAC Frame Processing
  ├── ARP (Address Resolution)
  └── Hardware Addressing
       ↓
Physical Layer
  └── NIC Driver (e.g., RTL8139)
```

## Ethernet Frame Format

```c
struct EthernetFrame {
    uint8_t dest_mac[6];           // Destination MAC
    uint8_t src_mac[6];            // Source MAC
    uint16_t ethertype;            // 0x0800 = IPv4, 0x0806 = ARP
    uint8_t payload[1500];         // Up to 1500 bytes
    uint32_t fcs;                  // Frame check sequence (CRC32)
} __attribute__((packed));

#define ETHERTYPE_IPV4 0x0800
#define ETHERTYPE_ARP  0x0806
```

## IPv4 Header

```c
struct IPv4Header {
    uint8_t version_ihl;           // Version (4) + IHL (4)
    uint8_t dscp_ecn;              // DSCP (6) + ECN (2)
    uint16_t total_length;         // Header + payload
    uint16_t identification;       // Packet ID
    uint16_t flags_offset;         // Flags (3) + Fragment Offset (13)
    uint8_t ttl;                   // Time To Live
    uint8_t protocol;              // 6 = TCP, 17 = UDP
    uint16_t header_checksum;      // Checksum
    uint32_t src_ip;               // Source IP address
    uint32_t dest_ip;              // Destination IP address
} __attribute__((packed));
```

## TCP Header

```c
struct TCPHeader {
    uint16_t src_port;             // Source port
    uint16_t dest_port;            // Destination port
    uint32_t sequence_number;      // Byte sequence number
    uint32_t ack_number;           // Acknowledgment number
    uint8_t offset_flags;          // Offset (4) + Flags (4 reserved, 8 flags)
    uint8_t window_size_hi;        // Window size (bytes 1-2)
    uint8_t window_size_lo;        // Window size (bytes 3-4)
    uint16_t checksum;             // Checksum
    uint16_t urgent_pointer;       // Urgent data pointer
} __attribute__((packed));

#define TCP_FLAG_FIN 0x01
#define TCP_FLAG_SYN 0x02
#define TCP_FLAG_RST 0x04
#define TCP_FLAG_PSH 0x08
#define TCP_FLAG_ACK 0x10
#define TCP_FLAG_URG 0x20
```

## Socket API

```c
#define AF_INET 2
#define SOCK_STREAM 1  // TCP
#define SOCK_DGRAM 2   // UDP

struct sockaddr_in {
    uint16_t sin_family;           // AF_INET
    uint16_t sin_port;             // Port (network byte order)
    uint32_t sin_addr;             // IP address (network byte order)
    uint8_t sin_zero[8];           // Padding
} __attribute__((packed));

// Socket syscalls
int socket(int domain, int type, int protocol);
int bind(int sockfd, struct sockaddr *addr, int addrlen);
int listen(int sockfd, int backlog);
int accept(int sockfd, struct sockaddr *addr, int *addrlen);
int connect(int sockfd, struct sockaddr *addr, int addrlen);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
int close(int sockfd);

// Socket state structure
struct Socket {
    uint32_t sockfd;               // Socket file descriptor
    uint16_t protocol;             // SOCK_STREAM, SOCK_DGRAM
    uint16_t state;                // CLOSED, LISTEN, ESTABLISHED, etc.
    uint32_t local_ip;
    uint16_t local_port;
    uint32_t remote_ip;
    uint16_t remote_port;
    uint32_t send_buffer[2048];
    uint32_t recv_buffer[2048];
} Socket;
```

## TCP State Machine

```
       CLOSED
        ||
        ||  socket()
        ||  bind()
        ||  listen()
        ↓
      LISTEN
       ↙   ↖
      /     \
     / SYN   \ connect()
    ↓        ↓
SYN_RCVD  SYN_SENT
    |         |
    | SYN+ACK | SYN+ACK
    ↓         ↓
ESTABLISHED←──┘
    ||
    || FIN
    ↓
FIN_WAIT_1
    |  (or CLOSE_WAIT from remote FIN)
    | ACK
    ↓
FIN_WAIT_2
    | (remote sends FIN)
    ↓
TIME_WAIT → CLOSED (after 2×MSL)
```

## Socket Implementation

```c
// Create TCP socket
int socket_tcp_create() {
    struct Socket *sock = kmalloc(sizeof(struct Socket));
    sock->sockfd = ++next_sockfd;
    sock->protocol = SOCK_STREAM;
    sock->state = CLOSED;
    
    // Add to socket table
    socket_table[sock->sockfd] = sock;
    return sock->sockfd;
}

// Bind socket to local address
int socket_bind(int sockfd, struct sockaddr_in *addr) {
    struct Socket *sock = socket_table[sockfd];
    sock->local_ip = addr->sin_addr;
    sock->local_port = addr->sin_port;
    sock->state = LISTEN;
    return 0;
}

// Send data on TCP socket
ssize_t socket_send(int sockfd, const void *buf, size_t len) {
    struct Socket *sock = socket_table[sockfd];
    
    if (sock->state != ESTABLISHED) {
        return -1;  // Not connected
    }
    
    // Build TCP packet
    struct TCPHeader *tcp = kmalloc(sizeof(struct TCPHeader) + len);
    tcp->src_port = sock->local_port;
    tcp->dest_port = sock->remote_port;
    tcp->sequence_number = sock->seq_num++;
    tcp->flags_offset = (5 << 4) | TCP_FLAG_PSH | TCP_FLAG_ACK;
    memcpy(tcp + 1, buf, len);
    
    // Calculate checksum and send
    tcp->checksum = calculate_checksum(tcp, sizeof(struct TCPHeader) + len);
    nic_send_packet(tcp, sizeof(struct TCPHeader) + len);
    
    kfree(tcp);
    return len;
}
```

---

**Related Documentation:**
- [ARCHITECTURE.md](../ARCHITECTURE.md)
- [docs/KERNEL_SPEC.md](KERNEL_SPEC.md)