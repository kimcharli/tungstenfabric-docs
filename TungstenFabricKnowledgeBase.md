Miscellaneous topics for various deployment of tungsten fabric, which is not included in primer document.


## some configuration knobs which are not documented well

### forwarding mode

vRouter has several forwarding mode.
 - https://bugs.launchpad.net/juniperopenstack/+bug/1471637

By default, it will use l2 / l3 mode. L3 mode and L2 mode also have some usecase.


### flood unknown unicast

This knob is used when l2 BMS connectivity is used.
- https://bugs.launchpad.net/juniperopenstack/+bug/1424523

By default, vRouter will drop unknown unicast, since controller knows all mac address of VMs, although it is not the case when l2 BMS is used.

This knob need to be enabled in such case.


### allow tranisit

This knob is used with service-chain feature.
 - https://bugs.launchpad.net/opencontrail/+bug/1365277

When VM1 - VN1 - VNF1 - VN2 - VNF2 - VN3 - VM2 is created and VN1-VN2, VN2-VN3 service-chain is configured, by default, VM1 can't ping VM2, since ServiceChain prefix is not transitive.

When this knob is enabled in VN2, prefix in VN1 will be imported to VN3 and vice versa, so VM1 can ping to VM3.

### multiple service chain

I actually never tried this knob.

Detail is described in this url.
 - https://bugs.launchpad.net/juniperopenstack/+bug/1505023
 
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

### multi kube-master deployment

3 tungsten fabric controller nodes: m3.xlarge (4 vcpu) -> c3.4xlarge (16 vcpu) # since schema-transformer needed cpu resource for acl calculation, I need to add resource  
100 kube-master, 800 workers: m3.medium

tf-controller installation and first-containers.yaml is the same with this url
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#2-tungstenfabric-up-and-running

Ami is also the same (ami-3185744e), but kernel version is updated by yum -y update kernel (converted to image, and it is used to start instances)

/tmp/aaa.pem is the keypair, specified in ec2 instances

cni.yaml is attached:
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/multi-kube-master-deployment-cni-tungsten-fabric.yaml

```
(commands are typed on one of tungsten fabric controller nodes)
yum -y install epel-release
yum -y install parallel

aws ec2 describe-instances --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text | tr '\t' '\n' > /tmp/all.txt
head -n 100 /tmp/all.txt > masters.txt
tail -n 800 /tmp/all.txt > workers.txt

ulimit -n 4096
cat all.txt | parallel -j1000 ssh -i /tmp/aaa.pem -o StrictHostKeyChecking=no centos@{} id
cat all.txt | parallel -j1000 ssh -i /tmp/aaa.pem centos@{} sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
 -
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' ssh -i /tmp/aaa.pem centos@{2} sudo kubeadm init --token aaaaaa.aaaabbbbccccdddd --ignore-preflight-errors=NumCPU --pod-network-cidr=10.32.{1}.0/24 --service-cidr=10.96.{1}.0/24 --service-dns-domain=cluster{1}.local
-
vi assign-kube-master.py
computenodes=8
with open ('masters.txt') as aaa:
 with open ('workers.txt') as bbb:
  for masternode in aaa.read().rstrip().split('\n'):
   for i in range (computenodes):
    tmp=bbb.readline().rstrip()
    print ("{}\t{}".format(masternode, tmp))
 -
cat masters.txt | parallel -j1000 ssh -i /tmp/aaa.pem centos@{} sudo cp /etc/kubernetes/admin.conf /tmp/admin.conf
cat masters.txt | parallel -j1000 ssh -i /tmp/aaa.pem centos@{} sudo chmod 644 /tmp/admin.conf
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' scp -i /tmp/aaa.pem centos@{2}:/tmp/admin.conf kubeconfig-{1}
-
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' kubectl --kubeconfig=kubeconfig-{1} get node
-
cat -n join.txt | parallel -j1000 -a - --colsep '\t' ssh -i /tmp/aaa.pem centos@{3} sudo kubeadm join {2}:6443 --token aaaaaa.aaaabbbbccccdddd --discovery-token-unsafe-skip-ca-verification
-
(modify controller-ip in cni-tungsten-fabric.yaml)
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' cp cni-tungsten-fabric.yaml cni-{1}.yaml
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' sed -i -e "s/k8s2/k8s{1}/" -e "s/10.32.2/10.32.{1}/" -e "s/10.64.2/10.64.{1}/" -e "s/10.96.2/10.96.{1}/"  -e "s/172.31.x.x/{2}/" cni-{1}.yaml
-
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' kubectl --kubeconfig=kubeconfig-{1} apply -f cni-{1}.yaml
-
sed -i 's!kubectl!kubectl --kubeconfig=/etc/kubernetes/admin.conf!' set-label.sh 
cat masters.txt | parallel -j1000 scp -i /tmp/aaa.pem set-label.sh centos@{}:/tmp
cat masters.txt | parallel -j1000 ssh -i /tmp/aaa.pem centos@{} sudo bash /tmp/set-label.sh
-
cat -n masters.txt | parallel -j1000 -a - --colsep '\t' kubectl --kubeconfig=kubeconfig-{1} create -f first-containers.yaml
```

