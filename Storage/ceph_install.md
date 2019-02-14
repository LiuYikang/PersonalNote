## 1 虚拟机环境
* 创建三个虚拟机，虚拟机centos-7.2
    * node1 192.168.0.181
    * node2 192.168.0.182
    * node3 192.168.0.183
* 给虚拟机挂载一块虚拟磁盘
    * virsh命令找到虚拟机的uuid
    ```shell
    [root@centos ceph]# virsh list
     Id    Name                           State
    ----------------------------------------------------
     202   13b504e9-2431-40da-be32-3ca8f1f586ec running
     203   705caedf-dea8-44d1-9ebb-a610bf849d6d running
     204   02b392d4-5134-4959-8475-a0321f0f0ccd running
    ```
    * 找到虚拟机磁盘的存储路径，在该路径下创建一个10G的qcow2文件作为虚拟磁盘，以202这台虚拟机为例
    ```shell
    [root@centos 13b504e9-2431-40da-be32-3ca8f1f586ec]# qemu-img create -f qcow2 vdb.qcow2 10G
    [root@centos 13b504e9-2431-40da-be32-3ca8f1f586ec]# ls
    disk  vdb.qcow2
    ```
    * 编辑虚拟机的配置文件，增加以下vdb.qcow2的配置内容
    ```xml
    <devices>
       <emulator>/usr/libexec/qemu-kvm</emulator>
       <disk type='file' device='disk'>
         <driver name='qemu' type='qcow2' cache='writeback'/>
         <source file='/opt/instances/13b504e9-2431-40da-be32-3ca8f1f586ec/disk'/>
         <target dev='vda' bus='virtio'/>
         <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
       </disk>
       <disk type='file' device='disk'>
         <driver name='qemu' type='qcow2' cache='none'/>
         <source file='/opt/instances/13b504e9-2431-40da-be32-3ca8f1f586ec/vdb.qcow2'/>
         <target dev='vdb' bus='virtio'/>
         <address type='pci' domain='0x0000' bus='0x00' slot='0x09' function='0x0'/>
       </disk>
    ```
    * 重启虚拟机使挂载生效
    ```shell
    [root@centos ~]# virsh destroy 202 && virsh start 13b504e9-2431-40da-be32-3ca8f1f586ec
    ```
* 关闭虚拟机的防火墙
    ```shell
    systemctl stop firewalld
    systemctl disable firewalld
    ```
* 配置三个节点之间的ssh互信
    * 在每个节点上的/etc/hosts文件增加以下配置信息
    ```
    192.168.0.181  node1
    192.168.0.182  node2
    192.168.0.183  node3
    ```
    * 在每个节点上执行ssh-keygen，连续回车确认，生成ssh的秘钥
    * 拷贝秘钥到其他节点，以node1为例
    ```shell
    ssh-copy-id node1
    ssh-copy-id node2
    ssh-copy-id node3
    ```
* 配置公司的nds，编辑/etc/resolv.conf
    ```
    ; generated by /usr/sbin/dhclient-script
    nameserver 10.1.7.77
    ```
* 配置yum源
    ```shell
    yum clean all
    yum makecache
    ```
* 配置节点之间的ntp同步
    * 每个节点安装ntp
    ```shell
    yum install ntp
    ```
    * 配置node1作为ntp服务器，修改ntp服务器以下部分的内容
    ```
    # restrict default nomodify notrap nopeer noquery
    restrict 192.168.0.0 mask 255.255.255.0 nomodify notrap

    # server 0.centos.pool.ntp.org iburst
    # server 1.centos.pool.ntp.org iburst
    # server 2.centos.pool.ntp.org iburst
    # server 3.centos.pool.ntp.org iburst
    server  127.127.1.0     # local clock
    fudge   127.127.1.0 stratum 10
    ```
    * 启动ntpd服务，将node1作为ntp服务器
    ```shell
    systemctl enable ntpd
    systemctl start ntpd
    ```
    * node2、node3配置ntp定时任务，将下列内容写到/etc/crontab中
    ```shell
    0 * * * * /usr/sbin/ntpdate -u 192.168.0.181
    ```
