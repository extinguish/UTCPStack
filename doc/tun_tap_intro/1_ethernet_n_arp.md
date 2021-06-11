## Ethernet & ARP

### 1. TUN/TAP devices

To intercept low-level network traffic from the Linux kernel, we will use a Linux TAP device. 
In short, a TUN/TAP device is often used by networking userspace applications to manipulate L3/L2 traffic, respectively. 
A popular example is tunneling, where a packet is wrapped inside the payload of another packet.

The advantage of TUN/TAP devices is that they’re easy to set up in a userspace program and they are already being used in a multitude of programs, such as `OpenVPN`.

As we want to build the networking stack from the layer 2 up, we need a TAP device. We instantiate it like so:

```cpp
/*
 * Taken from Kernel Documentation/networking/tuntap.txt
 */
int tun_alloc(char *dev)
{
    struct ifreq ifr;
    int fd, err;

    if( (fd = open("/dev/net/tap", O_RDWR)) < 0 ) {
        print_error("Cannot open TUN/TAP dev");
        exit(1);
    }

    CLEAR(ifr);

    /* Flags: IFF_TUN   - TUN device (no Ethernet headers)
     *        IFF_TAP   - TAP device
     *
     *        IFF_NO_PI - Do not provide packet information
     */
    ifr.ifr_flags = IFF_TAP | IFF_NO_PI;
    if( *dev ) {
        strncpy(ifr.ifr_name, dev, IFNAMSIZ);
    }

    if( (err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0 ){
        print_error("ERR: Could not ioctl tun: %s\n", strerror(errno));
        close(fd);
        return err;
    }

    strcpy(dev, ifr.ifr_name);
    return fd;
}
```

After this, the returned file descriptor fd can be used to read and write data to the virtual device’s ethernet buffer.

The flag `IFF_NO_PI` is crucial here, otherwise we end up with unnecessary packet information prepended to the Ethernet frame. 
You can actually take a look at the kernel’s [source code](https://github.com/torvalds/linux/blob/v4.4/drivers/net/tun.c#L1306) of the tun-device driver and verify this yourself.


### 2. Ethernet Frame Format

The multitude of different Ethernet networking technologies are the backbone of connecting computers in `Local Area Networks (LANs)`. 
As with all physical technology, the Ethernet standard has greatly evolved from its first version2, published by Digital Equipment Corporation, Intel and Xerox in 1980.

The first version of Ethernet was slow in today’s standards - about 10Mb/s and it utilized `half-duplex` communication, meaning that you either sent or received data, but not at the same time. This is why a `Media Access Control (MAC)` protocol had to be incorporated to organize the data flow. Even to this day, Carrier Sense, Multiple Access with Collision Detection (CSMA/CD) is required as the MAC method if running an Ethernet interface in `half-duplex` mode.

The invention of the 100BASE-T Ethernet standard used twisted-pair wiring to enable full-duplex communication and higher throughput speeds. Additionally, the simultaneous increase in popularity of Ethernet switches made CSMA/CD largely obsolete.

The different Ethernet standards are maintained by the `IEEE 802.33` working group.

Next, we’ll take a look at the Ethernet Frame header. It can be declared as a C struct followingly:

```cpp
#include <linux/if_ether.h>

struct eth_hdr
{
    unsigned char dmac[6];
    unsigned char smac[6];
    uint16_t ethertype;
    unsigned char payload[];
} __attribute__((packed));
```