### Nested kubernetes installation on openstack

Nested kubernetes installation can be tried on all-in-one openstack node.

After that node is intalled by ansible-deployer,
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#openstack-1

one subnet for virtual-network will be created
 - for example, underlay1, 10.0.1.0/24, SNAT is checked

Additionally, link local service for vRouter TCP/9091 need to be manually created
 - https://github.com/Juniper/contrail-kubernetes-docs/blob/master/install/kubernetes/nested-kubernetes.md#option-1-fabric-snat--link-local-preferred

This configuration will create DNAT/SNAT, such as from src: 10.0.1.3:xxxx, dst-ip: 10.1.1.11:9091 to src: compute's vhost0 ip:xxxx dst-ip: 127.0.0.1:9091, so CNI inside openstack VM can directly communicate to vrouter-agent on the compute node, and pick port / ip info for containers.
 - ip address can be either one from the subnet or outside of the subnet.

On that node, two Centos7 (or ubuntu bionic) nodes will be created, and kubernetes cluster will be installed with the same procedure with this url,
 - https://github.com/tnaganawa/tungstenfabric-docs/blob/master/TungstenFabricPrimer.md#kubeadm

although yaml file needs to the one for nested install.
```
./resolve-manifest.sh contrail-nested-kubernetes.yaml > cni-tungsten-fabric.yaml

KUBEMANAGER_NESTED_MODE: "{{ KUBEMANAGER_NESTED_MODE }}" ## this needs to be "1"
KUBERNESTES_NESTED_VROUTER_VIP: {{ KUBERNESTES_NESTED_VROUTER_VIP }} ## this parameter needs to be the same IP with the one defined in link-local service (such as 10.1.1.11)
```

If coredns received ip, nested installation is working fine.

### Tungsten fabric deployment on public cloud
### erm-vpn
When erm-vpn is enabled, vrouter send multicast traffic to up to 4 nodes, to avoid ingress replication to all the nodes.
Control implements a tree to send multicast packets to all nodes.
 - https://tools.ietf.org/html/draft-marques-l3vpn-mcast-edge-00
 - https://review.opencontrail.org/c/Juniper/contrail-controller/+/256

### vRouter ml2 plugin 

I tried ml2 feature of vRouter neutron plugin.
 - https://opendev.org/x/networking-opencontrail/
 - https://www.youtube.com/watch?v=4MkkMRR9U2s

Three CentOS7.5 (4 cpu, 16 GB mem, 30 GB disk, ami: ami-3185744e) on aws are used.

Procedure based on this document is attached.
 - https://opendev.org/x/networking-opencontrail/src/branch/master/doc/source/installation/playbooks.rst