## 2 ceph的部署和配置
### 2.1 ceph的安装
ceph需要额外安装源地址，这里安装的ceph-luminous版本的源地址，操作流程如下（以下均以root用户运行）
```shell
# 安装源地址
yum install centos-release-ceph-luminous.noarch

# 更新yum源
yum clean all && yum makecache

# 安装ceph
yum install ceph
```

> 注意：ceph安装完成之后，会有一个ceph用户用来运行ceph的程序，注意文件的权限
### 2.2 ceph monitor的配置
* 生成一个随机的uuid作为ceph集群的id，本例子的id为：676a90f3-bcd1-4c62-9d26-251b17ebcb2a
    ```shell
    uuidgen
    ```
* 配置/etc/ceph/ceph.conf文件
    ```
    [global]
    fsid = 676a90f3-bcd1-4c62-9d26-251b17ebcb2a          #生成的uuid
    mon initial members = node1,node2,node3              #ceph集群中的三个节点
    mon host = 192.168.0.181,192.168.0.182,192.168.0.183    #节点的ip

    auth cluster required = cephx
    auth service required = cephx
    auth client required = cephx

    public network = 192.168.0.0/24                       #网络配置
    cluster network = 192.168.0.0/24

    mon pg warn max per osd = 1000
    mon_allow_pool_delete = true

    osd journal size = 7168

    osd pool default size = 2
    osd pool default min size = 1

    osd pool default pg num = 512
    osd pool default pgp num = 512

    rbd default features = 3

    osd crush update on start = true

    mon clock drift allowed = 2
    mon clock drift warn backoff = 30
    ```
* 生成monitor secret key，注意生成的/tmp/ceph.mon.keyring，所属用户需要是ceph
    ```shell
    sudo -u ceph ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
    ```
* 生成administrator keyring，并绑定client.admin用户
    ```shell
    ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
    ```
* 生成bootstrap-osd keyring，并绑定client.bootstrap-osd用户
    ```shell
    ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd'
    ```
* 将生成的administrator keyring和bootstrap-osd keyring加入ceph.mon.keyring
    ```shell
    sudo -u ceph ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
    sudo -u ceph ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
    ```
* 生成monitor map到/tmp/monmap
    ```shell
    sudo -u ceph monmaptool --create --add node1 192.168.0.181 --add node2 192.168.0.182 --add node3 192.168.0.183 --fsid 676a90f3-bcd1-4c62-9d26-251b17ebcb2a /tmp/monmap
    ```
* **拷贝/etc/ceph/ceph.conf /tmp/monmap /tmp/ceph.mon.keyring到node2和node3**

* 以下操作在所有node上都要执行，这里以node1为例
    ```shell
    # 创建monitor的目录
    sudo -u ceph mkdir /var/lib/ceph/mon/ceph-node1

    # 配置monitor的守护进程
    sudo -u ceph ceph-mon --mkfs -i node1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring

    # 启动monitor服务
    systemctl start ceph-mon@node1
    ```
> 注意，在重启node2和node3的monitor进程的时候，需要先手动进行ntp的时间同步，执行 ntpdate -u 192.168.0.181

* 查看ceph集群
    ```shell
    ceph -s
    ```

### 2.3 ceph mgr配置
* 为每一个节点确定一个mgr daemon的名字，本例使用的名字如下，后续操作以node1为例，node2和node3均需要进行操作
    ```
    node1:  nodes1
    node2:  nodes2
    node3:  nodes3
    ```
* 为node1的mgr daemon创建一个key
    ```shell
    ceph auth get-or-create mgr.node1 mon 'allow profile mgr' osd 'allow *' mds 'allow *'
    ```
* 创建node1的mgr目录，ceph-node1的ceph是ceph集群的名字，默认是ceph
    ```shell
    sudo -u ceph mkdir /var/lib/ceph/mgr/ceph-node1
    ```
* 将之前生成的key导入/var/lib/ceph/mgr/ceph-node1/keyring
    ```shell
    sudo -u ceph ceph auth get mgr.node1 -o  /var/lib/ceph/mgr/ceph-node1/keyring
    ```
* 启动mgr
    ```shell
    ceph-mgr -i node1
    ```
