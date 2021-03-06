#Multipath

---

##简介

普通的服务器都是一个硬盘挂接到一个总线上，这里是一对一的关系。而到了有FC或其他网络组成的 SAN 环境，由于主机和存储通过了交换机连接，就可以构成多对多的关系，也就是说，主机到存储可以有多条路径可以选择，主机到存储之间的 IO 由多条路径可以选择。

每个主机到所对应的存储可以经过几条不同的路径，如果是同时使用的话，I/O 流量如何分配？其中一条路径坏掉了，如何处理？角度来看，每条路径，操作系统会认为是一个实际存在的物理盘，但实际上只是通向同一个物理盘的不同路径而已，这样是在使用的时候，就给用户带来了困惑。

Multipath 就是为了解决上面的问题应运而生的。

##安装使用

安装过程较为简单：
```shell
$ sudo yum install device-mapper-multipath
```
安装完成后，会以 multipathd 的服务形式存在。

在最一开始，需要将无需使用 multipath 管理的磁盘加入到黑名单中以避免出现问题，建议尽量使用磁盘的 WWID 而不是使用磁盘名称，因为不能确保重启后磁盘仍为同一名称。
```shell
$ ls /dev/sd*
/dev/sda  /dev/sda1  /dev/sdb  /dev/sdc    #sdb与sdc为存储后端一块磁盘通过两条路径连接到本机器
$ scsi_id -g -u -s /block/sda    #获取sda的wwid
3600508e000000000dc7200032e08af0b
$ cat /etc/multipath.conf
blacklist {
    wwid 3600508e000000000dc7200032e08af0b    #wwid与devnode二选一，建议使用devnode
    devnode "^sda$"    
}
multipaths {
    multipath {
        wwid 3600508e000000000dc7200032e08af0c
        alias mpatha    #为该multipath设备创建一个别名    
    }
}
$ modprobe dm_multipath
$ systemctl restart multipathd
$ multipath -ll
mpatha (3600508e000000000dc7200032e08af0c)
size=20GB features="0" hwhandler="0" wp=rw
|-+- policy='round-robin 0' prio=50 status=active
| |- 20:0:0:0 sdb 8:16 active ready running
\-+- policy='round-robin 0' prio=10 status=enable
  |- 20:0:0:0 sdc 8:17 active ready running
$ ls /dev/mapper/
control    mpatha
```
最终可以看到生成了一块 multipath 设备且名为 mpatha。
