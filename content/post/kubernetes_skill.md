---
title: "Kubernetes_skill"
date: 2018-07-06T16:52:51+08:00
draft: false
subtitle:
bigimg: [{src: "", desc: ""}]
tags: []
---

<!--more-->
记录kubernetes的日常使用以及应用场景构思。


1、kubernetes是通过pod中容器的hostname来注册，当我们手动或者程序修改了hostname，应该就会出现服务异常；对于如何预防，通过gosu来设置程序运行时的用户限制使用root权限，防止出现安全事件。
2、centos系统内核为3.10.0-229.7.2.el7.x86_64版本，会导致节点服务器重启，丢失kubelet相关信息，查看另一台内核同样为3.10.0-229.7.2.el7.x86_64的机器同样出现重新现象导致节点kubelet相关信息丢失，内核为3.10.0-693.el7.x86_64则没有出现该现象。怀疑是kube-router功能不支持低版本内核。待确认原因
3、因为节点主机不支持overlay2的存储类型
```shell
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=info msg="loading plugin "io.containerd.snapshotter.v1.btrfs"..." module=containerd type=io.containerd.snapshotter.v1
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=warning msg="failed to load plugin io.containerd.snapshotter.v1.btrfs" error="path /var/lib/docker/containerd/daemon/io.containerd.snapshotter.v1.btrfs must
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=info msg="loading plugin "io.containerd.snapshotter.v1.overlayfs"..." module=containerd type=io.containerd.snapshotter.v1
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=warning msg="failed to load plugin io.containerd.snapshotter.v1.overlayfs" error="/var/lib/docker/containerd/daemon/io.containerd.snapshotter.v1.overlayfs do
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=info msg="loading plugin "io.containerd.metadata.v1.bolt"..." module=containerd type=io.containerd.metadata.v1
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=warning msg="could not use snapshotter btrfs in metadata plugin" error="path /var/lib/docker/containerd/daemon/io.containerd.snapshotter.v1.btrfs must be a b
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=warning msg="could not use snapshotter overlayfs in metadata plugin" error="/var/lib/docker/containerd/daemon/io.containerd.snapshotter.v1.overlayfs does not
Jul 06 18:26:06 node3 dockerd[10669]: time="2018-07-06T18:26:06+08:00" level=info msg="loading plugin "io.containerd.differ.v1.walking"..." module=containerd type=io.containerd.differ.v1
```
4、节点主机自动关机，在master机器上发现没有几秒内进行转移服务容器，待确定是在哪配置的时间项
5、在pv的服务存储卷节点机器上停止docker，待确认存储卷内容是否会进行转移，实现分布式存储；确定调度时间以及etcd性能是否有影响调度
6、在使用openebs作为存储介质，机器是通过vmare安装在windows的虚拟机，日志有报
```shell
[root@node5 fission-functions]# uname -a
Linux node5 3.10.0-693.el7.x86_64 #1 SMP Tue Aug 22 21:09:27 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux

Jul 10 09:40:06 node5 kernel: connection4:0: ping timeout of 5 secs expired, recv timeout 5, last rx 4809172764, last ping 4809179730, now 4809186632
Jul 10 09:40:06 node5 kernel: connection4:0: detected conn error (1022)
Jul 10 09:40:13 node5 kernel: NMI watchdog: BUG: soft lockup - CPU#0 stuck for 21s! [server:102065]
Jul 10 09:40:13 node5 kernel: Modules linked in: ip_vs_sh ip_vs_wrr ext4 mbcache jbd2 ipt_REJECT nf_reject_ipv4 xt_nat xt_recent veth nf_conntrack_netlink ipip tunnel4 ip_tunnel ip_vs_rr dummy xt_ipvs xt_set ip_vs ip_set_hash_ip ip_set_hash_net ip_set nfnetlink xt_comment xt_mark ipt_MASQUERADE nf_nat_masquerade_ipv4 iptable_nat nf_conntrack_ipv4 nf_defrag_ipv4 nf_nat_ipv4 xt_addrtype iptable_filter xt_conntrack nf_nat nf_conntrack br_netfilter bridge stp llc overlay(T) iscsi_tcp libiscsi_tcp libiscsi scsi_transport_iscsi snd_seq_midi snd_seq_midi_event snd_ens1371 snd_rawmidi snd_ac97_codec ac97_bus coretemp snd_seq snd_seq_device snd_pcm snd_timer snd ppdev iosf_mbi crc32_pclmul soundcore ghash_clmulni_intel vmw_balloon cryptd nfit libnvdimm sg pcspkr joydev parport_pc parport vmw_vmci shpchp i2c_piix4 ip_tables
Jul 10 09:40:13 node5 kernel: xfs libcrc32c sr_mod cdrom ata_generic pata_acpi sd_mod crc_t10dif crct10dif_generic vmwgfx drm_kms_helper syscopyarea sysfillrect sysimgblt fb_sys_fops ttm drm ata_piix libata crct10dif_pclmul crct10dif_common mptspi crc32c_intel scsi_transport_spi mptscsih i2c_core serio_raw mptbase e1000 dm_mirror dm_region_hash dm_log dm_mod
Jul 10 09:40:13 node5 kernel: CPU: 0 PID: 102065 Comm: server Tainted: G             L ------------ T 3.10.0-693.el7.x86_64 #1
Jul 10 09:40:13 node5 kernel: Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 07/02/2015
Jul 10 09:40:13 node5 kernel: task: ffff88003d064f10 ti: ffff88013d104000 task.ti: ffff88013d104000
Jul 10 09:40:13 node5 kernel: RIP: 0033:[<0000000000419899>]  [<0000000000419899>] 0x419898
Jul 10 09:40:13 node5 kernel: RSP: 002b:000000c420033e50  EFLAGS: 00010283
Jul 10 09:40:13 node5 kernel: RAX: 00007f9e3e777a70 RBX: ffffffff816b50eb RCX: 0000000000000057
Jul 10 09:40:13 node5 kernel: RDX: 000000000000004e RSI: 0000000000000000 RDI: 0000000000000368
Jul 10 09:40:13 node5 kernel: RBP: 000000c420033eb0 R08: 0000000000000001 R09: 00007f9e3e70f0a0
Jul 10 09:40:13 node5 kernel: R10: 000000000073a1b8 R11: 000000c420014a20 R12: 0000000000000010
Jul 10 09:40:13 node5 kernel: R13: 00007f9e3e76f4b0 R14: 0000000000000020 R15: 000000000000007c
Jul 10 09:40:13 node5 kernel: FS:  000000c420026868(0000) GS:ffff8801a1a00000(0000) knlGS:0000000000000000
Jul 10 09:40:13 node5 kernel: CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
Jul 10 09:40:13 node5 kernel: CR2: 0000000003a741d7 CR3: 000000019b62d000 CR4: 00000000000407f0
Jul 10 09:40:13 node5 kernel: DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
Jul 10 09:40:13 node5 kernel: DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
```