```
openstack-controller: 172.31.15.248
tungsten-fabric-controller (vRouter): 172.31.10.212
nova-compute (ovs): 172.31.0.231

(commands are on tungsten-fabric-controller, centos user is used (not root))

sudo yum -y remove PyYAML python-requests
sudo yum -y install git patch
sudo easy_install pip
sudo pip install PyYAML requests ansible==2.8.8
ssh-keygen
 add id_rsa.pub to authorized_keys on all three nodes (centos user (not root))

git clone https://opendev.org/x/networking-opencontrail.git
cd networking-opencontrail
patch -p1 < ml2-vrouter.diff 

cd playbooks
cp -i hosts.example hosts
cp -i group_vars/all.yml.example group_vars/all.yml

(ssh to all the nodes once, to update known_hosts)

ansible-playbook main.yml -i hosts

 - devstack's logs are in /opt/stack/logs/stack.sh.log
 - openstack processes' log are written in /var/log/messages
 - 'systemctl list-unit-files | grep devstack' show systemctl entry of openstack processes

(openstack controller node)
 if devstack failed with mariadb login error, please type this to fix. (last two lines need to be modified for openstack controller's ip and fqdn)
 commands will be typed by 'centos' user (not root).
   mysqladmin -u root password admin
   mysql -uroot -padmin -h127.0.0.1 -e 'GRANT ALL PRIVILEGES ON *.* TO '\''root'\''@'\''%'\'' identified by '\''admin'\'';'
   mysql -uroot -padmin -h127.0.0.1 -e 'GRANT ALL PRIVILEGES ON *.* TO '\''root'\''@'\''172.31.15.248'\'' identified by '\''admin'\'';'
   mysql -uroot -padmin -h127.0.0.1 -e 'GRANT ALL PRIVILEGES ON *.* TO '\''root'\''@'\''ip-172-31-15-248.ap-northeast-1.compute.internal'\'' identified by '\''admin'\'';'
```

