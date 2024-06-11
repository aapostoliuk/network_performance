# VyOS NIC Tuning Guide

## Linux network packet reception

![linux pack reception](/images/linux-networking-recv.png)

1. Packets arrive at the NIC
1. NIC will verify `MAC` (if not on promiscuous mode) and `FCS` and decide to drop or to continue
1. NIC will [DMA packets at RAM](https://en.wikipedia.org/wiki/Direct_memory_access), in a region previously prepared (mapped) by the driver
1. NIC will enqueue references to the packets at receive [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer) queue `rx` until `rx-usecs` timeout or `rx-frames`
1. NIC will raise a `hard IRQ`
1. CPU will run the `IRQ handler` that runs the driver's code
1. Driver will `schedule a NAPI`, clear the `hard IRQ`, and return
1. Driver raises a `soft IRQ (NET_RX_SOFTIRQ)`
1. NAPI will poll data from the receive ring buffer until `netdev_budget_usecs` timeout or `netdev_budget` and `dev_weight` packets
1. Linux will also allocate memory to `sk_buff`
1. Linux fills the metadata
1. Linux will pass the skb to the kernel stack (`netif_receive_skb`)
1. It will set the network header, clone `skb` to taps (i.e. tcpdump) and pass it to tc ingress
1. Packets are handled to a qdisc sized `netdev_max_backlog` with its algorithm defined by `default_qdisc`
1. It calls `ip_rcv` and packets are handed to IP
1. It calls netfilter (`PREROUTING`)
1. It looks at the routing table, if forwarding or local
1. If it's local it calls netfilter (`LOCAL_IN`)
1. It calls the L4 protocol (for instance `tcp_v4_rcv`)
1. It finds the right socket
1. It goes to the tcp finite state machine
1. Enqueue the packet to the receive buffer and sized as `tcp_rmem` rules
    1. If `tcp_moderate_rcvbuf` is enabled kernel will auto-tune the receive buffer
1. Kernel will signalize that there is data available to apps (epoll or any polling system)
1. Application wakes up and reads the data

## Linux kernel network transmission

![linux pack transmission](/images/linux-networking-recv.png)

1. Application sends message (`sendmsg` or other)
1. TCP send message allocates skb_buff
1. It enqueues skb to the socket write buffer of `tcp_wmem` size
1. Builds the TCP header (src and dst port, checksum)
1. Calls L3 handler (in this case `ipv4` on `tcp_write_xmit` and `tcp_transmit_skb`)
1. L3 (`ip_queue_xmit`) does its work: build ip header and call netfilter (`LOCAL_OUT`)
1. Calls output route action
1. Calls netfilter (`POST_ROUTING`)
1. Fragment the packet (`ip_output`)
1. Calls L2 send function (`dev_queue_xmit`)
1. Feeds the output (QDisc) queue of `txqueuelen` length with its algorithm `default_qdisc`
1. The driver code enqueues the packets at the `ring buffer tx`
1. The driver will do a `soft IRQ (NET_TX_SOFTIRQ)` after `tx-usecs` timeout or `tx-frames`
1. Re-enable hard IRQ to NIC
1. Driver will map all the packets (to be sent) to some DMA'ed region
1. NIC fetches the packets (via DMA) from RAM to transmit
1. After the transmission NIC will raise a `hard IRQ` to signal its completion
1. The driver will handle this IRQ (turn it off)
1. And schedule (`soft IRQ`) the NAPI poll system
1. NAPI will handle the receive packets signaling and free the RAM

## General Performance Tuning Guide

### Hardware selection

1. Platform.

    VyOS supports various devices, but if you are interested in a high performance level, the platform should be selected responsibly. Select motherboards/servers that will fully support all installed components. Check system requirements for your CPU, Memory, NIC for details and compare with motherboard parameters.

2. CPU.

    A powerful CPU is crucial for a good performance. There are few recommendations you can use for it:

    - Avoid using multi-CPU systems.

      The connection speed between CPUs is not so fast as inside the CPU itself, thus it is better to have more CPU cores in a single CPU than two CPUs with fewer cores each.

    - Use a CPU with more powerful cores if your traffic cannot be split into multiple streams.

      If a router forwards traffic that must be processed on a single CPU core (a single TCP/UDP connection, tunnel or VPN, etc.), prefer more GHz than cores number. Each network stream utilizes only a single CPU core, thus if you expect to have high-speed streams, pay attention to single-core performance.

    - Check instructions.

      In case of complex calculations, like encryption or hashing, different CPU types can have different efficiency. The availability of modern instruction sets can improve performance a lot.

    - Check accelerations.

      Some specific tasks can be offloaded from a main CPU work, by utilizing dedicated components. For example, Intel Quick Assist can offload all encryption/decryption of IPSec traffic and unlock much faster speeds.

    - Supported memory.

      Prefer CPUs with faster memory support. For networking tasks total memory size is not very important - even 20-30 GB usually is enough for even very complex configurations. But, number of supported channels and memory speed impacts a performance a lot.

3. Memory.

    - Use memory with a maximum speed supported by your CPU, or faster.

      Pay attention: in many cases, high-speed memory which speed is limited by the CPU will work faster than memory that supports exactly the same speed as the CPU.

    - Avoid using registered memory.

      Registered memory adds some penalty to latency, and is not optimal for routers.

4. NIC.

    - Do not use a router as a switch.

      In the vast majority of cases, you do not need more than 2-4 physical interfaces in a router. Connecting a router to a core switch with bonding created from 2-4 interfaces is a much better idea than termination of different physical links in a router itself.

    - Select well-known and popular vendors.

      VyOS supports hundreds different NIC types, which means all of them can forward your traffic. But, in the wild, only a few vendors are widely available, well-tested, and have enough feedback to consider them stable and predictable. We recommend using NICs from Intel or NVIDIA whenever it is possible.

    - PCI compatibility.

      Pay special attention, if your PCI slot is compatible with a NIC:

        - check the maximum power available in a slot
        - check the physical size of a slot and place for NIC
        - check PCIe slot generation

### CPU Governor

By default Linux uses the 'powersave' CPU governor. On 100G hosts, we've seen throughput increase by up by 30% by changing to the 'performance' governor instead.

```bash
sudo cpupower frequency-set -g performance
```

To watch the CPU governor in action, you can do this:

```bash
watch -n 1 grep MHz /proc/cpuinfo
```

To see the range of possible clock speeds for your CPUs, you can use the *lscpu* command.

You can also set the default CPU governor in the BIOS. Note that if you manually set the CPU power setting in the BIOS, then this change command may not work.

### Firewall and NAT rules tunning

Check you firewall and NAT rules. Optimize them. Use different kind of groups <https://docs.vyos.io/en/sagitta/configuration/firewall/groups.html>
Decreasing the number of firewall rules will increase throughput performance.

### Using Flowtable

Flowtables allow you to accelerate packet forwarding in software (and in hardware if your NIC supports it) by using a conntrack-based network stack bypass.
Try to use flowtables <https://docs.vyos.io/en/sagitta/configuration/firewall/flowtables.html>

## NIC Tunning for Forwarding

### Jumbo Frames

When the expected traffic environment consists of large blocks of data being transferred, it might be beneficial to enable the jumbo frame feature (increase MTU up to 9000).
It is usually used in datacenters or server farms.

```vyos cli
set interfaces ethernet ethX mtu 9000
```

### Ring Buffer - rx,tx

 NIC Ring buffer. It’s a circular buffer, fixed size, FIFO, located at RAM. Buffer to smoothly accept bursts of connections without dropping them, you might need to increase these queues when you see drops or overrun, aka there are more packets coming than the kernel is able to consume them, the side effect might be increased latency.

- **What** - number of descriptors wich the RX/TX ring buffer can queue.
- **Why** - buffer to smoothly accept bursts of connections without dropping them, you might need to increase these queues when you see drops or overrun, aka there are more packets coming than the kernel is able to consume them, the side effect might be increased latency.
- **How:**
  - **Check command:** `show interfaces ethernet ethX physical`
  - **Change command:** `set interfaces ethernet ethX ring-buffer [tx|rx] <count>`
  - **How to monitor:** `show interfaces ethernet ethX statistics | grep -e "err" -e "drop" -e "over" -e "miss" -e "timeout" -e "reset" -e "restar" -e "collis" -e "over" | grep -v "\: 0"`

Recommendations: Increase values step by step while errors, drops, etc. increase. The maximum values are in the 'Pre-set maximums' section.

### Interrupt Coalescence (IC) - rx-usecs, tx-usecs, rx-frames, tx-frames (hardware IRQ)

NIC enqueue references to the packets at receive ring buffer queue rx until rx-usecs timeout, then raises a HardIRQ. This is called Interrupt coalescence:

- The amount of traffic that a network will receive/transmit (number of frames) rx/tx-frames, or time that passes after receiving/transmitting traffic (timeout) rx/tx-usecs.
  - Interrupting too soon: poor system performance (the kernel stops a running task to handle the hardIRQ)
  - Interrupting too late: traffic isn’t taken off the NIC soon enough -> more traffic -> overwrite -> traffic loss!

- **What** - number of microseconds/frames to wait before raising a hardIRQ, from the NIC perspective it'll DMA data packets until this timeout/number of frames
- **Why** - reduce CPUs usage, hard IRQ, might increase throughput at cost of latency.
- **How:**
  - **Check command:** `ethtool -c ethX`
  - **Change command:** `ethtool -C ethX rx-usecs/rx-frames value tx-usecs/tx-frames value`
  - **How to monitor:** `show interfaces ethernet ethX statistics | grep -e "err" -e "drop" -e "over" -e "miss" -e "timeout" -e "reset" -e "restar" -e "collis" -e "over" | grep -v "\: 0"`

Recommendations: Disable adaptive interrupt moderation and set a static value. Value range is 0-236. `ethtool -C ethX adaptive-rx off adaptive-tx off ethtool -C ethX rx-usecs 64 tx-usecs 64`

### Interrupt Coalescing (soft IRQ)

The polling routine has a budget which determines the CPU time the code is allowed, by using netdev_budget_usecs timeout or netdev_budget packets. This is required to prevent SoftIRQs from monopolizing the CPU. On completion, the kernel will exit the polling routine and re-arm, then the entire procedure will repeat itself.

- **What** - maximum number of microseconds in one [NAPI](https://en.wikipedia.org/wiki/New_API) polling cycle. Polling will exit when either `netdev_budget_usecs` has elapsed during the poll cycle or the number of packets processed reaches  `netdev_budget`.
- **Why** - instead of reacting to tons of softIRQ, the driver keeps polling data; keep an eye on `dropped` (# of packets that were dropped because `netdev_max_backlog` was exceeded) and `squeezed` (# of times ksoftirq ran out of `netdev_budget` or time slice with work remaining).
- **How:**
  - **Check command:** `sysctl net.core.netdev_budget_usecs`
  - **Change command:** `set system sysctl net.core.netdev_budget_usecs value <value>`
  - **How to monitor:** 3rd column in `cat /proc/net/softnet_stat`;
- **What** - `netdev_budget` is the maximum number of packets taken from all interfaces in one polling cycle (NAPI poll). In one polling cycle interfaces that are registered to polling are probed in a round-robin manner. Also, a polling cycle may not exceed `netdev_budget_usecs` microseconds, even if `netdev_budget` has not been exhausted.
- **How:**
  - **Check command:** `sysctl net.core.netdev_budget`
  - **Change command:** `set system sysctl net.core.netdev_budget value`
  - **How to monitor:** 3rd column in`cat /proc/net/softnet_stat`;

Recommendations: If counters of the 3rd column increase, set the parameters to double their current values. Repeat this process until the counters no longer increase.

### Network device backlog queue

Per-CPU backlog queue. The netif_receive_skb() kernel function will find the corresponding CPU for a packet, and enqueue packets in that CPU’s queue. If the queue for that processor is full and already at maximum size, packets will be dropped. The default size of queue - netdev_max_backlog value is 1000, this may not be enough for multiple interfaces operating at 1Gbps, or even a single interface at 10Gbps.

- **What** - `netdev_max_backlog` is the maximum number of packets, queued on the INPUT side (*the ingress qdisc*), when the interface receives packets faster than the kernel can process them.
- **How:**
  - **Check command:** `sysctl net.core.netdev_max_backlog`
  - **Change command:** `sysctl -w net.core.netdev_max_backlog value`
  - **How to monitor:** 2nd column in `cat /proc/net/softnet_stat`;

Recommendations: If the counters of the 2nd column increase, set the parameter to double its current value. Repeat this process until the counters no longer increase.

### RSS

Receive-Side Scaling (RSS), also known as multi-queue receive, distributes network receive processing across several hardware-based receive queues, allowing inbound network traffic to be processed by multiple CPUs. RSS can be used to relieve bottlenecks in receive interrupt processing caused by overloading a single CPU and to reduce network latency.

![rss](/images/rss.png)

#### Enabling RSS

It should be enabled by default.

- **How:**
  - **Check command:** `ethtool -k ethX | grep receive-hashing`
  - **Change command:** `ethtool -K ethX rx-hashing on`

#### Adjusting the number of RX queues

- **What** - Maximum  hardware-based queue count
- **Why** - RSS should be configured so each queue goes to a different CPU core
- **How:**
  - **Check command:** `ethtool -l ethX`
  - **Change command:** `ethtool -L ethX`
Recommendations: The best value equals the number of CPU cores.

#### Adjusting the processing weight of RX queues

- **What** - Some NICs support the ability to adjust the distribution of network data among the RX queues by setting a weight.
- **Why** - According to the CPU cores load you can change the weight of cores or RSS function type.
- **How:**
  - **Check command:** `ethtool -x ethX`
  - **Change command:** `ethtool -X ethX`

#### Adjusting the rx hash fields for network flows

- **What** - Some NICs support the ability to adjust fields that will be used when computing a hash for use with RSS.
- **Why** - According to traffic specific you can change fields that will be used when computing a hash for use with RSS.
- **How:**
  - **Check command:** `ethtool -n ethX rx-flow-hash <traffic_type>`
  - **Change command:** `ethtool -N ethX rx-flow-hash <traffic_type> <function_fields>`

#### Ntuple filtering for steering network flows

Some NICs support a feature known as “ntuple filtering.” This feature allows the user to specify (via ethtool) a set of parameters to use to filter incoming network data in hardware and queue it to a particular RX queue. For example, the user can specify that TCP packets destined for a particular port should be sent to RX queue 1.
On Intel NICs this feature is commonly known as Intel Ethernet Flow Director. Other NIC vendors may have other marketing names for this feature.
As we’ll see later, ntuple filtering is a crucial component of another feature called Accelerated Receive Flow Steering (aRFS), which makes using ntuple much easier if your NIC supports it.

- **How:**
  - **Check if enabled command:** `ethtool -k ethX | grep ntuple-filters`
  - **Enabling command:** `ethtool -K ethX ntuple on`
  - **Check command:** `ethtool -u ethX`
  - **Change command:** `ethtool -U ethX flow-type <traffic_type> <function_fields>`

### RPS

Receive Packet Steering (RPS) is similar to RSS in that it is used to direct packets to specific CPUs for processing. However, RPS is implemented at the software level and helps to prevent the hardware queue of a single network interface card from becoming a bottleneck in network traffic.
RPS has several advantages over hardware-based RSS:

- RPS can be used with any network interface card.
- It is easy to add software filters to RPS to deal with new protocols.
- RPS does not increase the hardware interrupt rate of the network device. However, it does introduce inter-processor interrupts.

![rps](/images/rps.png)

For network devices with single transmit queues, the best performance can be achieved by configuring RPS to use CPUs in the same memory domain. On non-NUMA systems, this means that all available CPUs can be used. If the network interrupt rate is extremely high, excluding the CPU that handles network interrupts may also improve performance.

For network devices with multiple queues, there is typically no benefit to configuring both RPS and RSS, as RSS is configured to map a CPU to each receive queue by default. However, RPS may still be beneficial if there are fewer hardware queues than CPUs, and RPS is configured to use CPUs in the same memory domain.

#### Enabling RPS

- **How:**
  - **Change command:** `set interfaces ethernet ethX offload rps`

### RFS

Receive Flow Steering (RFS) extends RPS behavior to increase the CPU cache hit rate and thereby reduce network latency. Where RPS forwards packets based solely on queue length, RFS uses the RPS backend to calculate the most appropriate CPU, then forwards packets based on the location of the application consuming the packet. This increases CPU cache efficiency.

#### Enabling RFS

- **How:**
  - **Change command:** `set interfaces ethernet ethX offload rfs`

### aRFS

RFS can be speed up with the use of hardware acceleration; the NIC and the kernel can work together to determine which flows should be processed on which CPUs. To use this feature, it must be supported by the NIC and your driver.
Consult your NIC’s data sheet to determine if this feature is supported and how to enable it.

## NIC Tuning for IPSEC

Some NICs support ESP offloading. Try to enable it.

`ethtool -K ethX esp-hw-offload on`

Using different encryption and hashing algorithms impacts throughput performance.
For example, using AES-CGM instead AES-SHA1 increases throughput up to 50%.

## NIC Tuning for Wireguard

For site-to-site wireguard tunnel we recommend to create several tunnels and use ECMP strategy. It will balance traffic between CPU cores.

## NIC Tuning for PPPoE

If PPPoE traffic uses only one CPU core.
If your nic supports RSS, enable it and try tuning it using `ethtool -N` or `ethtool -U`.
If your nic does not support RSS, enable RPS.

## References

- <https://github.com/leandromoreira/linux-network-performance-parameters/blob/master/README.md?plain=1>
- <https://ntk148v.github.io/posts/linux-network-performance-ultimate-guide/>
- <https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf>
- <https://www.youtube.com/watch?v=MpjlWt7fvrw>
- <https://www.youtube.com/watch?v=g4w3ydS62S0>
- <https://fasterdata.es.net/host-tuning/linux/100g-tuning/>
- <https://www.intel.com/content/www/us/en/support/articles/000088688/ethernet-products/800-series-network-adapters-up-to-100gbe.html>
