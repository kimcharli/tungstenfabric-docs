Miscellaneous topics for various deployment of tungsten fabric, which is not included in primer document.

## charm install

Tungsten Fabric can also be installed by juju charm.
- bionic and openstack queens is used, 4 nodes (juju node, openstack controller, openstack compute, tunsten-fabric controller)

```
# apt update
# snap install --classic juju
# juju add-cloud

Select cloud type: manual
Enter a name for your manual cloud: manual-cloud-1
Enter the controller's hostname or IP address: (juju node's ip)

# ssh-keygen
# cd .ssh
# cat id_rsa.pub >> authorized_keys
# cd
# ssh-copy-id (other nodes' ip)

# juju bootstrap manual-cloud-1

# git clone https://github.com/Juniper/contrail-charms -b R5

# juju add-machine ssh:root@(openstack-controller ip)
# juju add-machine ssh:root@(openstack-compute ip)
# juju add-machine ssh:root@(TungstenFabric-controller ip)

# vi set-juju.sh
juju deploy ntp
juju deploy rabbitmq-server --to lxd:0
juju deploy percona-cluster mysql --config root-password=contrail123 --config max-connections=1500 --to lxd:0
juju deploy openstack-dashboard --to lxd:0
juju deploy nova-cloud-controller --config console-access-protocol=novnc --config network-manager=Neutron --to lxd:0
juju deploy neutron-api --config manage-neutron-plugin-legacy-mode=false --config neutron-security-groups=true --to lxd:0
juju deploy glance --to lxd:0
juju deploy keystone --config admin-password=contrail123 --config admin-role=admin --to lxd:0

juju deploy nova-compute --config ./nova-compute-config.yaml --to 1

CHARMS_DIRECTORY=/root
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-keystone-auth --to 2
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-controller --config auth-mode=rbac --config cassandra-minimum-diskgb=4 --config cassandra-jvm-extra-opts="-Xms1g -Xmx2g" --to 2
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-analyticsdb --config cassandra-minimum-diskgb=4 --config cassandra-jvm-extra-opts="-Xms1g -Xmx2g" --to 2
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-analytics --to 2
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-openstack
juju deploy $CHARMS_DIRECTORY/contrail-charms/contrail-agent

juju expose openstack-dashboard
juju expose nova-cloud-controller
juju expose neutron-api
juju expose glance
juju expose keystone

juju expose contrail-controller
juju expose contrail-analytics

juju add-relation keystone:shared-db mysql:shared-db
juju add-relation glance:shared-db mysql:shared-db
juju add-relation keystone:identity-service glance:identity-service
juju add-relation nova-cloud-controller:image-service glance:image-service
juju add-relation nova-cloud-controller:identity-service keystone:identity-service
juju add-relation nova-cloud-controller:cloud-compute nova-compute:cloud-compute
juju add-relation nova-compute:image-service glance:image-service
juju add-relation nova-compute:amqp rabbitmq-server:amqp
juju add-relation nova-cloud-controller:shared-db mysql:shared-db
juju add-relation nova-cloud-controller:amqp rabbitmq-server:amqp
juju add-relation openstack-dashboard:identity-service keystone

juju add-relation neutron-api:shared-db mysql:shared-db
juju add-relation neutron-api:neutron-api nova-cloud-controller:neutron-api
juju add-relation neutron-api:identity-service keystone:identity-service
juju add-relation neutron-api:amqp rabbitmq-server:amqp

juju add-relation contrail-controller ntp
juju add-relation nova-compute:juju-info ntp:juju-info

juju add-relation contrail-controller contrail-keystone-auth
juju add-relation contrail-keystone-auth keystone
juju add-relation contrail-controller contrail-analytics
juju add-relation contrail-controller contrail-analyticsdb
juju add-relation contrail-analytics contrail-analyticsdb

juju add-relation contrail-openstack neutron-api
juju add-relation contrail-openstack nova-compute
juju add-relation contrail-openstack contrail-controller

juju add-relation contrail-agent:juju-info nova-compute:juju-info
juju add-relation contrail-agent contrail-controller

# vi nova-compute-config.yaml 
nova-compute:
    virt-type: qemu 
    enable-resize: True
    enable-live-migration: True
    migration-auth-type: ssh

# bash set-juju.sh

(to check status, it takes 20 minutes for every application to be active)
# juju status
# tail -f /var/log/juju/*log | grep -v -w DEBUG
```