hosts, group_vars/all, and patch is attached. (some are bugfix only, but some change the default behavior)
```
[centos@ip-172-31-10-212 playbooks]$ cat hosts
controller ansible_host=172.31.15.248 ansible_user=centos

# This host should be one from the compute host group.
# Playbooks are not prepared to deploy tungsten fabric compute node separately.
contrail_controller ansible_host=172.31.10.212 ansible_user=centos local_ip=172.31.10.212

[contrail]
contrail_controller

[openvswitch]
other_compute ansible_host=172.31.0.231 local_ip=172.31.0.231 ansible_user=centos

[compute:children]
contrail
openvswitch
[centos@ip-172-31-10-212 playbooks]$ cat group_vars/all.yml
---
# IP address for OpenConrail (e.g. 192.168.0.2)
contrail_ip: 172.31.10.212

# Gateway address for OpenConrail (e.g. 192.168.0.1)
contrail_gateway:

# Interface name for OpenConrail (e.g. eth0)
contrail_interface:

# IP address for OpenStack VM (e.g. 192.168.0.3)
openstack_ip: 172.31.15.248

# OpenStack branch used on VMs.
openstack_branch: stable/queens

# Optionally use different plugin version (defaults to value of OpenStack branch)
networking_plugin_version: master

# Tungsten Fabric docker image tag for contrail-ansible-deployer
contrail_version: master-latest

# If true, then install networking_bgpvpn plugin with contrail driver
install_networking_bgpvpn_plugin: false

# If true, integration with Device Manager will be enabled and vRouter
# encapsulation priorities will be set to 'VXLAN,MPLSoUDP,MPLSoGRE'.
dm_integration_enabled: false

# Optional path to file with topology for DM integration. When set and
# DM integration enabled, topology.yaml file will be copied to this location
dm_topology_file:

# If true, password to the created instances for current ansible user
# will be set to value of instance_password
change_password: false
# instance_password: uberpass1

# If set, override docker deamon /etc config file with this data
# docker_config:
[centos@ip-172-31-10-212 playbooks]$ 



[centos@ip-172-31-10-212 networking-opencontrail]$ cat ml2-vrouter.diff 
diff --git a/playbooks/roles/contrail_node/tasks/main.yml b/playbooks/roles/contrail_node/tasks/main.yml
index ee29b05..272ee47 100644
--- a/playbooks/roles/contrail_node/tasks/main.yml
+++ b/playbooks/roles/contrail_node/tasks/main.yml
@@ -7,7 +7,6 @@
       - epel-release
       - gcc
       - git
-      - ansible-2.4.*
       - yum-utils
       - libffi-devel
     state: present
@@ -61,20 +60,20 @@
     chdir: ~/contrail-ansible-deployer/
     executable: /bin/bash
 
-- name: Generate ssh key for provisioning other nodes
-  openssh_keypair:
-    path: ~/.ssh/id_rsa
-    state: present
-  register: contrail_deployer_ssh_key
-
-- name: Propagate generated key
-  authorized_key:
-    user: "{{ ansible_user }}"
-    state: present
-    key: "{{ contrail_deployer_ssh_key.public_key }}"
-  delegate_to: "{{ item }}"
-  with_items: "{{ groups.contrail }}"
-  when: contrail_deployer_ssh_key.public_key
+#- name: Generate ssh key for provisioning other nodes
+#  openssh_keypair:
+#    path: ~/.ssh/id_rsa
+#    state: present
+#  register: contrail_deployer_ssh_key
+#
+#- name: Propagate generated key
+#  authorized_key:
+#    user: "{{ ansible_user }}"
+#    state: present
+#    key: "{{ contrail_deployer_ssh_key.public_key }}"
+#  delegate_to: "{{ item }}"
+#  with_items: "{{ groups.contrail }}"
+#  when: contrail_deployer_ssh_key.public_key
 
 - name: Provision Node before deploy contrail
   shell: |
@@ -105,4 +104,4 @@
     sleep: 5
     host: "{{ contrail_ip }}"
     port: 8082
-    timeout: 300
\ No newline at end of file
+    timeout: 300
diff --git a/playbooks/roles/contrail_node/templates/instances.yaml.j2 b/playbooks/roles/contrail_node/templates/instances.yaml.j2
index e3617fd..81ea101 100644
--- a/playbooks/roles/contrail_node/templates/instances.yaml.j2
+++ b/playbooks/roles/contrail_node/templates/instances.yaml.j2
@@ -14,6 +14,7 @@ instances:
       config_database:
       config:
       control:
+      analytics:
       webui:
 {% if "contrail_controller" in groups["contrail"] %}
       vrouter:
diff --git a/playbooks/roles/docker/tasks/main.yml b/playbooks/roles/docker/tasks/main.yml
index 8d7971b..5ed9352 100644
--- a/playbooks/roles/docker/tasks/main.yml
+++ b/playbooks/roles/docker/tasks/main.yml
@@ -6,7 +6,6 @@
       - epel-release
       - gcc
       - git
-      - ansible-2.4.*
       - yum-utils
       - libffi-devel
     state: present
@@ -62,4 +61,4 @@
       - docker-py==1.10.6
       - docker-compose==1.9.0
     state: present
-    extra_args: --user
\ No newline at end of file
+    extra_args: --user
diff --git a/playbooks/roles/node/tasks/main.yml b/playbooks/roles/node/tasks/main.yml
index 0fb1751..d9ab111 100644
--- a/playbooks/roles/node/tasks/main.yml
+++ b/playbooks/roles/node/tasks/main.yml
@@ -1,13 +1,21 @@
 ---
-- name: Update kernel
+- name: Install required utilities
   become: yes
   yum:
-    name: kernel
-    state: latest
-  register: update_kernel
+    name:
+      - python3-devel
+      - libibverbs  ## needed by openstack controller node
+    state: present
 
-- name: Reboot the machine
-  become: yes
-  reboot:
-  when: update_kernel.changed
-  register: reboot_machine
+#- name: Update kernel
+#  become: yes
+#  yum:
+#    name: kernel
+#    state: latest
+#  register: update_kernel
+#
+#- name: Reboot the machine
+#  become: yes
+#  reboot:
+#  when: update_kernel.changed
+#  register: reboot_machine
diff --git a/playbooks/roles/restack_node/tasks/main.yml b/playbooks/roles/restack_node/tasks/main.yml
index a11e06e..f66d2ee 100644
--- a/playbooks/roles/restack_node/tasks/main.yml
+++ b/playbooks/roles/restack_node/tasks/main.yml
@@ -9,7 +9,7 @@
   become: yes
   pip:
     name:
-      - setuptools
+      - setuptools==43.0.0
       - requests
     state: forcereinstall
 

[centos@ip-172-31-10-212 networking-opencontrail]$ 

```



