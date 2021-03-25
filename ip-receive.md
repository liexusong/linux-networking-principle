# IP数据包接收过程

在《网络中断下半部处理》一文中介绍过，当网卡接收到网络数据包后，会由网卡驱动通过调用 `netif_rx` 函数把数据包添加到待处理队列中，然后唤起网络中断下半部处理。

而网络中断下半部处理由 `net_rx_action` 函数完成的，其主要功能就是从待处理队列中获取一个数据包，然后根据数据包的网络层协议类型来找到相应的处理接口来处理数据包。我们通过 图1 来展示 `net_rx_action` 函数的处理过程：

![net_rx_action](https://raw.githubusercontent.com/liexusong/understanding-the-linux-networking/master/images/net_rx_action.png)

图1 `net_rx_action` 处理过程

网络层的处理接口保存在 `ptype_base` 数组中，其元素的类型为 `packet_type` 结构，而索引为网络层协议的类型，所以通过网络层协议的类型就能找到对应的处理接口，我们先来看看 `packet_type` 结构的定义：

```c
struct packet_type
{
    unsigned short      type;   // 网络层协议类型
    struct net_device   *dev;   // 绑定的设备对象, 一般设置为NULL
    int                 (*func)(struct sk_buff *, struct net_device *, struct packet_type *); // 处理接口
    void                *data;  // 私有数据域, 一般设置为NULL
    struct packet_type  *next;
};
```

`ptype_base` 数组的定义如下：

```c
static struct packet_type *ptype_base[16];
```

从 `ptype_base` 数组的定义可知，其只有 16 个元素，那么如果内核超过 16 种网络层协议怎么办？我们可以从网络层处理接口的注册函数 `dev_add_pack` 中找到答案：

```c
void dev_add_pack(struct packet_type *pt)
{
    int hash;

    br_write_lock_bh(BR_NETPROTO_LOCK);

    // 如果类型为ETH_P_ALL, 表示需要处理所有协议的数据包
    if (pt->type == htons(ETH_P_ALL)) { 
        netdev_nit++;
        pt->next = ptype_all;
        ptype_all = pt;
    } else {
        // 对网络层协议类型与15进行与操作, 得到一个 0 到 15 的整数
        hash = ntohs(pt->type) & 15;

        // 通过 next 指针把冲突的处理接口连接起来
        pt->next = ptype_base[hash];
        ptype_base[hash] = pt;
    }

    br_write_unlock_bh(BR_NETPROTO_LOCK);
}
```

从 `dev_add_pack` 函数的实现可知，内核把 `ptype_base` 数组当成了哈希表，而键值就是网络层协议类型，哈希函数就是对协议类型与 15 进行与操作。如果有冲突，就通过 `next` 指针把冲突的处理接口连接起来。

我们再来看看 IP 协议是怎样注册处理接口的，如下代码：

```c
/* /net/ipv4/ip_output.c */

static struct packet_type ip_packet_type = {
     __constant_htons(ETH_P_IP),
     NULL,
     ip_rcv,  // 处理 IP 协议数据包的接口
     (void*)1,
     NULL,
 };
 
 void __init ip_init(void)
{
     // 注册 IP 协议处理接口
     dev_add_pack(&ip_packet_type);
     ...
 }
```

从上面的代码可以看到，在 `ip_init` 函数中调用了 `dev_add_pack(&ip_packet_type)` 函数来注册了 IP 协议的处理接口，而 IP 协议的处理接口为 `ip_rcv` 函数。

我们再看看 `net_rx_action` 函数的处理：

```c
static void net_rx_action(struct softirq_action *h)
{
    ...
    for (;;) {
        ...
        {
            struct packet_type *ptype, *pt_prev;
            unsigned short type = skb->protocol; // 网络层协议类型
 
            pt_prev = NULL;
            ...
            // 从 ptype_base 数组中查找网络层处理接口
            for (ptype = ptype_base[ntohs(type) & 15]; 
                 ptype; 
                 ptype = ptype->next) 
            {
                if (ptype->type == type // 如果协议类型匹配
                    && (!ptype->dev || ptype->dev == skb->dev))
                {
                    if (pt_prev) {
                        atomic_inc(&skb->users);
                        // 如处理IP协议数据包的 ip_rcv() 函数
                        pt_prev->func(skb, skb->dev, pt_prev);
                    }

                    pt_prev = ptype;
                }
            }
            ...
        }
        ...
    }
    ...
    return;
}
```

现在就非常清晰了，就是根据数据包的网络层协议类型，然后从 `ptype_base` 数组中找到对应的处理接口处理数据包，如 IP 协议的数据包就调用 `ip_rcv` 函数处理。

