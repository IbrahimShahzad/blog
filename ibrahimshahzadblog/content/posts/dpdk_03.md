---
title: DPDK 03
date: 2021-07-02
image: ./dpdk.png
banner: ./fast.jpg
description: Extracting packet headers
---

# Introduction

Hello There. Today we will parse a few layers. 

> This is the continuation of the DPDK series. You will need to have gone through [<font color="blue">DPDK-02</font>](https://ibrahimshahzad.github.io/blog/dpdk_02/). 

**Let us begin!**

# PreReqs

- [<font color="blue">DPDK-02</font>](https://ibrahimshahzad.github.io/blog/dpdk_02/)
- Live dpdk binded port with traffic  

# The Layers



- To understand packet filtering, you first have to understand packets and how they are handled at each layer of the `TCP/IP` protocol stack:
  1. Application layer (e.g., FTP, Telnet, HTTP)
  2. Transport layer (TCP or UDP)
  3. Internet layer (IP)
  4. Network access layer (e.g., Ethernet, FDDI, ATM)
![Layers](https://i.imgur.com/adlnXNm.gif)
- We are going to parse Ethernet, IP and Transport Layer.
- We will only get a few values from each and log them.
- We will extract following params:
- Source/Destination Mac Addresses
  - From Ethernet Layer
- Source/Destination IP
  - From IP Layer
- Source/Destination Port
  - From Transport Layer
- let's step in our working folder
```bash
cd $RTE_SDK/examples/my_app
```

## Here is what we are aiming for

<center>

![design](https://i.imgur.com/SoVK42r.png)

</center>

## The Network Access Layer

- The packet here has got two parts: 
  - Ethernet header
    - Packet kind
    - Ethernet source address
    - Ethernet destination address 
  - Ethernet body
    - rest of packet data

### The code

- Lets  write some code to parse the layer now.
- we are going to use the `rte_ether_hdr` struct from `rte_ether.h`
  - it has following members
    - `struct rte_ether_addr d_addr`
        - this contains destination mac address array
    - `struct rte_ether_addr s_addr`
        - this contains source mac address array
    - `ether_type`
      - Frame type
  - see more [<font color="blue">here</font>](https://doc.dpdk.org/api/structrte__ether__hdr.html)
- in the `main.c` file go to the `worker_main` function and add a loop for going over all the packets.
```c
int worker_main(void *arg){
  const u_int8_t nb_ports = rte_eth_dev_count_avail();
  u_int8_t port;
  u_int8_t dest_port;

  /* Run until app is killed or quit */
  for(;;){
    /* Receive packets on port */
    for(port=0;port< nb_ports;port++){
      struct rte_mbuf *bufs[BURST_SIZE];
      u_int16_t nb_rx;
      u_int16_t buf;

      /* Get burst fo RX packets */
      nb_rx = rte_eth_rx_burst(port,0,
          bufs,BURST_SIZE);
      if (unlikely(nb_rx==0))
        continue;

      for(int i =0;i<nb_rx;i++){
        /* Write Code here */
      }

      /* send burst of Tx packets to the 
       * second port
       */
      dest_port = port ^ 1;
      nb_tx= rte_eth_tx_burst(dest_port, 0,
          bufs, nb_rx);
      
      /* Free any unsent packets. */
      for (buf=0;buf<nb_rx;buf++)
        rte_pktmbuf_free(bufs[buf]);

    }
  }
  return 0;
}
```
- lets first define the ether_header
```c
struct rte_ether_hdr *ethernet_header;
```
- Now we will arrays to store source/destination MAC addresses
- The length of the array is defined by `RTE_ETHER_ADDR_LEN` which is `6`
```c
u_int8_t source_mac_address[RTE_ETHER_ADDR_LEN];
u_int8_t destination_mac_address[RTE_ETHER_ADDR_LEN];
```
- now we use `rte_pktmbuf_mtod()` function
  - The way to remember this function is `packet-mbuf-to-data` function.
  - we populate our `ethernet_header` struct using this.
```c
ethernet_header = rte_pktmbuf_mtod(bufs[i], struct rte_ether_hdr *); 
```
- Now we simply populate our value holders.
```c
u_int16_t ethernet_type;
ethernet_type = ethernet_header->ether_type;
rte_memcpy(source_mac_address,&ethernet_header->s_addr,sizeof(u_int8_t)*RTE_ETHER_ADDR_LEN);
rte_memcpy(destination_mac_address,&ethernet_header->d_addr,sizeof(u_int8_t)*RTE_ETHER_ADDR_LEN);
```
- Finally we will log all the values extracted.
- All together it looks like below:
```c
/* Get burst fo RX packets */
nb_rx = rte_eth_rx_burst(port,0,
    bufs,BURST_SIZE);
if (unlikely(nb_rx==0))
  continue;

for(int i=0;i<nb_rx;i++){
  /* Write Code here */
  struct rte_ether_hdr *ethernet_header;
  u_int8_t source_mac_address[RTE_ETHER_ADDR_LEN];
  u_int8_t destination_mac_address[RTE_ETHER_ADDR_LEN];
  ethernet_header = rte_pktmbuf_mtod(bufs[i], struct rte_ether_hdr *); 
  ethernet_type = ethernet_header->ether_type;
  rte_memcpy(source_mac_address,&ethernet_header->s_addr,sizeof(u_int8_t)*RTE_ETHER_ADDR_LEN);
  rte_memcpy(destination_mac_address,&ethernet_header->d_addr,sizeof(u_int8_t)*RTE_ETHER_ADDR_LEN);
  RTE_LOG(INFO,APP,"Source Mac: ");
  for(int i=0;i<RTE_ETHER_ADDR_LEN;i++)
    printf("%x",source_mac_address[i]);
  printf("\n");
  RTE_LOG(INFO,APP,"Destination Mac: ");
  for(int i=0;i<RTE_ETHER_ADDR_LEN;i++)
    printf("%x",source_mac_address[i]);
  printf("\n");
  RTE_LOG(INFO,APP,"ether type: %d",ethernet_type);
}
/* 
* send burst of Tx packets to the 
* second port
*/
dest_port = port ^ 1;
nb_tx= rte_eth_tx_burst(dest_port, 0,
    bufs, nb_rx);
```

## The Internet Layer

- This layer is responsible for routing messages between different local networks.
- IP addresses in IPv4 follow a format of `xxx.xxx.xxx.xxx`, where each decimal value (0–255) translates into 8 binary bits called an octet.
- We are going to use `struct rte_ipv4_hdr` from `rte_ip.h`
  - contains IP-related defines
  - contains `struct rte_ipv4_hdr` and `struct rte_ipv6_hdr`
- `rte_ipv4_hdr` has following members
  - `version_ihl`
    - version and header length
  - `type_of_service`
    - type of service
  - `total_length`
    - Length of packet
  - `packet_id`
    - packet ID
  - `fragment_offset`
    - fragmentation offset
  - `time_to_live`
    - time to live
  - `next_proto_id`
    - protocol ID
  - `hdr_checksum`
    - header checksum
  - `src_addr`
    - source ip address
  - `dst_addr`
    - destination ip address

### The Code

- The last two bytes in the ethernet layer tell us about the next layer.
- Let's write a function that will take the 16 bit value and tell us whether the layer is ipv4 or not.
  - the value is `2048 (dec)` in case of ipv4
- write down the following functions
```c
#define IPV4_PROTO 2048 
#define IPV6_PROTO 34525
u_int16_t get_Ether_Type(char *pointer)
{
    u_int8_t slb= 0;
    u_int8_t lb= 0;
    slb= *(pointer - 2 );// second last byte of ETH layer
    lb= *(pointer - 1 );// last last byte of ETH layer
    return (slb* 256) +lb;
}

bool is_ipv4(u_int16_t val)
{
  if (val == IPV4_PROTO)
    return true;
  return false;
}
```
- now let's create a pointer that we will use to traverse the packet bytes
```c
void* pHdrTraverse = (void*) ((unsigned char*) ether_header + sizeof (struct rte_ether_hdr));// Pointer to Next Layer to Eth
```

- now lets call `get_Ether_type()` and assign the value to `next_proto`
```c
u_int16_t next_proto = get_Ether_Type(pHdrTraverse);// holds last two byte value of ETH Layer
```
- now let's check what the next layer is
  - incase the `next_proto` is not `IPv4` we will log it
```c
if (is_ipv4(next_proto)){

}else
{
  RTE_LOG(INFO,APP,"NOT IPv4\n");
}
```
- now within the if clause we will parse the `ipv4 header`.
```c
struct rte_ipv4_hdr ipv4_header;
u_int32_t u32SrcIPv4;
u_int32_t u32DstIPv4;
ipv4_header= rte_pktmbuf_mtod_offset(bufs[i], struct rte_ipv4_hdr*, sizeof (struct rte_ether_hdr));
```
- okay so now the struct may be filled however, we need to check whether it is a valid ipv4 or not.
  - for that we create `is_valid_ipv4_pkt()` function
```c
static inline int is_valid_ipv4_pkt(struct rte_ipv4_hdr *pkt, uint32_t link_len)
{
    /* From http://www.rfc-editor.org/rfc/rfc1812.txt section 5.2.2 */
    /*
     * 1. The packet length reported by the Link Layer must be large
     * enough to hold the minimum length legal IP datagram (20 bytes).
     */
    if (link_len < sizeof(struct rte_ipv4_hdr))
        return -1;
    /* 2. The IP checksum must be correct. */
    /* this is checked in H/W */
    /*
     * 3. The IP version number must be 4. If the version number is not 4
     * then the packet may be another version of IP, such as IPng or
     * ST-II.
     */
    if (((pkt->version_ihl) >> 4) != 4)
        return -3;
    /*
     * 4. The IP header length field must be large enough to hold the
     * minimum length legal IP datagram (20 bytes = 5 words).
     */
    if ((pkt->version_ihl & 0xf) < 5)
        return -4;
    /*
     * 5. The IP total length field must be large enough to hold the IP
     * datagram header, whose length is specified in the IP header length
     * field.
     */
    if (rte_cpu_to_be_16(pkt->total_length) < sizeof(struct rte_ipv4_hdr))
        return -5;
    return 0;
}
```
- now lets call it in our if clause.
- we are going to hold ipv4 src and dst addresses in unsigned int 32 variables.
```c
if (is_valid_ipv4_pkt(pIP4Hdr,bufs[i]->pkt_len)>=0){
    /* update TTL and CHKSM */
    --(pIP4Hdr->time_to_live);
    ++(pIP4Hdr->hdr_checksum);

    u32SrcIPv4 = rte_bswap32(pIP4Hdr->src_addr);
    u32DstIPv4 = rte_bswap32(pIP4Hdr->dst_addr);
} else {
  RTE_LOG(INFO,APP,"invalid IPv4\n");
}
```
- The `rte_bswap32` is used to swap `bytes` in a `32-bit value`.

## The Transport Layer

- The protocols of this layer provide host-to-host communication services for applications.
  - Different applications use either `TCP` or `UDP` to establish a connection.
  - The ports used by the application are contained within the header as the `source port` and `destination port`

### The Code

- The next layer can be checked by `next_proto_id` within the `rte_ipv4_hdr` struct.
  - We are going to extract the source and destination ports incase of
    - TCP notified by `IPPROTO_TCP` i.e. `6`
    - UDP notified by `IPPROTO_UDP` i.e. `17`
```c
switch(pIP4Hdr->next_proto_id){
  case IPPROTO_TCP:    
    break;

  case IPPROTO_UDP: 
      break;

  default:
      u16DstPort = 0;
      u16SrcPort = 0;
      break;

}
```
- Default will handle the case in which `next_proto_id` is neither `TCP` or `UDP`.
  - We will set the default value `0` for both ports indicating this case
- Now let's parse the TCP header
  - We will create a `rte_tcp_hdr` struct contained in `rte_tcp.h`
  - Read more about `rte_tcp.h` [<font color="blue">here</font>](http://doc.dpdk.org/api/rte__tcp_8h.html)
  - The `rte_tcp_hdr` contains
    - src port
    - dst port
    - sent_seq
    - recv_ack
    - data_off
    - tcp_flags
    - rx_win
    - cksum
    - tcp_urp
  - Read more about `rte_tcp_hdr` [<font color="blue">here</font>](http://doc.dpdk.org/api/structrte__tcp__hdr.html)
- Include the header 
```c
include <rte_tcp.h>
```
- Create the `rte_tcp_hdr struct`
```c
struct rte_tcp_hdr pTcpHdr;
```
- Lastly, handle the TCP case
```c
case IPPROTO_TCP:
    pTcpHdr = (struct rte_tcp_hdr *) ((unsigned char *) pIP4Hdr + sizeof(struct rte_ipv4_hdr));
    u16DstPort = rte_bswap16(pTcpHdr->dst_port);
    u16SrcPort = rte_bswap16(pTcpHdr->src_port);
    break;

```
- Create the `rte_tcp_hdr struct`
```c
struct rte_udo_hdr pUdpHdr;
```
- Lastly, handle the UDP case
```c
#include <rte_udp.h>
```
  - We will create a `rte_udp_hdr` struct contained in `rte_udp.h`
```c
struct rte_udp_hdr pUdpHdr;
```
  - The `rte_udp_hdr` contains
    - src port
    - dst port
    - dgram_len
    - dgram_cksum
    - Read more about `rte_upd_hdr` [<font color="blue">here</font>](https://doc.dpdk.org/api/structrte__udp__hdr.html)
  - Read more about `rte_udp.h` [<font color="blue">here</font>](https://doc.dpdk.org/api/rte__udp_8h.html)
```c
case IPPROTO_UDP:
    pUdpHdr = (struct rte_udp_hdr *) ((unsigned char *) pIP4Hdr + sizeof (struct rte_ipv4_hdr));
    u16DstPort = rte_bswap16(pUdpHdr->dst_port);
    u16SrcPort = rte_bswap16(pUdpHdr->src_port); 
    break;
```

# All Together


```c
for(int i=0;i<nb_rx;i++){
    /* Write Code here */
    struct rte_ether_hdr *ethernet_header;
    u_int8_t source_mac_address[RTE_ETHER_ADDR_LEN];
    u_int8_t destination_mac_address[RTE_ETHER_ADDR_LEN];
    struct rte_ipv4_hdr     *pIP4Hdr;
    struct rte_ipv6_hdr     *pIP6Hdr;
    struct rte_udp_hdr      *pUdpHdr;
    struct rte_tcp_hdr      *pTcpHdr;
    u_int32_t u32SrcIPv4= 0;
    u_int32_t u32DstIPv4= 0;
    u_int16_t u16SrcPort= 0;
    u_int16_t u16DstPort= 0;
    u_int16_t ethernet_type;
    ethernet_header = rte_pktmbuf_mtod(bufs[i], struct rte_ether_hdr *);
    ethernet_type = ethernet_header->ether_type;
    rte_memcpy(source_mac_address,&ethernet_header->s_addr,sizeof(u_int8_t)*RTE_ETHER_ADDR_LEN);
    rte_memcpy(destination_mac_address,&ethernet_header->d_addr,sizeof(u_int8_t)*RTE_ETHER_ADDR_LEN);
    RTE_LOG(INFO,APP,"Source Mac: ");
    for(int i=0;i<RTE_ETHER_ADDR_LEN;i++)
        printf("%x ",source_mac_address[i]);
    printf("\n");
    RTE_LOG(INFO,APP,"Destination Mac: ");
    for(int i=0;i<RTE_ETHER_ADDR_LEN;i++)
        printf("%x ",source_mac_address[i]);
    printf("\n");
    RTE_LOG(INFO,APP,"ether type: %d",ethernet_type);
    void* pHdrTraverse = (void*) ((unsigned char*) ethernet_header + sizeof (struct rte_ether_hdr));// Pointer to Next Layer to Eth
    u_int16_t next_proto = get_Ether_Type(pHdrTraverse);// holds last two byte value of ETH Layer
    RTE_LOG(INFO,APP,"next proto %u\n",next_proto);
    if (is_ipv4(next_proto)){
        pIP4Hdr = rte_pktmbuf_mtod_offset(bufs[i], struct rte_ipv4_hdr*, sizeof (struct rte_ether_hdr));
        /* check for valid packet */
        if (is_valid_ipv4_pkt(pIP4Hdr,bufs[i]->pkt_len)>=0){
            /* update TTL and CHKSM */
            --(pIP4Hdr->time_to_live);
            ++(pIP4Hdr->hdr_checksum);
            u32SrcIPv4 = rte_bswap32(pIP4Hdr->src_addr);
            u32DstIPv4 = rte_bswap32(pIP4Hdr->dst_addr);
            RTE_LOG(INFO,APP,"IPv4 src %u dst %u\n",u32SrcIPv4,u32DstIPv4);

            switch(pIP4Hdr->next_proto_id){
            case IPPROTO_TCP:
                pTcpHdr = (struct rte_tcp_hdr *) ((unsigned char *) pIP4Hdr + sizeof(struct rte_ipv4_hdr));
                u16DstPort = rte_bswap16(pTcpHdr->dst_port);
                u16SrcPort = rte_bswap16(pTcpHdr->src_port);
                break;

            case IPPROTO_UDP:
                pUdpHdr = (struct rte_udp_hdr *) ((unsigned char *) pIP4Hdr + sizeof (struct rte_ipv4_hdr));
                u16DstPort = rte_bswap16(pUdpHdr->dst_port);
                u16SrcPort = rte_bswap16(pUdpHdr->src_port);
                break;

            default:
               u16DstPort = 0;
               u16SrcPort = 0;
               break;

            }
            RTE_LOG(INFO,APP,"TL src %u dst %u\n",u16SrcPort,u16DstPort);
        }

    }
}
```

---

# Now Lets Run this

- First we build our program
- This should have created a `build` directory within our directory
```bash
  make
```
- now run the program using the following command
```bash
  ./build/my_app -l 0-3 -n 3 -- -p 0x3
```
-  you should see some logs as following
```bash
  APP: Source Mac: 0 ff 56 88 84 ff
  APP: Destination Mac: 0 50 56 ff 84 ff
  APP: ether type: 8APP: next proto 2048
  APP: IPv4 src 3232200221 dst 1700832002
  APP: TL src 14550 dst 443
```
# WORD

<center>

![Meow](https://i.imgflip.com/2dm9ef.jpg)

</center>

- Have a cookiee. Treat-yo-self! We are finally done with this series
- You can look at the code [<font color="blue">here</font>](https://github.com/IbrahimShahzad/dpdk-learning)
- As always, checkout the [<font color="blue">DPDK documentation</font>](https://www.dpdk.org/) as well.
   