About 50 minutes is needed for installation to finish. 

Although /home/centos/devstack/openrc can be used for 'demo' user login, admin access is needed to specify network-type (empty for vRouter, 'vxlan' for ovs),
so adminrc need to be manually created.


```
[centos@ip-172-31-15-248 ~]$ cat adminrc 
export OS_PROJECT_DOMAIN_ID=default
export OS_REGION_NAME=RegionOne
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_IDENTITY_API_VERSION=3
export OS_PASSWORD=admin
export OS_AUTH_TYPE=password
export OS_AUTH_URL=http://172.31.15.248/identity  ## this needs to be modified
export OS_USERNAME=admin
export OS_TENANT_NAME=admin
export OS_VOLUME_API_VERSION=2
[centos@ip-172-31-15-248 ~]$ 


openstack network create testvn
openstack subnet create --subnet-range 192.168.100.0/24 --network testvn subnet1
openstack network create --provider-network-type vxlan testvn-ovs
openstack subnet create --subnet-range 192.168.110.0/24 --network testvn-ovs subnet1-ovs

 - two virtual-networks are created
[centos@ip-172-31-15-248 ~]$ openstack network list
+--------------------------------------+------------+--------------------------------------+
| ID                                   | Name       | Subnets                              |
+--------------------------------------+------------+--------------------------------------+
| d4e08516-71fc-401b-94fb-f52271c28dc9 | testvn-ovs | 991417ab-7da5-44ed-b686-8a14abbe46bb |
| e872b73e-100e-4ab0-9c53-770e129227e8 | testvn     | 27d828eb-ada4-4113-a6f8-df7dde2c43a4 |
+--------------------------------------+------------+--------------------------------------+
[centos@ip-172-31-15-248 ~]$

 - testvn's provider:network_type is empty

[centos@ip-172-31-15-248 ~]$ openstack network show testvn
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
| created_at                | 2020-01-18T16:14:42Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | e872b73e-100e-4ab0-9c53-770e129227e8 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | testvn                               |
| port_security_enabled     | True                                 |
| project_id                | 84a573dbfadb4a198ec988e36c4f66f6     |
| provider:network_type     | local                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 3                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 27d828eb-ada4-4113-a6f8-df7dde2c43a4 |
| tags                      |                                      |
| updated_at                | 2020-01-18T16:14:44Z                 |
+---------------------------+--------------------------------------+
[centos@ip-172-31-15-248 ~]$ 

 - and it is created in tungsten fabric's db

(venv) [root@ip-172-31-10-212 ~]# contrail-api-cli --host 172.31.10.212 ls -l virtual-network
virtual-network/e872b73e-100e-4ab0-9c53-770e129227e8  default-domain:admin:testvn
virtual-network/5a88a460-b049-4114-a3ef-d7939853cb13  default-domain:default-project:dci-network
virtual-network/f61d52b0-6577-42e0-a61f-7f1834a2f45e  default-domain:default-project:__link_local__
virtual-network/46b5d74a-24d3-47dd-bc82-c18f6bc706d7  default-domain:default-project:default-virtual-network
virtual-network/52925e2d-8c5d-4573-9317-2c346fb9edf0  default-domain:default-project:ip-fabric
virtual-network/2b0469cf-921f-4369-93a7-2d73350c82e7  default-domain:default-project:_internal_vn_ipv6_link_local
(venv) [root@ip-172-31-10-212 ~]# 


 - on the other hand, testvn-ovs's provider:network_type is vxlan, and segmentation id, mtu is automatically specified

[centos@ip-172-31-15-248 ~]$ openstack network show testvn
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
| created_at                | 2020-01-18T16:14:42Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | e872b73e-100e-4ab0-9c53-770e129227e8 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | testvn                               |
| port_security_enabled     | True                                 |
| project_id                | 84a573dbfadb4a198ec988e36c4f66f6     |
| provider:network_type     | local                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 3                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 27d828eb-ada4-4113-a6f8-df7dde2c43a4 |
| tags                      |                                      |
| updated_at                | 2020-01-18T16:14:44Z                 |
+---------------------------+--------------------------------------+
[centos@ip-172-31-15-248 ~]$ openstack network show testvn-ovs
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
| created_at                | 2020-01-18T16:14:47Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | d4e08516-71fc-401b-94fb-f52271c28dc9 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | testvn-ovs                           |
| port_security_enabled     | True                                 |
| project_id                | 84a573dbfadb4a198ec988e36c4f66f6     |
| provider:network_type     | vxlan                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 50                                   |
| qos_policy_id             | None                                 |
| revision_number           | 3                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 991417ab-7da5-44ed-b686-8a14abbe46bb |
| tags                      |                                      |
| updated_at                | 2020-01-18T16:14:49Z                 |
+---------------------------+--------------------------------------+
[centos@ip-172-31-15-248 ~]$ 

```

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