* 查看ceph的状态，完成以上monitor和mgr的配置步骤后，ceph的状态如下
    ```shell
    [root@host-192-168-0-181 ~]# ceph -s
      cluster:
        id:     676a90f3-bcd1-4c62-9d26-251b17ebcb2a
        health: HEALTH_OK

      services:
        mon: 3 daemons, quorum node1,node2,node3
        mgr: node1(active), standbys: node2, node3
        osd: 0 osds: 0 up, 0 in

      data:
        pools:   0 pools, 0 pgs
        objects: 0 objects, 0 bytes
        usage:   0 kB used, 0 kB / 0 kB avail
        pgs:
    ```
### 2.4 配置osd进程
**以下osd的配置需要在所有节点上运行**

* 为osd生成一个uuid
    ```shell
    UUID=$(uuidgen)
    ```
* 为osd生成一个cephx key
    ```shell
    OSD_SECRET=$(ceph-authtool --gen-print-key)
    ```
* 配置osd，并获取osd id
    ```shell
    ID=$(echo "{\"cephx_secret\": \"$OSD_SECRET\"}" | ceph osd new $UUID -i - -n client.bootstrap-osd -k /tmp/ceph.mon.keyring)
    ```
* 生成osd的目录
    ```shell
    sudo -u ceph mkdir /var/lib/ceph/osd/ceph-$ID
    ```
* 格式化磁盘并挂载到osd目录，**此处虚拟机内部空闲的磁盘为 /dev/vdb**，如果不一样需要进行修改
    ```
    mkfs.xfs /dev/vdb
    mount /dev/vdb /var/lib/ceph/osd/ceph-$ID
    ```
* 写入osd的keyring
    ```shell
    ceph-authtool --create-keyring /var/lib/ceph/osd/ceph-$ID/keyring --name osd.$ID --add-key $OSD_SECRE
    ```
* 初始化osd的数据目录
    ```shell
    ceph-osd -i $ID --mkfs --osd-uuid $UUID
    ```
* 修改osd目录权限
    ```shell
    chown -R ceph:ceph /var/lib/ceph/osd/ceph-$ID
    ```
* 移动osd到合适的位置
    ```shell
    ceph osd crush add-bucket node1 host
    ceph osd crush move node1 root=default
    ceph osd crush add osd.$ID 1.0 host=node1
    ```
* 启动osd服务
    ```shell
    systemctl enable ceph-osd@$ID
    systemctl start ceph-osd@$ID
    ```
* node1、node2和node3都配置完成后，查询osd如下
    ```shell
    [root@host-192-168-0-181 ceph-common]# ceph osd tree
    ID  CLASS WEIGHT  TYPE NAME                  STATUS REWEIGHT PRI-AFF
     -1       3.00000 root default
     -9       1.00000     host host-192-168-0-181
      0   hdd 1.00000         osd.0                  up  1.00000 1.00000
    -13       1.00000     host host-192-168-0-182
      1   hdd 1.00000         osd.1                  up  1.00000 1.00000
    -11       1.00000     host host-192-168-0-183
      2   hdd 1.00000         osd.2                  up  1.00000 1.00000
     -3             0     host node1
     -5             0     host node2
     -7             0     host node3
    ```
### 2.5 创建rbd pool
* 创建pool
    ```shell
    ceph osd pool create c360 128 128
    ceph osd pool create kube 128 128
    ```
* 查看pool
    ```shell
    [root@host-192-168-0-181 ~]# ceph osd pool ls
    c360
    kube
    ```
* 查看ceph状态
    ```shell
    [root@host-192-168-0-181 ~]# ceph -s
      cluster:
        id:     676a90f3-bcd1-4c62-9d26-251b17ebcb2a
        health: HEALTH_OK

      services:
        mon: 3 daemons, quorum node1,node2,node3
        mgr: node1(active), standbys: node2, node3
        osd: 3 osds: 3 up, 3 in

      data:
        pools:   2 pools, 256 pgs
        objects: 10 objects, 89 bytes
        usage:   21827 MB used, 8862 MB / 30690 MB avail
        pgs:     256 active+clean
    ```