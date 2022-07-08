---
layout: post
title:  openstack中region、az、host aggregate、cell 概念
date:   2018-06-03
categories: document
tag:
  - openstack

---
* content
{:toc}

### 前言
[参考1](https://www.cnblogs.com/xingyun/p/4703325.html)
[参考2](http://www.aboutyun.com/thread-11406-1-1.html)
### 概述

<img src="{{ '/styles/images/openstack-region-cell-az.jpg' | prepend: site.baseurl }}" alt="" width="610" />

为了提供规模化、分布式部署、资源优化利用和兼容 AWS 的功能，openstack 引入了 Region，Cell，Availability Zone(AZ) 和 Host Aggregates Zone(HAZ) 四个概念，其中 Region 和 AZ 是从公有云大哥 AWS 引入，Cell 是为了扩充一个 Region 下的集群的规模而引入的，Host Aggregates 是优化资源调度和利用引入的。这四个概念均和集群部署相关;从部署层次来说，它们有以下关系 Region > Cell > Availabiliy Zone > Host Aggregates

<img src="{{ '/styles/images/region-cell-az-haz.png' | prepend: site.baseurl }}" alt="" width="610" />

### region

顾名思义，Region 直译过来就是区域，地域的概念，而事实上，AWS 按地域(国家或者城市)设置一个 Region，每个 Region 下有多个 Availability Zone。Openstack 同样支持 Region 的概念，支持全球化部署，比如为了降低网络延时，用户可以选择特定的 Region 来部署服务。各个 Region 之间的计算资源、网络资源、存储资源都是独立的，但所有 Region 共享账户用户信息，因为 Keystone 是实现 openstack 租户用户管理和认证的功能的组件，所以 Keystone 全局唯一，所有 Region 共享一个 Keystone，Keystone endpoint 中存储了访问各个 Region 的 URL。

<img src="{{ '/styles/images/openstack-region.jpg' | prepend: site.baseurl }}" alt="" width="310" />

### cell
Cell 概念的引入，是为了扩充单个 Region 下的集群规模，主要解决 AMQP 和 Database 的性能瓶颈，每个 Region 下的 openstack 集群都有自己的消息中间件和数据库，当计算节点达到一定规模(和IBM，easystack，华为等交流的数据是300~500)，消息中间件就成为了扩展计算节点的性能瓶颈。Cell 的引入就是为了解决单个 Region 的规模问题，每个 Region 下可以有多个 Cell，每个 Cell 维护自己的数据库和消息中间件，所有 Cell 共享本 Region 下的 nova-api，共享全局唯一的 Keystone。

<img src="{{ '/styles/images/openstack-cell-02.jpg | prepend: site.baseurl }}" alt="" width="610" />

### AZ & HAZ
即Availability Zone & Host Aggregates Zone

之所以把 AZ 和 HAZ 放到一同分析，是因为二者的概念实在类似。

AWS 每个 Region 下有多个 AZ。Openstack 也引入了 AZ 的概念， AZ 的引入是基于安全的角度考虑，比如我们定义一个机房为一个 AZ，把该机房所有计算节点纳入到一个 AZ 中，其中一个机房因为某种原因down 掉，不会影响其它机房的虚拟机和网络；同时， AZ 对用户来说是一个可见的概念，用户创建虚拟机时，可以明确指出在哪个 AZ，用户可以通过在多个 AZ 创建虚拟机来保证高可靠性。</p>
在grizzly版本之后，AZ都是基于aggregate实现的。原本数据库中的表service的availability_zone字段已删除。并且配置项`node_availability_zone`也无效。新增一个配置项`default_availability_zone=nova`，表示如果host不属于任何aggregate，或者aggregate没有设置availability-zone，那么host的AZ为nova。

HAZ 也是把一批具有共同属性的计算节点划分到同一个 Zone 中，HAZ 可以对 AZ 进一步细分，一个 AZ 可以有多个 HAZ。 同一个 HAZ 下的机器都具有某种共同的属性，比如高性能计算，高性能存储(SSD)，高性能网络(支持SRIOV等)。HAZ 和 AZ 另一个不同之处在于 HAZ 对用户不是明确可见的，用户在创建虚拟机时不能像指定 AZ 一样直接指定 HAZ，但是可以通过在 Instance Flavor 中设置相关属性，由 nova-scheduler 调度根据该调度策略调度到满足该属性的的 Host Aggr egates Zones 中。</p>

{% raw %}
<img src="{{ '/styles/images/openstack-az-haz.jpg | prepend: site.baseurl }}" alt="" width="610" />
{% endraw %}

### AZ及HAZ的使用方法

#### Availability Zone 使用方法
Nova 调用创建 HAZ 的 API 创建 AZ，即在创建 HAZ 时，定义一个 AZ。
```
$nova   aggregate-create   HAZ-01   AZ-01
+——+—————+——————————+————+—————+
| Id      | Name         | Availability Zone          | Hosts      | Metadata   |
+——+—————+——————————+————+—————+
| 3       | HAZ-01      |  AZ-01                            |                |                    |
+——+—————+——————————+————+—————+
```

创建好 AZ 后，把计算节点加入到该 HAZ，因为 HAZ 属于 AZ，因此新增的计算节点也属于该 AZ
```
$nova   aggregate-add-host   3   compute01
+——+————+————————+————————+——————————————+
| Id      | Name     | Availability Zone   | Hosts                     | Metadata                                    |
| 3       | HAZ-01  | AZ-01                      | [u'compute-1']     | {u'availability_zone': u'AZ-01′}   |
+——+————+————————+————————+——————————————+
```

创建虚拟机时，指定 AZ 名字即可
```
nova boot  –flavor m1.small  –image cirros –availability-zone AZ-01 vm
```

#### Host Aggregates Zone 的使用方法
配置 nova.conf
```
scheduler_default_filters=AggregateInstanceExtraSpecsFilter,AvailabilityZoneFilter,RamFilter,ComputeFilter
```

创建一个 HAZ
```
$nova aggregate-create HAZ-SSD
+——+—————+————————+———+—————+
| Id      | Name         | Availability Zone   | Hosts  | Metadata   |
+——+—————+————————+———+—————+
| 4       | HAZ-SSD    | None                     |            |                    |
+——+—————+————————+———+—————+
```

设置 Metadata 属性
```
$nova aggregate-set-metadata 4 ssd=true
+——+—————+————————+————+—————————+
| Id      | Name         | Availability Zone   | Hosts      | Metadata                  |
+——+—————+————————+————+—————————+
| 4       | HAZ-SSD    | None                     | []             | {u'ssd': u'true'}         |
+——+—————+————————+————+—————————+
```
添加计算节点到 HAZ
```
nova aggregate-add-host 4 compute02
+——+—————+————————+———————+————————+
| Id      | Name         | Availability Zone   | Hosts                 | Metadata              |
+——+—————+————————+———————+————————+
| 5       | HAZ-SSD    | None                     | [u'compute02'] | {u'ssd': u'true'}       |
+——+—————+————————+———————+————————+
```
创建 flavor
```
$nova flavor-create m1.ssd auto 4096 10 2 –is-public true

+———+————+——————+———+—————+———+———+——————+————+——————+
| ID         | Name      | Memory_MB  | Disk    | Ephemeral  | Swap   | VCPUs | RXTX_Factor  | Is_Public  | extra_specs   |
+———+————+——————+———+—————+———+———+——————+————+——————+
| 1          | m1.ssd    | 4096       | 10       | 0         |        | 2     | 1.0          | True        | {}            |
+———+————+——————+———+—————+———+———+——————+————+——————+
```

设置 flavor 属性
```
$nova flavor-key m1.ssd set ssd=true

$nova flavor-show m1.ssd
+———+————+——————+———+—————+———+———+——————+————+———————+
| ID         | Name      | Memory_MB  | Disk    | Ephemeral  | Swap   | VCPUs | RXTX_Factor  | Is_Public  | extra_specs       |
+———+————+——————+———+—————+———+———+——————+————+———————+
| 1          | m1.ssd    | 4096       | 10      | 0          |        | 2     | 1.0          | True       | {u'ssd': u'true'}  |
+———+————+——————+———+—————+———+———+——————+————+———————+
```

采用 m1.ssd 创建虚拟机
```
nova boot –flavor m1.ssd –image  cirros vm_ssd
```

### 附
#### 另一个比较好的例子：

Example: Specify compute hosts with SSDs

```
/etc/nova/nova.conf:
scheduler_default_filters=AggregateInstanceExtraSpecsFilter,AvailabilityZoneFilter,RamFilter,ComputeFilter

$ nova aggregate-create fast-io nova
+----+---------+-------------------+-------+----------+
| Id | Name    | Availability Zone | Hosts | Metadata |
+----+---------+-------------------+-------+----------+
| 1  | fast-io | nova              |       |          |
+----+---------+-------------------+-------+----------+

$ nova aggregate-set-metadata 1 ssd=true
+----+---------+-------------------+-------+-------------------+
| Id | Name    | Availability Zone | Hosts | Metadata          |
+----+---------+-------------------+-------+-------------------+
| 1  | fast-io | nova              | []    | {u'ssd': u'true'} |
+----+---------+-------------------+-------+-------------------+

$ nova aggregate-add-host 1 node1
+----+---------+-------------------+-----------+-------------------+
| Id | Name    | Availability Zone | Hosts      | Metadata          |
+----+---------+-------------------+------------+-------------------+
| 1  | fast-io | nova              | [u'node1'] | {u'ssd': u'true'} |
+----+---------+-------------------+------------+-------------------+

$ nova aggregate-add-host 1 node2
+----+---------+-------------------+---------------------+-------------------+
| Id | Name    | Availability Zone | Hosts                | Metadata          |
+----+---------+-------------------+----------------------+-------------------+
| 1  | fast-io | nova              | [u'node1', u'node2'] | {u'ssd': u'true'} |
+----+---------+-------------------+----------------------+-------------------+
$ nova flavor-create ssd.large 6 8192 80 4
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public | extra_specs |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
| 6  | ssd.large | 8192      | 80   | 0         |      | 4     | 1           | True      | {}          |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
$nova flavor-key set_key --name=ssd.large  --key=ssd --value=true
$nova flavor-show ssd.large
+----------------------------+-------------------+
| Property                   | Value             |
+----------------------------+-------------------+
| OS-FLV-DISABLED:disabled   | False             |
| OS-FLV-EXT-DATA:ephemeral  | 0                 |
| disk                       | 80                |
| extra_specs                | {u'ssd': u'true'} |
| id                         | 6                 |
| name                       | ssd.large         |
| os-flavor-access:is_public | True              |
| ram                        | 8192              |
| rxtx_factor                | 1.0               |
| swap                       |                   |
| vcpus                      | 4                 |
+----------------------------+-------------------+
Now, when a user requests an instance with the ssd.large flavor, the scheduler only considers hosts with the ssd=true key-value pair. In this example, these are node1 and node2.
```

*说明*
此外每一个aggregate还可以配置不同的metadata(AZ也是一个metadata) 如上图中所示，aggregateA被标记了ssd：true，表示这些host支持SSD，如果在flavor的extra-specs中配置了ssd：true，那么调度时，仅有这个aggregate的宿主机会通过过滤器。换言之，此flavor创建的虚拟机只能落在此aggregate上。

#### 命令行接口
```
nova aggregate-list
# Print a list of all aggregates.

nova aggregate-create <name> <availability-zone>
# Create a new aggregate named <name> in availability zone <availability-zone>. Returns the ID of the newly created aggregate.

nova aggregate-delete <id>
# Delete an aggregate with id <id>.

nova aggregate-details <id>
# Show details of the aggregate with id <id>.

nova aggregate-add-host <id> <host>
# Add host with name <host> to aggregate with id <id>.

nova aggregate-remove-host <id> <host>
# Remove the host with name <host> from the aggregate with id <id>.

nova aggregate-set-metadata <id> <key=value> [<key=value> ...]
# Add or update metadata (key-value pairs) associated with the aggregate with id <id>.

nova aggregate-update <id> <name> [<availability_zone>]
# Update the aggregate's name and optionally availability zone.

nova host-list
# List all hosts by service.

nova host-update --maintenance [enable | disable]
#Put/resume host into/from maintenance.
```
[参考](http://docs.openstack.org/trunk/openstack-compute/admin/content/host-aggregates.html)

#### 关于flavor
类型模板(flavor) 在Openstack中，虚机硬件模板被称为类型模板(flavor)，包括RAM和硬盘大小，CPU核数等。标准安装后有5个缺省的类型