To make this work, two points are to be cared of.
1. Since juju internally use LXD and its own subnets, at least tungsten fabric nodes need to have some static route to that subnet (if AWS is used, VPC's route table can be used, Source/Dest Check also need to be disabled)
2. Since LXD by default won't allow docker to run, it needs to be allowed by lxc config.

```
juju ssh 0
  sudo su -
    lxc list
    lxc config set juju-cb8047-0-lxd-4 security.nesting true
    lxc config show juju-cb8047-0-lxd-4
    lxc restart juju-cb8047-0-lxd-4
```

### How to build tungsten fabric

This repo's readme mostly worked well.
https://github.com/Juniper/contrail-dev-env

```
yum -y install docker git
git clone https://github.com/Juniper/contrail-dev-env
cd contrail-dev-env
./startup.sh
docker exec -it contrail-developer-sandbox bash

cd /root/contrail-dev-env
yum -y remove python-devel ## it is needed to resolve dependency issue
make sync
make fetch_packages
make setup
make dep
```

To build all the modules, this command can be used (it takes 1-2 hours, depending on machine power)
```
make rpm
make containers
```

To build more specific modules, those command also can be used.
One notice is that rpm-contrail is a big package itself, and can't decomposed more .. (controller, vrouter etc is included)
```
make list
make rpm-contrail

make list-containers
make container-general-base
make container-base
make container-kubernetes_kube-manager

- those make targets are included from this file:
 /root/contrail/tools/packages/Makefile
 https://github.com/Juniper/contrail-packages/blob/master/Makefile
```

To build only vrouter.ko, this command worked for me.
```
build:
cd /root/contrail
scons --opt=production --kernel-dir=/lib/modules/3.10.0-1062.el7.x86_64/build build-kmodule

clean:
cd /root/contrail/vrouter
make KERNELDIR=/lib/modules/3.10.0-1062.el7.x86_64/build clean
```

### Nested kubernetes installation on openstack
### Tungsten fabric deployment on public cloud
### erm-vpn

### Random tungsten fabric patch (not tested)
#### static schedule for svc-monitor logic to choose available vRouters
```
diff --git a/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py b
index f40de26..d5c2478 100644
--- a/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py
+++ b/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py
@@ -200,3 +200,8 @@ class RandomScheduler(VRouterScheduler):
         self._vnc_lib.ref_update('virtual-router', chosen_vrouter,
             'virtual-machine', vm.uuid, None, 'ADD')
         return chosen_vrouter
+
+class StaticScheduler(VRouterScheduler):
+    """Statically assign vRouter nodes for v1 service-chain, haproxy lb, SNAT e
+    def schedule(self, si, vm):
+        return ['bms11', 'bms12']
```

#### decouple analytics from svc-monitor logic
```
diff --git a/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py b/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.
index f40de26..7fd1f0a 100644
--- a/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py
+++ b/src/config/svc-monitor/svc_monitor/scheduler/vrouter_scheduler.py
@@ -115,6 +115,8 @@ class VRouterScheduler(object):
         return response_dict
 
     def vrouters_running(self):
+        ## implement logic to see available vRouter, without checking analytics response (possible choice is xmpp status from control node)
+
         # get az host list
         az_vrs = self._get_az_vrouter_list()
```


#### more scalable haproxy loadbalancer and SNAT
```
diff --git a/src/config/svc-monitor/svc_monitor/services/loadbalancer/drivers/ha_proxy/driver.py b/src/config/svc-monitor/svc_monitor/services/loadbalancer/drivers/ha_proxy/driver.py
index 5487b2b..1bee992 100644
--- a/src/config/svc-monitor/svc_monitor/services/loadbalancer/drivers/ha_proxy/driver.py
+++ b/src/config/svc-monitor/svc_monitor/services/loadbalancer/drivers/ha_proxy/driver.py
@@ -92,8 +92,8 @@ class OpencontrailLoadbalancerDriver(
 
         # set interfaces and ha
         props.set_interface_list(if_list)
-        props.set_ha_mode('active-standby')
-        scale_out = ServiceScaleOutType(max_instances=2, auto_scale=False)
+        props.set_ha_mode('active-active')
+        scale_out = ServiceScaleOutType(max_instances=10, auto_scale=False)
         props.set_scale_out(scale_out)
 
         return props
diff --git a/src/config/svc-monitor/svc_monitor/snat_agent.py b/src/config/svc-monitor/svc_monitor/snat_agent.py
index 54ea709..f5bce37 100644
--- a/src/config/svc-monitor/svc_monitor/snat_agent.py
+++ b/src/config/svc-monitor/svc_monitor/snat_agent.py
@@ -169,7 +169,7 @@ class SNATAgent(Agent):
             si_obj.fq_name = project_fq_name + [si_name]
             si_created = True
         si_prop_obj = ServiceInstanceType(
-            scale_out=ServiceScaleOutType(max_instances=2,
+            scale_out=ServiceScaleOutType(max_instances=10,
                                           auto_scale=True),
             auto_policy=False)
 
@@ -181,7 +181,7 @@ class SNATAgent(Agent):
         right_if = ServiceInstanceInterfaceType(
             virtual_network=':'.join(vn_obj.fq_name))
         si_prop_obj.set_interface_list([right_if, left_if])
-        si_prop_obj.set_ha_mode('active-standby')
+        si_prop_obj.set_ha_mode('active-active')
 
         si_obj.set_service_instance_properties(si_prop_obj)
         si_obj.set_service_template(st_obj)
```

#### three xmpp connections (to cover double failure scenario)
```
diff --git a/src/vnsw/agent/cmn/agent.h b/src/vnsw/agent/cmn/agent.h
index 3e48812..832b476 100644
--- a/src/vnsw/agent/cmn/agent.h
+++ b/src/vnsw/agent/cmn/agent.h
@@ -284,7 +284,10 @@ extern void RouterIdDepInit(Agent *agent);
 #define MULTICAST_LABEL_BLOCK_SIZE 2048
 
 #define MIN_UNICAST_LABEL_RANGE 4098
-#define MAX_XMPP_SERVERS 2
+
+/* to cover double failure case */
+#define MAX_XMPP_SERVERS 3 
+
 #define XMPP_SERVER_PORT 5269
 #define XMPP_DNS_SERVER_PORT 53
 #define METADATA_IP_ADDR ntohl(inet_addr("169.254.169.254"))
```

#### vlan-base interop
```
diff --git a/src/bgp/evpn/evpn_route.cc b/src/bgp/evpn/evpn_route.cc
index 36412b2..a830b5c 100644
--- a/src/bgp/evpn/evpn_route.cc
+++ b/src/bgp/evpn/evpn_route.cc
@@ -487,7 +487,7 @@ void EvpnPrefix::BuildProtoPrefix(BgpProtoPrefix *proto_prefix,
                 proto_prefix->prefix.begin() + esi_offset);
         }
         size_t tag_offset = esi_offset + kEsiSize;
-        put_value(&proto_prefix->prefix[tag_offset], kTagSize, tag_);
+        put_value(&proto_prefix->prefix[tag_offset], kTagSize, 0);
         size_t mac_len_offset = tag_offset + kTagSize;
         proto_prefix->prefix[mac_len_offset] = 48;
         size_t mac_offset = mac_len_offset + 1;
```

#### 'enable_nova: no' can be configurable

```
git clone -b contrail/queens https://github.com/Juniper/contrail-kolla-ansible

diff --git a/ansible/post-deploy-contrail.yml b/ansible/post-deploy-contrail.yml
index e603207..c700d88 100644
--- a/ansible/post-deploy-contrail.yml
+++ b/ansible/post-deploy-contrail.yml
@@ -63,6 +63,8 @@
       - ['baremetal-hosts', 'virtual-hosts']
     register: command_result
     failed_when: "command_result.rc == 1 and 'already exists' not in command_result.stderr"
+    when:
+      - enable_nova | bool
     run_once: yes
 
   - name: Add compute hosts to virtual-hosts Aggregate Group
```

#### security endpoint stats per tag as UVE
https://review.opencontrail.org/c/Juniper/contrail-specs/+/55761

#### kubernetes multi master setup
1. https://review.opencontrail.org/c/Juniper/contrail-controller/+/55758
1. https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#k8sk8s

#### tc-flower offload

```
For someone interested,

I tried two vRouter setup and type those commands on one node to bypass vRouter datapath in favor of tc,
and found tc-flower based vxlan datapath (egress) and vRouter's vxlan datapath can communicate :)
 - ingress vxlan decap won't work well, I'm still investigating ..

vRouter0: 172.31.4.175 (container, 10.0.1.251)
vRouter1: 172.31.1.214 (container, 10.0.1.250, connected to tapeth0-038fdd)

[from specific tap to known ip address, vxlan encap could be offloaded to tc]
 - typed on vRouter1
ip link set vxlan7 up
ip link add vxlan7 type vxlan vni 7 dev ens5 dstport 0 external
tc filter add dev tapeth0-038fdd protocol ip parent ffff: \
                flower \
                  ip_proto icmp dst_ip 10.0.1.251 \
                action simple sdata "ttt" action tunnel_key set \
                  src_ip 172.31.1.214 \
                  dst_ip 172.31.4.175 \
                  id 7 \
                  dst_port 4789 \
                action mirred egress redirect dev vxlan7

[although for egress traffic vRouter1 is bypassed, it can still communicate]

[root@ip-172-31-1-214 ~]# tcpdump -nn -i ens5 udp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens5, link-type EN10MB (Ethernet), capture size 262144 bytes
04:55:41.566458 IP 172.31.1.214.57877 > 172.31.4.175.4789: VXLAN, flags [I] (0x08), vni 7
IP 10.0.1.250 > 10.0.1.251: ICMP echo request, id 60416, seq 180, length 64
04:55:41.566620 IP 172.31.4.175.61117 > 172.31.1.214.4789: VXLAN, flags [I] (0x08), vni 7
IP 10.0.1.251 > 10.0.1.250: ICMP echo reply, id 60416, seq 180, length 64
04:55:42.570917 IP 172.31.1.214.57877 > 172.31.4.175.4789: VXLAN, flags [I] (0x08), vni 7
IP 10.0.1.250 > 10.0.1.251: ICMP echo request, id 60416, seq 181, length 64
04:55:42.571056 IP 172.31.4.175.61117 > 172.31.1.214.4789: VXLAN, flags [I] (0x08), vni 7
IP 10.0.1.251 > 10.0.1.250: ICMP echo reply, id 60416, seq 181, length 64
^C
4 packets captured
5 packets received by filter
0 packets dropped by kernel
[root@ip-172-31-1-214 ~]#

/ # ping 10.0.1.251
PING 10.0.1.251 (10.0.1.251): 56 data bytes
64 bytes from 10.0.1.251: seq=0 ttl=64 time=5.183 ms
64 bytes from 10.0.1.251: seq=1 ttl=64 time=4.587 ms
^C
--- 10.0.1.251 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 4.587/4.885/5.183 ms
/ # 

[tap's RX is not incrementing since that is bypassed (TX increments, since ingress traffic still uses vRouter datapath)]

[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3393  bytes:288094 errors:0
            TX packets:3438  bytes:291340 errors:0
[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3393  bytes:288094 errors:0
            TX packets:3439  bytes:291438 errors:0
[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3394  bytes:288136 errors:0
            TX packets:3442  bytes:291676 errors:0
[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3394  bytes:288136 errors:0
            TX packets:3444  bytes:291872 errors:0
[root@ip-172-31-1-214 ~]# vif --get 8 | grep bytes
            RX packets:3394  bytes:288136 errors:0
            TX packets:3447  bytes:292166 errors:0
[root@ip-172-31-1-214 ~]#
```
