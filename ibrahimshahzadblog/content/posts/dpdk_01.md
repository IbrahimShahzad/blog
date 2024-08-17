---
title: DPDK 01
date: 2021-06-13
image: ./dpdk.png
banner: ./fast.jpg
description: Setting up DPDK
---

# Introduction

## What is DPDK?


Data Plane Development Kit (DPDK) is a set of libraries that enable fast packet processing.

## Why would we want to use it?

If you are writing any application that deals with packets AND require fast
performance then you might need to look into fast packet processing libraries
such as DPDK.

In a normal flow the kernel receives a packet and does does its processing after
which it forwards the packet to the application in the userspace. Doing this few
each and every packet is quite costly in terms of performance.

With DPDK you bind the interface with the drivers provided, after which the
packet received can directly be accessed by the application  in userspace. 

In short DPDK allows you to bypass the kernel and process the received packets
directly in user space.

_DATAPATH_ ususally entails 10s to 100s of Gigs per second on an interface. If you
want to write code dealing with such a traffic DPDK would be a fair choice.


## What Do you need?

In order to follow along with the material you will need the following:


- A linux based system:
    
    - 2 Intel processors, each with 8 cores, 8 GB RAM 

    - [<font color="blue">DPDK compatible NIC</font>](http://core.dpdk.org/supported/)

- Some knowledge of C

Furthermore, for later you would also need to read up on following concepts:

- Enviroment Abstraction Layer

- Hugepages

- Non-Uniform Memory Architecture (NUMA)

- Transition Lookaside Buffer (TLB)

- CPU isolation and pinning


Some Useful Links

- [<font color="blue">Programmers Guide</font>](http://doc.dpdk.org/guides/prog_guide/intro.html)

- [<font color="blue">API Documentation</font>](http://doc.dpdk.org/api/)

- [<font color="blue">DPDK Home</font>](https://www.dpdk.org/)

---

# Lets Get Started

## DPDK installation

We will be using DPDK stable version 20.11.1. We would need to do A couple
things in this section

1. Install the dependencies and Build DPDK

2. Set up some `Hugepages`

3. Bind the NIC to the `igb_uio` dirver

Fire up that ubuntu terminal and let us begin...


### Installing dependencies/DPDK

First lets install some dependencies

```bash
sudo apt-get install -y build-essential \
                        git \
                        libnuma-dev \
                        software-properties-common \
                        python3 \
                        python3-pip \
                        python3-pyelftools
```

Now that the dependencies have been taken care of lets get the DPDK tar file
and build it.

We will write a script that would help installing DPDK later as well. (i have
a habit of writing scripts XD)


open up a new file with your favourite editor (i hope its vim)

```bash
vim install-dpdk.sh
```

Now paste the following in the file

```bash
#!/bin/bash
pushd /opt/
wget https://fast.dpdk.org/rel/dpdk-20.11.1.tar.xz
tar xf dpdk-20.11.1.tar.xz
pushd dpdk-stable-20.11.1
pip3 install meson ninja
meson build
pushd build
ninja
ninja install
ldconfig
popd
popd
popd

```

The above code should do the following:

1. get the `dpdk-20.11.1.tar.xz` file in `/opt/` directory

2. untar it

3. install `meson ninja`

4. build the code

5. update the library path


Make the file executable and run the script by following:

```bash
chmod u+x install-dpdk.sh
./install-dpdk.sh
```

### Setting up Hugepages

There are two sizes of hugepages that we can set; `2M` or `1G`. 

We can set it 
the following way either by 

- using the script in dpdk/usertools directory 

**OR** 
    
- by setting them up in Grub file

#### Usertools

- goto `/opt/dpdk-stable-20.11.1/usertools` where you will find 
`dpdk-hugepages.py` python script

- You can see the usage by running it with `-h` flag which should show you
following information

```bash

        usage: dpdk-hugepages.py [-h] [--show] [--clear] [--mount] [--unmount] 
        [--node NODE] [--pagesize SIZE] [--reserve SIZE] [--setup SIZE]

        Setup huge pages

        optional arguments:
        -h, --help            show this help message and exit
        --show, -s            print the current huge page configuration
        --clear, -c           clear existing huge pages
        --mount, -m           mount the huge page filesystem
        --unmount, -u         unmount the system huge page directory
        --node NODE, -n NODE  select numa node to reserve pages on
        --pagesize SIZE, -p SIZE
                                choose huge page size to use
        --reserve SIZE, -r SIZE
                                reserve huge pages. Size is in bytes with K, M, or G suffix
        --setup SIZE          setup huge pages by doing clear, unmount, reserve and mount

        Examples:

        To display current huge page settings:
            dpdk-hugepages.py -s

        To a complete setup of with 2 Gigabyte of 1G huge pages:
            dpdk-hugepages.py -p 1G --setup 2G
```

- you can set up 4 hugepages with size of 1G each by running the following 
command

```bash
python3 dpdk-hugepages.py -p 1G --setup 4G
```

#### Grub file

- We can set hugepages in the grub file as well. This would allow the machine to
set the pages up during boot time.
Run the following command
```bash
sed -i -e 's/GRUB_CMDLINE_LINUX="/GRUB_CMDLINE_LINUX="default_hugepagesz=1G hugepagesz=1G hugepages=4 iommu=pt intel_iommu=on isolcpu=0,1,2,3,4,5 /'  /etc/default/grub
```

The above command inserts a line in `/etc/deault/grub` file that:

1. sets 4 hugepages with size `1G` each
2. isolates certain cpus from kernel scheduling so that we can dedicate those for our application.

- Update the grub

```bash
sudo update-grub
```

- Restart your machine

```bash
sudo reboot
```

- you can check whether the `hugepages` are set up or not by running the
following:

```bash
cat /proc/meminfo | grep -i "huge"

        AnonHugePages:         0 kB
        ShmemHugePages:        0 kB
        FileHugePages:         0 kB
        HugePages_Total:      20
        HugePages_Free:       20
        HugePages_Rsvd:        0
        HugePages_Surp:        0
        Hugepagesize:    1048576 kB
        Hugetlb:        20971520 kB

```


- Pay attention to `HugePages_Total`, `HugePages_Free` and `Hugepagesize` here.

### installing the drivers

- Get the drivers

```bash
sudo apt install dpdk-igb-uio-dkms -y
```

- load the driver

```bash
sudo modprobe uio
```

### bind the NIC to the driver


- goto `/opt/dpdk-stable-20.11.1/usertools` where you will find 
`dpdk-devbind.py` python script

- run with `-s` flag that will show the staus of all the NICs

```bash
python3 dpdk-devbind.py -s

        Network devices using kernel driver
        ===================================
        0000:0b:00.0 'example' if=ens1 drv=i40e unused=vfio-pci,igb_uio *Active*
        0000:01:00.0 'example' if=ens2 drv=i40e unused=vfio-pci,igb_uio
        0000:01:00.1 'example' if=ens3 drv=i40e unused=vfio-pci,igb_uio
        0000:01:00.2 'example' if=ens4 drv=i40e unused=vfio-pci,igb_uio
        0000:01:00.3 'example' if=ens5 drv=i40e unused=vfio-pci,igb_uio

```

- we can bind the desired NIC to the `igb_uio` driver using its PCI address as
follows

```bash
python3 ./dpdk-devbind.py -b igb_uio 0000:01:00.0
```

This should change the status as follows:

```bash
python3 dpdk-devbind.py -s

        Network devices using DPDK compatible driver
        ===================================
        0000:01:00.0 'example' if=ens2 drv=igb_uio unused=i40e,vfio-pci

        Network devices using kernel driver
        ===================================
        0000:0b:00.0 'example' if=ens1 drv=i40e unused=vfio-pci,igb_uio *Active*
        0000:01:00.1 'example' if=ens3 drv=i40e unused=vfio-pci,igb_uio
        0000:01:00.2 'example' if=ens4 drv=i40e unused=vfio-pci,igb_uio
        0000:01:00.3 'example' if=ens5 drv=i40e unused=vfio-pci,igb_uio

```

---
# ALL Done

![phew](https://i.pinimg.com/originals/59/54/b4/5954b408c66525ad932faa693a647e3f.jpg)

- We have finally set up the **DPDK**.

- In the next one we will _write C code to receive and transmit the packets
using DPDK_.

- In the meanwhile go to `/opt/dpdk-stable-20.11.1/examples` where you will 
find a lot of example code. I would suggest to try the `helloworld` and 
`l2fwd`.

- you cand read about **_L2FWD_** [<font color="blue">here</font>](https://doc.dpdk.org/guides/sample_app_ug/l2_forward_real_virtual.html) 