#### vRouters on GCE cannot reach other nodes in the same subnet

when vRouter is installed in GCE, it can't reach nodes in the same subnet.
This patch fixes the issue.

```
diff --git a/containers/vrouter/agent/entrypoint.sh b/containers/vrouter/agent/entrypoint.sh
index 18d15e2..222f5b4 100755
--- a/containers/vrouter/agent/entrypoint.sh
+++ b/containers/vrouter/agent/entrypoint.sh
@@ -131,20 +131,6 @@ vrouter_ip=${vrouter_cidr%/*}
 agent_name=${VROUTER_HOSTNAME:-"$(resolve_hostname_by_ip $vrouter_ip)"}
 [ -z "$agent_name" ] && agent_name="$DEFAULT_HOSTNAME"
 
-# Google has point to point DHCP address to the VM, but we need to initialize
-# with the network address mask. This is needed for proper forwarding of pkts
-# at the vrouter interface
-gcp=$(cat /sys/devices/virtual/dmi/id/chassis_vendor)
-if [ "$gcp" == "Google" ]; then
-    intfs=$(curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/network-interfaces/)
-    for intf in $intfs ; do
-        if [[ $phys_int_mac == "$(curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/network-interfaces/${intf}/mac)" ]]; then
-            mask=$(curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/network-interfaces/${intf}/subnetmask)
-            vrouter_cidr=$vrouter_ip/$(mask2cidr $mask)
-        fi
-    done
-fi
-
 vrouter_gateway_opts=''
 if [[ -n "$VROUTER_GATEWAY" ]] ; then
     vrouter_gateway_opts="gateway=$VROUTER_GATEWAY"
```

#### when it will be used with multus

By default, vRouter CNI uses thick-plugin mode, and it can assign multiple vRouter vnic to a container by itself.
https://github.com/Juniper/contrail-specs/blob/master/5.1/K8s_pods_multiple_interfaces.md

Unfortunately, it is not compabtible with multus (but I haven't yet fully tested), so when it is used with other CNI (such as SR-IOV, bridge, ..), meta-plugin mode (by default, false is used) need to be specified.
 - I guess if only one vNIC is used, meta-plugin parameter won't change something, but I haven't yet tested.

```
(contrail-container-builder)

diff --git a/containers/kubernetes/cni-init/entrypoint.sh b/containers/kubernetes/cni-init/entrypoint.sh
index 01a4698..18d36c8 100755
--- a/containers/kubernetes/cni-init/entrypoint.sh
+++ b/containers/kubernetes/cni-init/entrypoint.sh
@@ -41,7 +41,8 @@ cat << EOM > /host/etc_cni/net.d/10-contrail.conf
         "poll-timeout"  : 5,
         "poll-retries"  : 15,
         "log-file"      : "$LOG_DIR/cni/opencontrail.log",
-        "log-level"     : "4"
+        "log-level"     : "4",
+        "meta-plugin"     : true
     },
 
     "name": "contrail-k8s-cni",
```
