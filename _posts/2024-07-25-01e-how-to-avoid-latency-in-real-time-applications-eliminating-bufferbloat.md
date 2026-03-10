---
title: "01E | How to Avoid Latency in Real-Time Applications (Eliminating Bufferbloat)"
date: 2024-07-25 00:00:00 -0300
categories: [Artigos]
tags: [Bufferbloat, Latency, Networking]
image:
  path: assets/img/posts/post-01e/capa.png
---

# English Version. Originally posted on Medium on: 01/12/24

Do you know what is **bufferbloat**? and how this might be affecting your experience with gameplay online and your streaming video without you know? Are you experiencing these difficulties? I hope to help you with this knowledge!

I just could simply show you what must be done to control this problem. But, understanding all processes, will open your mind.

So, let’s go!

It’s usual to think the cause for the **lag** in games or **streaming** with poor quality/crashing is because:

- dynamic routes of the internet
- from your internet service provider (ISP)
- from the game/stream server that might be congested by high-demand
- or because of the server’s geographical location

Some settings that most people do to avoid this are opening **UDP/TCP** ports on their routers or modifying the **MTU** (Max Transmission Unit)

But, if I told you that there exists one more variable in this history (less known) and that most time, may be the biggest problem. it’s the **bufferbloat**!

### Definition:

**Bufferbloat** it’s a high latency in packet-switched networks caused by excess **buffering**¹ of packets. Mainly in your local network (the network from your house, like your router, computer, or smartphone that is connected by cable or WiFi).

**Bufferbloat** can also cause packet delay variation (also known as **jitter**), as well as reduce the overall network throughput. When a router or switch is configured to use excessively large buffers, even very high-speed networks can become practically unusable for many interactive applications like **voice over IP (VoIP)**, **audio streaming**, **online gaming**, and even ordinary web browsing.

¹ **buffer**: Basically, it’s a temporary memory that when the packets arrive on the router, these packets stay stored in memory, waiting for other packets that came first, to be forwarded.

Some communications equipment manufacturers have placed excessively large buffers in some of their networking products. in this equipment, the **bufferbloat** occurs when the network link is congested, causing packets to buffered in a queue for a long time. The excessively large buffers result in longer queues, with high latency, and don’t better network performance.

In other words, there is no point in wanting low latency in games or streaming without interruptions that are on other networks (Internet) if your internal network (intranet) is congested. And, a little spoiler: having a high-speed networking bandwidth from your ISP (ex: 300mbps) will not guarantee to solve your difficulty. That it’s the premise.

Now, that we understand the problem and how to solve it, let’s get started!

Our solution against **bufferbloat** is the **QoS (Quality of Service)**, which uses packet queue with algorithms like **CoDel**, **FQ_CoDel**, and **Cake**.

### QoS (Quality of Service)

**QoS** (Quality of Service) is the use of mechanisms or technologies that privilege data traffic on certain devices on your network and the services that are being used. (User defined). The idea of **QoS** is that not all types of traffic are equal in importance, making it possible to determine which devices and services will have the highest connection priority. **Streaming** and **multiplayer games** are examples of activities that need a better connection to work well, which is not completely necessary for system updates, exchanging messages via **WhatsApp**, and browsing the Internet, for example.

**QoS** networking technology works by tagging packets to identify service types and then configuring routers to create separate virtual queues for each application based on its priority. This way, bandwidth is reserved for essential applications that have been given priority access.

In other words, when the communication packets between your **PC** and the game server reach the router, the **QoS** will create a priority queue (as if it were a bank queue for the elderly, pregnant women, etc.), thus providing greater speed when forwarding these packets. and will avoid wasting time in the router buffering. Ensuring low latency.

An important detail: nothing it’s free, you will have to give up the total bandwidth you have, using less bandwidth on your computer to create a more effective queue. They usually indicate if you use 40% to 70% of your total bandwidth

See the picture for example:

