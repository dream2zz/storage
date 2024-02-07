# GlusterFS 快速入门

本章节将指导你在最短时间内，成功地快速部署完成一套GlusterFS管理系统。然后基于我们刚刚部署完成的GlusterFS系统，再深入说明系统架构原理和管理员需要具备的相关技能要求。对于测试环境可以使用/etc/hosts来定义主机名与IP的映射关系，对于生产环境建议部署DNS服务提供解析服务。此外，生产环境中使用GlusterFS，应该部署和使用NTP服务器。


# 一、准备最少两个节点
* 名为 server1 及 server2 的两个 CentOS 7 服务器 （xfs文件系统）；
* 主机名分别命名为server1和server2；
* 有效的网络连接 
* 每台主机至少两块磁盘，一块用于安装OS，其它的用于GlusterFS storage；
* GlusterFS 默认地把动态生成的配置数据存放于/var/lib/glusterd目录下，日志数据放于/var/log下，请确保它们所在的分区空间足够多，避免因磁盘满而导致GlusterFS 运行故障或服务当机；


```
systemctl stop firewalld.service
systemctl disable firewalld.service
```

# 二、格式化及挂载砖块
（在两个节点）： > 注：这些样例假设brick将会位于 /dev/sdb1。
```
mkfs.xfs -i size=512 /dev/sdb1
mkdir -p /data/brick1
echo '/dev/sdb1 /data/brick1 xfs defaults 1 2' >> /etc/fstab
mount -a && mount
```

# 三、安装 GlusterFS
（在两个节点） 安装所需软件：
```
yum install centos-release-gluster -y
yum install glusterfs-server -y
```

启动 GlusterFS 的守护程序：
```
systemctl enable glusterd

systemctl start glusterd
systemctl status glusterd
glusterd.service - LSB: glusterfs server
        Loaded: loaded (/etc/rc.d/init.d/glusterd)
    Active: active (running) since Mon, 13 Aug 2012 13:02:11 -0700; 2s ago
    Process: 19254 ExecStart=/etc/rc.d/init.d/glusterd start (code=exited, status=0/SUCCESS)
    CGroup: name=systemd:/system/glusterd.service
        ├ 19260 /usr/sbin/glusterd -p /run/glusterd.pid
        ├ 19304 /usr/sbin/glusterfsd --xlator-option georep-server.listen-port=24009 -s localhost...
        └ 19309 /usr/sbin/glusterfs -f /var/lib/glusterd/nfs/nfs-server.vol -p /var/lib/glusterd/...
```

# 四、设置互信组别
在 server1 上
```
gluster peer probe server2
```
> 注：采用主机名称时，首台服务器必须获另一台服务器侦察才能设置主机名称。

在 server2 上
```
gluster peer probe server1
```
> 注：设立组别后，只有获信任的成员才能侦察新服务器加进组别。一台新的服务器不能侦察组别，它必须由组别侦察出来。

# 五、设立 GlusterFS 扇区
在 server1 及 server2 上：
```
mkdir /bricks/brick1/gv0
```
在任何一台服务器上：
```
gluster volume create gv0 replica 2 server1:/bricks/brick1/gv0 server2:/bricks/brick1/gv0
gluster volume start gv0
```
确定扇区汇报 Started（已引导）：
```
gluster volume info
```
> 注：要是扇区未能引导，你可在其中一台或两台服务器的 /var/log/glusterfs 目录下的日志档找寻有关错误的线索 —— 信息多数记录在 etc-glusterfs-glusterd.vol.log

# 六、测试 GlusterFS 扇区
在此步骤，我们将会利用其中一台服务器挂载扇区。一般来说，你应该在另一台计算机，称为「客端」，进行此动作。但由于这个步骤涉及在客端计算机上安装额外的组件，因此服务器为我们的首轮测试提供了一个方便的测试点。
```
mount -t glusterfs server1:/gv0 /mnt
for i in `seq -w 1 100`; do cp -rp /var/log/messages /mnt/copy-test-$i; done
```
首先，检查挂载点：
```
ls -lA /mnt | wc -l
```
你应该看见 100 个文件。 接著，检查每台服务器的 GlusterFS 挂载点：
```
ls -lA /bricks/brick1/gv0
```
采用上述的测试方法，你应该在每台服务器上看见 100 个文件。在一个不复制、分布式的扇区上（不会在此详述），你应该在每台机器看见约 50 个文件。

# 总结

就这样了，这大概是最快速的方法。

阅读 Admin Guide 及 New User Guide 来取得更深入的理解，但这里已经为测试提供了一个很好的起点。

本篇源自 Gluster 社群的快速入门指南。

# ubuntu 

```
apt-get install software-properties-common
add-apt-repository ppa:gluster/glusterfs-3.10
apt-get update
```
```
Server# apt-get install glusterfs-server -y
Client# apt-get install glusterfs-client -y
```
```
创建volume
Server# gluster volume create g3 172.18.26.14:/g3 force
启动volume
Server# gluster volume start g3
查看当前所有volume状态
Server# gluster volume info
若要使用Cache,则使用
Server# gluster volume set single-volume performance.cache-size 1GB
```
> Gluster自动生成配置文件，存储在 /var/lib/glusterd 文件夹中
```
Client# mount.glusterfs 192.168.113.173:volume-name /mnt/local-volume
```