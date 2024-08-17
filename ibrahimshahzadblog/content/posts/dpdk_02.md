---
title: DPDK 02
date: 2021-06-23
image: ./dpdk.png
banner: ./fast.jpg
description: Writing the application
---

# Introduction

In this section we will write a smiple C application to recieve packets. 

Let's dive in!


# PreReqs

Make sure that

- DPDK is built

- A DPDK compatible NIC is binded to the `igb_uio` driver

- Hugepages are setup

- Minimum system requirements are met

Please see [<font color="blue">DPDK-01</font>](https://ibrahimshahzad.github.io/blog/Writing_your_first_dpdk_app/) if any of the aforementioned pre-reqs are not set.

# File Setup

> From here on out we will refer to our dpdk installation directory as `RTE_SDK`. 

- Go into the your dpdk installation directory and run the following command

```bash
export RTE_SDK=$(pwd)
```
- The directory should contain following folders

```bash
app/
build/
buildtools/
.ci/
config/
devtools/
doc/
drivers/
.editorconfig
examples/
.gitattributes
.gitignore
kernel/
lib/
license/
MAINTAINERS
Makefile
meson.build
meson_options.txt
README
.travis.yml
usertools/
VERSION
```

- For our purposes, we are concerned with the `examples` directory and the `usertools` directory.

- The `examples` directory contains the example codes and we will also create our application here

- The `usertools` directory contains user-help scripts such as 
  -  `cpu_layout.py`; prints out the Architecture,        
  -  `dpdk-devbind.py`; for binding/unbinding PCI devices,  
  -  `dpdk-hugepages.py`; for setting up hugepages

- some of these we used at the time of setup.

- Now, go to the examples folder and create a folder for our app

```bash
cd $RTE_SDK/examples
mkdir my_app
cd my_app
```

## MAKE File

- lets create a simple Makefile

```bash
vim Makefile
```

- we are going to name our app `my_app` and use `main.c` as the source c file 

```makefile
# binary name
APP = my_app

# all source are stored in SRCS-y
SRCS-y := main.c

# Build using pkg-config variables if possible
ifneq ($(shell pkg-config --exists libdpdk && echo 0),0)
$(error "no installation of DPDK found")
endif

all: shared
.PHONY: shared static
shared: build/$(APP)-shared
        ln -sf $(APP)-shared build/$(APP)
static: build/$(APP)-static
        ln -sf $(APP)-static build/$(APP)

PKGCONF ?= pkg-config

PC_FILE := $(shell $(PKGCONF) --path libdpdk 2>/dev/null)
CFLAGS += -O3 $(shell $(PKGCONF) --cflags libdpdk)
LDFLAGS_SHARED = $(shell $(PKGCONF) --libs libdpdk)
LDFLAGS_STATIC = $(shell $(PKGCONF) --static --libs libdpdk)

CFLAGS += -DALLOW_EXPERIMENTAL_API

build/$(APP)-shared: $(SRCS-y) Makefile $(PC_FILE) | build
        $(CC) $(CFLAGS) $(SRCS-y) -o $@ $(LDFLAGS) $(LDFLAGS_SHARED)

build/$(APP)-static: $(SRCS-y) Makefile $(PC_FILE) | build
        $(CC) $(CFLAGS) $(SRCS-y) -o $@ $(LDFLAGS) $(LDFLAGS_STATIC)

build:
        @mkdir -p $@

.PHONY: clean
clean:
        rm -f build/$(APP) build/$(APP)-static build/$(APP)-shared
        test -d build && rmdir -p build || true

```

## Main.c

- Now lets move on to the main program

```c
#include <stdio.h>
#include <stdlib.h>

#include <rte_eal.h>
#include <rte_common.h>


int main(int argc, char* argv[]){
  printf("\n");

  

  return 0;
}
```




### EAL: "Environment Abstraction Layer"


-  The first thing that is required is initialising up the `EAL`. We use `rte_eal_init()` function.
   -  It gets parameters from cli adn sets up a some of the following:
      -  cpu_init: fill cpu_info structure
      -  log_init
      -  config_init: create memory configuration in shared memory
      -  pci_init: scan pci bus
      -  memory_init (hugepages)
      -  memzone_init: initialize memzone subsystem
      -  lcore_init: Create a thread per lcore
      -  pci_probe: probel all physical devices
-  lets start by including `rte_eal.h`
   -  contains the EAL configuration functions
   -  Contains the defitnion for `rte_eal_init()`
      -  The `rte_eal_init` returns `-ive` in case of an error.
   -  See more [<font color="blue">here</font>](https://doc.dpdk.org/api/rte__eal_8h.html#a5c3f4dddc25e38c5a186ecd8a69260e3).
-  we will also include `rte_common.h` 
   -  It contains generic, commonly-used macro and inline function definitions for DPDK.
   -  It contains the defitnion for `rte_exit()`
      -  we would need to exit in case of an error.
   -  See more [<font color="blue">here</font>](https://doc.dpdk.org/api/rte__common_8h.html)

```c
#include <stdio.h>
#include <stdlib.h>

#include <rte_eal.h>
#include <rte_common.h>


int main(int argc, char* argv[]){
        
  printf("\n");
  int ret;

  /* The EAL arguments are passed when calling the program */   
  ret = rte_eal_init(argc,argv);
  if (ret<0)
    rte_exit(EXIT_FAILURE,"EAL Init failed\n");

  argc -= ret;
  argv += ret;
  return 0;
}

```


### Get Port Count

-  We are going to be receiving on one port and transmitting on the other.
For that we need even number of ports.
-  lets first include `rte_ethdev.h` in our program.
   -  includes functions to setup/configure an Ethernet device.
   -  Ethernet devices are represented by a generic data structure of type `rte_eth_dev`.
   -  contains the definition of `rte_eth_dev_count_avail()`
       -  returns the total number of dpdk binded devices.
   -  see more [<font color="blue">here</font>](https://doc.dpdk.org/api-17.08/rte__ethdev_8h.html)
-  lets do that nex within our main function

```c
  nb_ports = rte_eth_dev_count_avail();
  if(nb_ports < 2 || (nb_ports & 1))
    rte_exit(EXIT_FAILURE,"Invalid port number\n");
```
### Logs

-  We are going to include `rte_log` for logging.
   -  contains log API to RTE applications.
   -  contains the `RTE_LOG()` function we are going to use for logging.
-  define the User type logs as follows:   
```c
#define RTE_LOGTYPE_APP RTE_LOGTYPE_USER1  
```

-  now update the code as follows
```c
/*...*/
nb_ports = rte_eth_dev_count_avail();
if(nb_ports < 2 || (nb_ports & 1))
  rte_exit(EXIT_FAILURE,"Invalid port number\n");
RTE_LOG(INFO, APP, "Number of ports:%u\n",nb_ports);
/*...*/
```
### MBUF Pool Creation

We need to reserve some memory for holding the packets in our application. In this section we are going to do just that 

-  We are going to include `rte_mbuf.h` for memory management.
   -  The mbuf library provides the ability to create and destroy buffers that may be used by the RTE application to store message buffers.
   -  we are going to use `rte_pktmbuf_pool_create` to create set up a buffer to hold packets. 
   -  it uses `mempool` library.
   -  Read DPDK Docs regarding `mbuf` [<font color="blue">here</font>](https://doc.dpdk.org/guides/prog_guide/mbuf_lib.html)
-  [<font color="blue">A good read about general concepts here</font>](https://www.dpdk.org/blog/2019/08/21/memory-in-dpdk-part-1-general-concepts/)
-  First We are going to define `NUM_MBUFS` and `MBUF_CACHE_SIZE`.

```c
#include <rte_mbuf.h>
#define NB_MBUFS 8191
#define MBUF_CACHE_SIZE 250
#define RX_RING_SIZE 128
#define TX_RING_SIZE 512
```
-  Now we are going to call `rte_pktmbuf_pool_create` in the `main` function.
   -  This function creates and initializes a packet mbuf pool.
   -  This function uses `rte_memzone_reserve()` to allocate memory.
      -  It reserves a portion of physical memory from hugepages
-  Required params are as follows
```bash
|---------------|-------------------------------------------------------|
| param         |                     Description                       |
|---------------|-------------------------------------------------------|
| name          | The name of the mbuf pool. we are setting it as       |
|               | "MBUF_POOL"                                           |
| n             | The number of elements in the mbuf pool. We are       |
|               | setting it as NB_MBUF * number of ports. The optimum  |
|               | size (in terms of memory usage) for a mempool is when |           
|               |  n is a power of two minus one: n = (2^q - 1).        |
| cache_size    | Size of the per-core object cache.                    | 
|               | Set to MBUF_CACHE_SIZE                                |
| priv_size     | Size of application private are between the rte_mbuf  |
|               | structure and the data buffer. Set to 0.              |
| data_room_size| Size of data buffer in each mbuf,                     |
|               | including RTE_PKTMBUF_HEADROOM.                       |
| socket_id     | The socket identifier where the memory should be      |
|               | allocated. The value can be SOCKET_ID_ANY             |
|               | if there is no NUMA constraint for the reserved zone. |
|---------------|-------------------------------------------------------|
```  
-   If the creation is unsuccessful we are going to exit the program.
```c
/* Create a new mbuf mempool */
mbuf_pool = rte_pktmbuf_pool_create("MBUF_POOL",
    NB_MBUFS *nb_ports,
    MBUF_CACHE_SIZE, 0, RTE_MBUF_DEFAULT_BUF_SIZE,
    rte_socket_id());

if (mbuf_pool == NULL)
  rte_exit(EXIT_FAILURE,"mbuff_pool create failed\n");
```

### Ports Initialisation

-   We will create a function ` port_init` for initialising the ports.
```c
port_init(u_int8_t port,struct rte_mempool *mbuf_pool){

  return 0;
}
```
-  We will loop through all the ports and initialise them one by one.
```c
int main(int argc, char* argv[]){
  /*...*/
  /* Initialize all ports */
  for (portid = 0; portid < nb_ports; portid++){
    if(port_init(portid,mbuf_pool) != 0)
      rte_exit(EXIT_FAILURE,"port init failed\n");
  }
  /*...*/
}
```
-  Now lets update the `port_init` function

-  First we set the `rte_eth_conf`
   -  A structure used to configure an Ethernet port
   -  We can set up Receive mode and Transmit mode flags
   -  If you want to enable `RSS` (Receive Side Scaling) this would be the place to start. 
```c
        struct rte_eth_conf port_conf = {
          .rxmode = { .max_rx_pkt_len = RTE_ETHER_MAX_LEN 
          }
        };
```
-  Now we define the rx/tx queues we are going to use per port i.e. 1 per port, lcore

```c
  const u_int16_t nb_rx_queues = 1;
  const u_int16_t nb_tx_queues = 1; 
```
-  Next we use the main function `rte_eth_dev_configure()` function to set up the ports.
   -  configures the Ethernet device. 
   -  this function must be invoked first before any other function in the Ethernet API.

```c
  /* configure the ethernet device */
  ret = rte_eth_dev_configure(port,
      nb_rx_queues,
      nb_tx_queues,
      &port_conf);
```
-  Now we allocate one `RX queue` per port.
-  We will use function `rte_eth_rx_queue_setup()`
   -  The function allocates a contiguous block of memory for receive descriptors
   -  Read more [<font color="blue">here</font>](https://doc.dpdk.org/api/rte__ethdev_8h.html#a36ba70a5a6fce2c2c1f774828ba78f8d)
-  We will set up 1 rx queue per port

```c
  for (q=0;q<nb_rx_queues;q++){
    ret=rte_eth_rx_queue_setup(port,q,RX_RING_SIZE,
        rte_eth_dev_socket_id(port),
        NULL, mbuf_pool);
    if (ret<0)
      return ret;
  }
```
-  Now we allocate one `TX queue` per port.
-  We will use function `rte_eth_tx_queue_setup()`
   -  Allocate and set up a transmit queue for an Ethernet device.
   -  Read more [<font color="blue">here</font>](https://doc.dpdk.org/api/rte__ethdev_8h.html#a796c2f20778984c6f41b271e36bae50e)
-  We will set up 1 tx queue per port

```c
  for (q=0;q<nb_tx_queues;q++){
    ret=rte_eth_tx_queue_setup(port,q,TX_RING_SIZE,
      rte_eth_dev_socket_id(port),
      NULL);
    if (ret<0)
      return ret;
  }
```
-  All togethor now:
```c
int port_init(u_int8_t port,struct rte_mempool *mbuf_pool){

  struct rte_eth_conf port_conf = {
        .rxmode = { .max_rx_pkt_len = ETHER_MAX_LEN }
        };
  
  const u_int16_t nb_rx_queues = 1;
  const u_int16_t nb_tx_queues = 1;
  int ret;

  /* configure the ethernet device */
  ret = rte_eth_dev_configure(port,
                nb_rx_queues,
                nb_tx_queues,
                &port_conf);
  if (ret != 0)
    return ret;
  
  /* Allocate and setup 1 RX queue per Ethernet port */
  for (q=0;q<nb_rx_queues;q++){
    ret=rte_eth_queue_setup(port,q,RX_RING_SIZE,
                        rte_eth_dev_socket_id(port),
                        NULL, mbuf_pool);
    if (ret<0)
     return ret;
  }

  /* Allocate and setup 1 RX queue per Ethernet port */
  for (q=0;q<nb_tx_queues;q++){
    ret=rte_eth_tx_queue_setup(port,q,TX_RING_SIZE,
      rte_eth_dev_socket_id(port),
      NULL);
  
    if (ret<0)
      return ret;
  }
  
  return 0;

}
```
-  Finally, we start the device using `rte_eth_dev_start()`
-  we will also enable promisuous mode `rte_eth_promiscuous_enable()`
   -  in `promiscuous` mode every data packet transmitted can be received and read by a network adapter
-  Update the main function as follows
```c
/* start the ethernet port */
ret = rte_eth_dev_start(port);

if (ret<0){
  return ret;
}

/* Enable RX in promiscuous mode for the Ethernet device */
rte_eth_promiscuous_enable(port);

```
### Check Link Status

-  We are going to create a new function to check the link status of the ports
-  It is going to tell us which of the links are up and which are down.

```c
static int check_link_status(u_int16_t nb_ports)
{
  struct rte_eth_link link;
  u_int8_t port;
  for (port=0;port<nb_ports;port++){
    rte_eth_link_get(port,&link);
    if(link.link_status == ETH_LINK_DOWN){
      RTE_LOG(INFO,APP,"Port: %u Link DOWN\n",port);
      return -1;
    }
    RTE_LOG(INFO,APP,"Port: %u Link UP Speed %u\n",
        port,link.link_speed);
  }
}

```
-  Now we can call our function in the `main()`

```c
ret = check_link_status(nb_ports);
  if ( ret < 0 ){
    RTE_LOG(WARNING,APP,"Some ports are down\n");
  }
```

### Workers Init

-  We will create a worker function that will run all the cores
-  This worker will be responsible for
   -  receiving packets
   -  parsing packets
   -  transmitting packets

```c
int worker_main(void *arg){
  /* Run until app is killed or quit */
  for(;;){

  }
  return 0;
}
```
-  In the main program we will launch the `woker_main()` function on all the lcores available
-  We will use `rte_eal_mp_remote_launch()` for this
   -  Launches a function on all lcores
   -  first argument is the function name (`worker_main`) that needs to be launched on all cores
   -  The next argrument takes in the arguments that need to be passed. Since we have nothing to pass we will set this as `NULL`.
   -  The third arguemnt specifies whether the function needs to run on all cores 
      -  including `MASTER`/`MAIN` core (`CALL_MAIN` flag)
      -  excluding `MASTER`/`MAIN` core (`SKIP_MAIN` flag)
-  Then we use `rte_eal_mp_wait_lcore` so that `MASTER` core waits for other worker cores.
   -  This keeps the program from exiting.
   -  We can print stats on master core if needed.
```c
  ret=check_link_status(nb_ports);
  if (ret<0){
    RTE_LOG(WARNING,APP,"Some ports are down\n");
  }
  rte_eal_mp_remote_launch(worker_main,NULL,SKIP_MAIN);
  rte_eal_mp_wait_lcore();
```

### Receive/Transmit Packets
-  Lets define the `BURST_SIZE` first.
   =  This defines how many packets will the core pick up at a time.
```c
  #define BURST_SIZE 32
```
-  In the worker_main we will set up struct rte_mbuf array that will be of the size `BURST_SIZE`
   -  This will be defined by `struct rte_mbuf *bufs[BURST_SIZE];`
-  We will recieve packets using `rte_eth_rx_burst` function
   -  It requires 
      -  `port` number on which to receive
      -  `queue` number on which to receive
      -  `rte_mbuf` structure to hold the packets
      -  Number of packets to receive.
   - It will return the total number of packets received (`nb_rx`)
   - We will transmit the recieved packets on the opposite port.
     -  Receive on `1`, send on `0`
     -  Receive on `0`, send on `1`
   - For Transmitting we use the function `rte_eth_tx_burst`
      -  `port` number on which to transmit
      -  `queue` number on which to transmit
      -  `rte_mbuf` structure that holds the packets
      -  Number of packets to transmit
   - After that we will loop through packets again and free all the unsent packets
     -  packets are freed using `rte_pktmbuf_free()` function
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
### Some Stats
-  Lets print some stats when we exit the program.
-  We'll need to add the `signal handler` to first catch the `interrupt`.
   -  First lets create a `flag` that we will change when the signal is caught

```c

  static volatile bool force_quit;

```
   -  Now the `handler` which updates the flag and displays the the stats

```c
static void signal_handler(int signum){
  if(signum== SIGINT || signum== SIGTERM){
    printf("\n\nSignal %d received,preparing to exit...\n",signum);
    force_quit = true;
    print_stats();
  }
}
```
-  lets register
   - add following in the `main` function
```c
   force_quit = false;
   signal(SIGINT,signal_handler);
   signal(SIGTERM,signal_handler);
```
-  update the `for loop` condition in `worker_main`
   -  use `while(!force_quit)` instead of `for(;;)` 
-  Lastly, lets create a function to display the stats
   -  It uses the `rte_eth_stats` struct and `rte_eth_stats_get` function 
   -  the function takes in `port_id` for which the stats are required and `rte_eth_stats` struct
   -  It fills the `rte_eth_stats` struct which contains
      - `ipackets`: packets received by the interface
      - `opackets`: packets sent by the interface
      - `imissed`: packets dropped by the interface
```c
static void
print_stats(void){
  struct rte_eth_stats stats;
  u_int8_t nb_ports = rte_eth_dev_count_avail();
  u_int8_t port;

  for(port=0;port<nb_ports;port++){
    printf("\nStatistics for the port %u\n",port);
    rte_eth_stats_get(port,&stats);
    printf("RX:%911u Tx:%911u dropped:%911u\n",
        stats.ipackets,stats.opackets,stats.imissed);
  }
}
```





---
## Now Lets Run this

-  First we build our program
-  this should have created a `build` directory within our directory
```bash
  make
```
-  We will pass our program arguments just like `L2FWD`.
   -  `-l` will indicate cores
   -  `-p` will indicate portmask
      - `0x1` indicates 1 port i.e binary mask `0001`
      - `0x3` indicates 2 port i.e binary mask `0011`
      - `0x7` indicates 3 port i.e binary mask `0111`
```bash
  ./build/my_app -l 0-3 -n 3 -- -p 0x3
```
-  you should see something like
```bash
EAL: Detected 18 lcore(s)
EAL: Detected 1 NUMA nodes
EAL: Detected shared linkage of DPDK
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'PA'
EAL: No available hugepages reported in hugepages-2048kB
EAL: Probing VFIO support...
EAL: VFIO support initialized
EAL:   Invalid NUMA socket, default to 0
EAL:   Invalid NUMA socket, default to 0
EAL:   Invalid NUMA socket, default to 0
EAL:   Invalid NUMA socket, default to 0
EAL: No legacy callbacks, legacy socket not created
APP: Number of ports:2
APP: MAC address swapping enabledi
APP: Port: 0 Link UP Speed 10000
APP: Port: 1 Link UP Speed 10000
APP: Some ports are down
APP: lcore 2 exiting
APP: lcore 3 exiting
```
-  Press `Ctrl-C` to exit the program and print some stats
```bash
Signal 2 received,preparing to exit...

Statistics for the port 0
RX: 0           Tx: 13664640    dropped: 0

Statistics for the port 1
RX: 13664640    Tx: 0           dropped: 0
```
---

# WORD

![Meow](https://i.kym-cdn.com/entries/icons/original/000/035/692/cover1.jpg)

-  Give yourself a pat on the back you are done. (FOR NOW!)
-  We will parse a few layers in the next one.
-  I followed [ferruhy/dpdk-simple-app](https://github.com/ferruhy/dpdk-simple-app) and it was of a great help to me. 
   -  Please do check his repo out.
   -  Infact this whole thing was inspired by this simple learning repo of his.
-  You can look at the code [<font color="blue">here</font>](https://github.com/IbrahimShahzad/dpdk-learning)
-  Checkout the [<font color="blue">DPDK documentation</font>](https://www.dpdk.org/) as well.