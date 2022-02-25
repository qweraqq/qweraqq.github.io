---
layout: post
title: "突破光猫千兆限制"
nav_order: 4
date: 2022-01-01 00:00:00 +0800
author: xiangxiang
categories: env-settings edgeos
tags: [网络折腾 F1855v2 er12]
---

 ![](/img/CT-OVER-1G.png){:width="512px"}

* auto-gen TOC:
{:toc}

## 要点
- 光猫为电信SDN光猫ZTE F1855V2, 已经改桥接
- bond0: balance-rr模式进行PPPoE拨号, 连接光猫的LAN1和LAN4口
- bond2: 802.3ad(LACP) hash-policy为layer3+4作为LAN接入交换机(MikroTik CSS610)
- er12的bonding默认不能PPPoE, 需要简单的hacking

## 0 光猫
- SDN光猫没有啥可配置的
- 已知LAN3口把IPTV的VLAN接了出来(数据绑定)
- LAN2(itv口)可能是IPTV的端口绑定
- LAN1和LAN4应该是正常的口, 选择这两个进行bonding


## 1 ER-12
- ER12共有12个口, eth0-eth7是有交换机芯片的(到CPU是4G), eth8/eth9/sfp0/sfp1都是直连CPU(到CPU都是1G), 所以参数一共是8G
- bond0: eth0-eth3, balance-rr, 用于PPPoE, 实际只连接了两个口
- bond2: eth4-eth7, 802.3ad(LACP), hash-policy为layer3+4, 作为LAN口
- 关闭可能影响offload的功能: 关闭switch的VLAN-AWARE及ipv6
- 注意对于上海电信国际精品网PPPoE MTU=1442 MSS=1402
- 如果要使用IPTV, 可以切换为dnsmasq并增加DHCP-force-option
- bonding口增加PPPoE的支持[https://community.ui.com/questions/PPPoE-client-not-supported-on-bonded-interfaces/b96fb5f7-6fe5-47ea-9242-9fc9083ff051](https://community.ui.com/questions/PPPoE-client-not-supported-on-bonded-interfaces/b96fb5f7-6fe5-47ea-9242-9fc9083ff051)
```bash
sudo su
cp -a /opt/vyatta/share/vyatta-cfg/templates/interfaces/ethernet/node.tag/pppoe /opt/vyatta/share/vyatta-cfg/templates/interfaces/bonding/node.tag/
```

```text
firewall {
    all-ping enable
    broadcast-ping disable
    ipv6-receive-redirects disable
    ipv6-src-route disable
    ip-src-route disable
    log-martians enable
    name WAN_IN {
        default-action drop
        description "WAN to internal"
        rule 10 {
            action accept
            description "Allow established/related"
            state {
                established enable
                related enable
            }
        }
        rule 20 {
            action drop
            description "Drop invalid state"
            state {
                invalid enable
            }
        }
    }
    name WAN_LOCAL {
        default-action drop
        description "WAN to router"
        rule 10 {
            action accept
            description "Allow established/related"
            state {
                established enable
                related enable
            }
        }
        rule 20 {
            action drop
            description "Drop invalid state"
            state {
                invalid enable
            }
        }
    }
    options {
        mss-clamp {
            mss 1400
        }
    }
    receive-redirects disable
    send-redirects enable
    source-validation disable
    syn-cookies enable
}
interfaces {
    bonding bond0 {
        description "Internet(PPPoE or DHCP)"
        firewall {
            in {
                name WAN_IN
            }
            local {
                name WAN_LOCAL
            }
        }
        hash-policy layer2
        mode round-robin
        pppoe 0 {
            default-route auto
            firewall {
                in {
                    name WAN_IN
                }
                local {
                    name WAN_LOCAL
                }
            }
            mtu 1442
            name-server auto
            password ...
            user-id ...
        }
    }
    bonding bond2 {
        address 192.168.88.1/24
        description LAN
        hash-policy layer3+4
        mode 802.3ad
    }
    ethernet eth0 {
        bond-group bond0
        duplex auto
        speed auto
    }
    ethernet eth1 {
        bond-group bond0
        duplex auto
        speed auto
    }
    ethernet eth2 {
        bond-group bond0
        duplex auto
        speed auto
    }
    ethernet eth3 {
        bond-group bond0
        duplex auto
        speed auto
    }
    ethernet eth4 {
        bond-group bond2
        duplex auto
        speed auto
    }
    ethernet eth5 {
        bond-group bond2
        duplex auto
        speed auto
    }
    ethernet eth6 {
        bond-group bond2
        duplex auto
        speed auto
    }
    ethernet eth7 {
        bond-group bond2
        duplex auto
        speed auto
    }
    ethernet eth8 {
        duplex auto
        speed auto
    }
    ethernet eth9 {
        address 10.0.0.1/24
        duplex auto
        poe {
            output off
        }
        speed auto
    }
    ethernet eth10 {
        duplex auto
        speed auto
    }
    ethernet eth11 {
        duplex auto
        speed auto
    }
    loopback lo {
    }
    switch switch0 {
        mtu 1500
        switch-port {
            vlan-aware disable
        }
    }
}
port-forward {
    auto-firewall enable
    hairpin-nat enable
    lan-interface bond2
    rule 1 {
        ...
    }
    wan-interface pppoe0
}
protocols {
    static {
    }
}
service {
    dhcp-server {
        disabled false
        hostfile-update disable
        shared-network-name LAN1 {
            authoritative enable
            subnet 192.168.88.1/24 {
                ...
            }
        }
        static-arp disable
        use-dnsmasq enable
    }
    dns {
        dynamic {
            interface pppoe0 {
                ...
            }
        }
        forwarding {
            cache-size 10000
            listen-on bond2
            name-server 223.5.5.5
            name-server 119.29.29.29
            options no-resolv
            system
        }
    }
    nat {
        rule 5010 {
            description "masquerade for WAN"
            outbound-interface pppoe0
            type masquerade
        }
    }
    ssh {
        port 22
        protocol-version v2
    }
    ubnt-discover {
        disable
    }
    unms {
        ...
    }
}
system {
    analytics-handler {
        send-analytics-report false
    }
    crash-handler {
        send-crash-report false
    }
    host-name EdgeRouter-12
    ipv6 {
        disable
    }
    login {
        ...
    }
    name-server 223.5.5.5
    ntp {
        server 0.ubnt.pool.ntp.org {
        }
        server 1.ubnt.pool.ntp.org {
        }
        server 2.ubnt.pool.ntp.org {
        }
        server 3.ubnt.pool.ntp.org {
        }
    }
    offload {
        hwnat disable
        ipv4 {
            bonding enable
            forwarding enable
            gre enable
            pppoe enable
            vlan enable
        }
        ipv6 {
            bonding enable
            forwarding enable
            pppoe disable
        }
    }
    package {
        repository stretch {
            components "main contrib non-free"
            distribution stretch
            password ""
            url http://http.us.debian.org/debian
            username ""
        }
    }
    time-zone Asia/Shanghai
}
```
## 2 交换机配置
- 交换机配置主要是对应配置LACP的端口即可
- 由于还需要IPTV, 额外连了光猫的LAN3(数据绑定口)将VLAN接了出来

![](/img/CT-OVER-1G-2.jpg)