![](assets/img/posts/post-01e/01.png)

In the image above, the green area is a queue created by the **QoS** algorithm, and the packets in this area, are game packets. See that area is smaller because this is less bandwidth. These queues mean that the first packets that arrive at the router are the first to leave. (here comes the concept of **FIFO** — **First In, First Out**, which is left as homework if you are curious to know more)

There are a lot of **QoS** algorithms that manage these queues, the most indicated to games are **CoDel** (Controlled Delay), **FQ_CoDel** (Fair Queuing Controlled Daley), and **Cake** (Common Applications Kept Enhanced). I will not do details about what’s vantages or how each works, I’ll leave it as another homework for you.

Now, let’s do some tests and apply **QoS**

On this test, will be used a **Laptop Acer Aspire 5**, with **Windows 11 22H2**, connected to the network on **WiFi 5Ghz**.

#### Test on the Speedtest website

![](assets/img/posts/post-01e/02.png)

The information highlighted in red is the latency variation in **Download** and **Upload**, respectively.

#### Another test on the **Waveform Bufferbloat** website

![](assets/img/posts/post-01e/03.png)

This site will give more details about latency and also a grade depending on the result.

Note that the test received a grade of **B** and a warning that **bufferbloat** may occur in **Gaming**.

### Now let’s get to the solution.

I will use the **Mikrotik RB750GR3** router in the **RouterOSv7.5** version with the **FQ_CoDel** algorithm. (This and other algorithms mentioned previously were added in version 7.1)

#### **QoS Algorithms in RouterOS**

![](assets/img/posts/post-01e/04.png)

Video showing the configuration process on **RouterOS**:

Applying **FQ_CoDel** type **QoS** to the **IP** assigned to my notebook and setting the total bandwidth to **50Mb Download** and **20Mb Upload** (could be more)

![](assets/img/posts/post-01e/05.png)

In my case I only need to define the **IP** of the priority machine, not needing to define the type of service (some router manufacturers have this option)

Let’s go back to the new tests:

#### **Speedtest**

![](assets/img/posts/post-01e/06.png)

#### **Waveform Bufferbloat**

![](assets/img/posts/post-01e/07.png)

#### Monitoring test traffic

![](assets/img/posts/post-01e/08.png)

Just look at the difference after **QoS** works. No **latency** and **bufferbloat**!

Now, when playing using this machine, all game packets will have priority on my internal network, drastically reducing high latency. If I’m no longer playing, just disable **QoS** and I can have all the bandwidth contracted for the laptop again.

Apply **QoS** and do your tests. If you don’t have a **Mikrotik** router, there are several others that have this functionality. Like **EdgeRouter X**, **NetGear** etc.

**Author**: Jefferson J. Raimon

--- 

### References and further reading:

- [Bufferbloat - Wikipedia](https://en.wikipedia.org/wiki/Bufferbloat)
- [Bufferbloat: What Can I Do About It?](https://www.bufferbloat.net/projects/bloat/wiki/What_can_I_do_about_Bufferbloat)
- [Fortinet - QoS (Quality of Service)](https://www.fortinet.com/br/resources/cyberglossary/qos-quality-of-service)
- [GeeksforGeeks - CoDel Queue Discipline](https://www.geeksforgeeks.org/controlled-delay-codel-queue-discipline)
- [pfSense - CoDel Limiters](https://docs.netgate.com/pfsense/en/latest/recipes/codel-limiters.html)
- [pfSense - Altq Scheduler Types](https://docs.netgate.com/pfsense/en/latest/trafficshaper/altq-scheduler-types.html#altq-codel)
- [Clube do Hardware - Mikrotik FQ_CoDel ou Cake Queue](https://www.clubedohardware.com.br/forums/topic/1581897-mikrotik-fq_codel-ou-cake-queue)
- [Mikrotik - Queues Documentation](https://help.mikrotik.com/docs/display/ROS/Queues)