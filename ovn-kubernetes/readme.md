# ovn-kubernetes

![image](https://user-images.githubusercontent.com/10542832/85043578-1789d400-b1bf-11ea-9565-3ad9dab19b5b.png)

# Prerequsite

## SDN, OVS, and OVN
- SDN, despite some using “SDN” and “OpenFlow” interchangeably

    - Software-defined networking (SDN) is a way to manage networks that separates the **control plane** from the forwarding plane. It does so by using **software to manage network functions** through a **centralized control point**. SDN is a complementary approach to network functions virtualization (NFV) for network management

    - SDN offers a centralized view of the network
    - Controller
        - act as the **brains** of the network. 
        - relays information to switches and routers
        - **southbound APIs**, to switches and routers
            - **OpenFlow**; however, it isn’t the only SDN standard 
        - **northbound APIs**, to the applications
            
    - ![image](https://user-images.githubusercontent.com/10542832/85045957-4d7c8780-b1c2-11ea-9cf4-6fce85e094a1.png)

- OVS
    - Open vSwitch is a production quality, multilayer virtual switch
    - support distribution across multiple physical servers 

- Open Virtual Networking, OVN
    - OVN’s main goal is to provide **Layers 2 and 3** networking, as well as security groups
        - https://github.com/ovn-org/ovn-kubernetes/blob/master/docs/ovs_offload.md
        - SDN and its controllers are, however,for general-purpose.
    - OVN 
        - Overlay* : OVN can create a logical network amongst containers running on multiple hosts **without pre-created networking.** 
        - Underlay : OVN requires a OpenStack setup to provide container networking

        For both the modes to work, a user has to install and start **Open vSwitch** in each VM/host that they plan to run their containers on.
    - Open vSwitch (OVS)
        - Provides L2/L3 virtual networking
            - Logical switches and routers
            - Security groups
            - L2/L3/L4 ACLs
            - Multiple tunnel overlays (**Geneve**, STT, and VXLAN)
            - TOR-based and software-based logical-physical gateways
        - Work on same platforms as OVS Linux (KVM and Xen)
            - Containers
            - DPDK
            - Hyper-V
                - Integration with OpenStack and other CMSs 

       
## Intallation
- You can **only change the configuration** for your Pod network provider **during cluster installation**.

- [IPI configuration](https://docs.openshift.com/container-platform/4.3/networking/cluster-network-operator.html#nw-operator-configuration-parameters-for-ovn-sdn_cluster-network-operator)
    ```
    -- output omitted -- 
    defaultNetwork:
      type: OVNKubernetes 
      ovnKubernetesConfig: 
        mtu: 1450 
        genevePort: 6081 
    -- output omitted -- 
    ```

- GENEVE
    - Tunnel (e.g. STT, GRE, VXLAN)
    - The MTU : 1450 for the `Generic Network Virtualization Encapsulation`,GENEVE, overlay network
    - **UDP port** for the GENEVE overlay network
      - you will see the GENEVE port on OVS switch


## ovn-kubernetes

- substitute iptables with OVN Logical Network
    - for example, when communicating with Pod from outside the cluster using NodePort or ExternalIP, DNAT is performed with iptables on the host and the packet is pulled into OVN Logical Network.

- Raft-based clustering, Master-Slave type replication functions are also available as OVSDB (NorthDB and SouthDB) redundancy methods.


### Components

- **kubeovn** : get pods, svc informatino from kubernetes
- **ovnkube-node** : node-level bookkeeping and configuration
- **ovnkube-master** : convert kubernetes objects in to nbdb logical network components
- **ovs-node** : ovsdb and ovs-vswitchd

![image](https://cdn-ak.f.st-hatena.com/images/fotolife/o/orimanabu/20191220/20191220030340.png)

### Defines
- **flow**
    - the most important kind of flow
    - define switch's policy.
    - support wildcard, priorities, and multi-tables

- **dataplan** 
    - Open vSwitch software switch implementation uses a second kind of flow **internally**, or kernel flow
    - do not support priorities and comprise only a single table
    - datapath flows do support wildcarding, in Open vSwitch 1.11 and later.

### Control

![](https://sites.google.com/a/cnsrl.cycu.edu.tw/da-shu-bi-ji/_/rsrc/1512474568148/openvswitch/OVS_struct.jpg)

`ovs-Xctl`

- **ovs-vsctl** : control ovs-vswitchd (ovsdb)
- **ovs-ofctl** : flow and flowtable on OpenFlow switch
- **ovs-dpctl** : ovs datapath (kernel flow)
- **ovs-appctl** : query and govern ovs daemon


## CNI
### Add
- ovn get ip/mac/gateway from annotation [](https://github.com/ovn-org/ovn-kubernetes/blob/d118b7ec31a0500ff19d95a35838bf94d84c2f94/go-controller/pkg/ovn/pods_test.go#L235)
- In container netns, OVN configures interface and port
    ```
    ovs-vsctl add-port br-int veth_outside \
      --set interface veth_outside \
        external_ids:attached_mac=mac_address \
        external_ids:iface-id=namespace_pod \
        external_ids:ip_address=ip_address
    ```

### Del
```
ovs-vsctl del-port br-int port
```

## OVS Control

### Bridge
```
ovs-vsctl show  
ovs-vsctl [--may-exist] add-br br0
ovs-vsctl [--if-exists] del-br br0  
ovs-vsctl list-br  
```

### Port
```
ovs-vsctl [--may-exist] add-port br0 eth1
ovs-vsctl add-bond br0 bond0 eth0 eth1 eth2  
ovs-vsctl [--if-exists] del-port br0 eth1  
ovs-vsctl list-ports br0 
ovs-vsctl list interface eth1
```
### SDN Controller
```
ovs-vsctl set-controller br0 tcp:1.2.3.4:6633
ovs-vsctl set-controller br0 tcp:1.2.3.4:6633 tcp:4.3.2.1:6633
ovs-vsctl set-controller br0 unix:/var/run/xx/xx.sock 
ovs-vsctl del-controller br0  
ovs-vsctl get-controller br0 
```
### VLAN tags
```
ovs-vsctl set port eth0 tag=10  
ovs-svctl add-port br0 eth1 tag=10  
ovs-vsctl add-port br0 eth1 trunk=9,10,11  
```

```
# key=100 stands for vni 100，default is 0
ovs-vsctl add-port ovs0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.10.10.1 options:key=100

# default
ovs-vsctl add-port ovs0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.10.10.1

# key=flow means the vni of the port can be configured by flow for instance 
# actions=set_field:100->tun_id
# actions=set_tunnel:100
ovs-vsctl add-port ovs0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.10.10.1 options:key=flow  
```

### app control
```
ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound
```

## Apps
- [Router](https://github.com/openshift/ovn-kubernetes/blob/005ccbb5723de22767ca4f2317f1f5779b0a5a0e/go-controller/pkg/ovn/gateway_init.go#L15)

    https://github.com/openshift/ovn-kubernetes/blob/005ccbb5723de22767ca4f2317f1f5779b0a5a0e/go-controller/pkg/ovn/master_test.go#L72

    - functions actually work based on flow
      https://github.com/openshift/ovn-kubernetes/blob/005ccbb5723de22767ca4f2317f1f5779b0a5a0e/go-controller/pkg/node/gateway_init_linux_test.go#L82

## Flows
![image](https://user-images.githubusercontent.com/10542832/83830863-ba027b80-a718-11ea-9d5a-80e9a6ae79bd.png)
![image](https://camo.githubusercontent.com/8751ce69e94ca6310a2179bf9b62190abb998933/68747470733a2f2f692e696d6775722e636f6d2f6937736369394f2e706e67)


## Related Works

you can refer to materials from:
- https://github.com/openshift/ovn-kubernetes
    - there're plenty of docs there : https://github.com/openshift/ovn-kubernetes/tree/master/docs

- https://github.com/openshift/cluster-network-operator

- https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/networking_with_open_virtual_network/open_virtual_network_ovn

- (Japanes) https://rheb.hatenablog.com/entry/openshift42ovn#fnref:5

- https://issues.redhat.com/browse/SDN-821

- http://docs.openvswitch.org/en/latest/topics/datapath/

- http://docs.openvswitch.org/en/latest/faq/design/

- https://www.sdxcentral.com/networking/sdn/

- (Chinese) https://feisky.gitbooks.io/sdn/ovs/ovn.html

- (Chinese) https://www.jianshu.com/p/d72207486e29