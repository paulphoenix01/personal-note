## Installing RHEL 7.4 and OpenStack Platform 11 with Contrail 4.1 

After OS install with GUI > All Virtualization enable + Security Tools + Developments tools > Register RHEL 
### On KVM & Undercloud VM
Register Redhat
```
sudo subscription-manager register
sudo subscription-manager list --available --all --matches="*OpenStack*" | grep Pool
sudo subscription-manager attach --pool=<$Pool_ID>      ### Copy Pool_ID from above

sudo subscription-manager repos --disable=*
sudo subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-11-rpms

yum install -y net-tools vim
```
Add 'stack' user
```
useradd stack
passwd stack
#Enter Password

echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
chmod 0440 /etc/sudoers.d/stack

su - stack
mkdir ~/images
mkdir ~/templates
```

Set hostname and hosts (On Undercloud)
```
sudo hostnamectl set-hostname undercloud.jnpr.net
sudo hostnamectl set-hostname --transient undercloud.jnpr.net

# vim /etc/hosts
192.168.250.5 undercloud.jnpr.net undercloud
```
On KVM: Set static IP and gw
```
# vim /etc/resolv.conf
search jnpr.net
nameserver 8.8.8.8

# 192.168.250.254 should be NAT -> Internet. Add default gw here if necessary
# vim /etc/sysconfig/network-scripts/ifcfg-ens192
<...>
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
NAME=ens192
DEVICE=ens192
ONBOOT=yes
IPADDR=192.168.250.254
PREFIX=24
GATEWAY=<x.x.x.x>
# vim /etc/sysconfig/network-scripts/ifcfg-ens224
<...>
TYPE=Ethernet
BOOTPROTO=none
NAME=ens224
DEVICE=ens224
ONBOOT=yes
IPADDR=172.16.250.254
NETMASK=255.255.255.0

# vim /etc/sysconfig/network-scripts/ifcfg-br0
DEVICE=br0
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=none

# vim /etc/sysconfig/network-scripts/ifcfg-br1
DEVICE=br1
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=none

# (Optional) Add 3rd interface if not using NAT.
<...>
TYPE="Ethernet"
BOOTPROTO="none"
DEFROUTE="yes"
NAME="ens256"
DEVICE="ens256"
ONBOOT="yes"
IPADDR="10.250.42.9"
PREFIX="24"
GATEWAY="10.250.42.1"
DNS1="8.8.8.8"
DOMAIN="jnpr.net"
```


Update and Reboot
```
sudo yum update -y
sudo yum install -y libguestfs libguestfs-tools openvswitch virt-install kvm libvirt libvirt-python python-virtinst
sudo reboot
```

Start libvirtd and openvswitch for bridge
```
systemctl start libvirtd
systemctl start openvswitch
systemctl enable libvirtd
systemctl enable openvswitch

ovs-vsctl add-br br0
ovs-vsctl add-br br1
ovs-vsctl add-port br0 ens192   <<< Nic #1
ovs-vsctl add-port br1 ens224   <<< Nic #2
cat << EOF > br0.xml
<network>
  <name>br0</name>
  <forward mode='bridge'/>
  <bridge name='br0'/>
  <virtualport type='openvswitch'/>
  <portgroup name='overcloud'>
    <vlan trunk='yes'>
      <tag id='700' nativeMode='untagged'/>
      <tag id='710'/>
      <tag id='720'/>
      <tag id='730'/>
      <tag id='740'/>
      <tag id='750'/>
    </vlan>
  </portgroup>
</network>
EOF
cat << EOF > br1.xml
<network>
  <name>br1</name>
  <forward mode='bridge'/>
  <bridge name='br1'/>
  <virtualport type='openvswitch'/>
</network>
EOF
virsh net-define br0.xml
virsh net-start br0
virsh net-autostart br0
virsh net-define br1.xml
virsh net-start br1
virsh net-autostart br1

## Issue ifconfig and check if br0 and br1 is up. If not use:
ifconfig br0 up
ifconfig br1 up
```

### On 'root'@kvm: Define VMs and run script
```
# vim define-vms.sh
num=0
for i in compute control contrail-controller contrail-analytics contrail-analytics-database
do
  num=$(expr $num + 1)
  qemu-img create -f qcow2 /var/lib/libvirt/images/${i}_${num}.qcow2 40G
  virsh define /dev/stdin <<EOF
$(virt-install --name ${i}_$num --disk /var/lib/libvirt/images/${i}_${num}.qcow2 --vcpus=4 --ram=16348 --network network=br0,model=virtio,portgroup=overcloud --network network=br1,model=virtio --virt-type kvm --import --os-variant rhel7 --serial pty --console pty,target_type=virtio --print-xml)
EOF
done
```
### Get provisioning interface mac addresses for ironic PXE
```
# vim get-ironic-mac.sh
num=0
for i in compute control contrail-controller contrail-analytics contrail-analytics-database
do
  num=$(expr $num + 1)
  prov_mac=`virsh domiflist ${i}_${num}|grep br0|awk '{print $5}'`
  echo ${prov_mac} ${i}_${num} 192.168.250.254 ${i} >> ironic_list.txt
done
```
```
# Output file would be: (without space and title) 
MAC_ADDRESS       VM_DOMAIN_NAME                br0_KVM_IP(for undercloud power_mgnt)    Profile_Role
52:54:00:5d:c7:41 compute_1                     192.168.250.254                          compute
52:54:00:c4:82:82 control_2                     192.168.250.254                          control
52:54:00:92:fe:3c contrail-controller_3         192.168.250.254                          contrail-controller
52:54:00:7b:c5:e2 contrail-analytics_4          192.168.250.254                          contrail-analytics
52:54:00:23:2e:88 contrail-analytics-database_5 192.168.250.254                          contrail-analytics-database
```
# On Undercloud

Configuring Undercloud, with 'stack' user
```
sudo yum install -y python-tripleoclient

su - stack
cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf
# vim ~/undercloud.conf
[DEFAULT]
local_ip = 192.168.250.1/24
undercloud_public_vip = 192.168.250.6
undercloud_admin_vip = 192.168.250.7
local_interface = eth1
masquerade_network = 192.168.250.0/24
dhcp_start = 192.168.250.10
dhcp_end = 192.168.250.50
enable_ui = true
network_cidr = 192.168.250.0/24
network_gateway = 192.168.250.1
inspection_iprange = 192.168.250.130,192.168.250.150
generate_service_certificate = true
certificate_generation_ca = local
```
Run the installation
```
openstack undercloud install
```

Install Image and DNS with 'stack' user
```
sudo systemctl list-units openstack-*
source ~/stackrc

sudo yum install rhosp-director-images rhosp-director-images-ipa
cd ~/images
for i in /usr/share/rhosp-director-images/overcloud-full-latest-11.0.tar /usr/share/rhosp-director-images/ironic-python-agent-latest-11.0.tar; do tar -xvf $i; done
openstack overcloud image upload --image-path /home/stack/images/
openstack subnet list
openstack subnet set --dns-nameserver 8.8.8.8 <subnet-uuid>
```

Double Check
```
$ openstack image list
+--------------------------------------+------------------------+
| ID                                   | Name                   |
+--------------------------------------+------------------------+
| 765a46af-4417-4592-91e5-a300ead3faf6 | bm-deploy-ramdisk      |
| 09b40e3d-0382-4925-a356-3a4b4f36b514 | bm-deploy-kernel       |
| ef793cd0-e65c-456a-a675-63cd57610bd5 | overcloud-full         |
| 9a51a6cb-4670-40de-b64b-b70f4dd44152 | overcloud-full-initrd  |
| 4f7e33f4-d617-47c1-b36f-cbe90f132e5d | overcloud-full-vmlinuz |
+--------------------------------------+------------------------+
$ ls -l /httpboot
total 341460
-rwxr-xr-x. 1 root              root                5153184 Mar 31 06:58 agent.kernel
-rw-r--r--. 1 root              root              344491465 Mar 31 06:59 agent.ramdisk
-rw-r--r--. 1 ironic-inspector  ironic-inspector        337 Mar 31 06:23 inspector.ipxe

```

### On KVM Host, provision contrail-vm
```
# vim define-contrail-vm.sh
num=0
for i in compute control contrail-controller contrail-analytics contrail-analyticsdatabase
do
  num=$(expr $num + 1)
  sudo qemu-img create -f qcow2 /var/lib/libvirt/images/${i}_${num}.qcow2 40G
  sudo virt-install --name ${i}_$num --disk /var/lib/libvirt/images/${i}_${num}.qcow2 --vcpus=4 --ram=16348 --network network=br0,model=virtio --network network=default,model=virtio --virt-type kvm --import --os-variant rhel7 --serial pty --console pty,target_type=virtio --print-xml > ${i}_$num.xml
  virsh define ${i}_$num.xml
done
```
```
# vim get-ironic-list.sh
for i in compute control contrail-controller contrail-analytics contrail-analyticsdatabase
do
  num=$(expr $num + 1)
  prov_mac=`virsh domiflist ${i}_${num}|grep br0|awk '{print $5}'`
  echo ${prov_mac} ${i}_${num} ${i} >> ironic_list.txt
done
```
Execute and copy to undercloud
```
chmod 700 *.sh
./define-contrail-vm.sh
./get-ironic-list.sh
scp ironic_list.txt stack@192.168.122.111:~/.
```

######
(Skip for now) Enable fake PXE (Power Management) for Ironic. Poweron/off will be done manually.
```
# sudo vi /etc/ironic/ironic.conf
# Add fake_pxe to enabled_drivers
enabled_drivers = .....,fake_pxe

#Restart service
sudo systemctl restart openstack-ironic-conductor   openstack-ironic-api
```
######

Back to Undercloud VM, on root.
Copy SSH key
```
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.122.1
```
On 'stack' user on Undercloud
```
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.122.1

#vim create-instackenv.sh
jq . << EOF > ~/instackenv.json
{
  "ssh-user": "root",
  "ssh-key": "$(awk '{printf "%s\\n", $0}' ~/.ssh/id_rsa)",
  "power_manager": "nova.virt.baremetal.virtual_power_driver.VirtualPowerManager",
  "host-ip": "192.168.122.1",
  "arch": "x86_64",
  "nodes": [
    {
      "mac": [
        "$(sed -n 1p ironic_list.txt  | awk '{print $1}')"
      ],
      "name":"$(sed -n 1p ironic_list.txt  | awk '{print $2}')",
      "capabilities":"profile:$(sed -n 1p ironic_list.txt  | awk '{print $3}')",
      "cpu": "4",
      "memory": "8192",
      "disk": "50",
      "arch": "x86_64",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(awk '{printf "%s\\n", $0}' ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "pm_user": "root"
    },
    {
      "mac": [
        "$(sed -n 2p ironic_list.txt  | awk '{print $1}')"
      ],
      "name":"$(sed -n 2p ironic_list.txt  | awk '{print $2}')",
      "capabilities":"profile:$(sed -n 2p ironic_list.txt  | awk '{print $3}')",
      "cpu": "4",
      "memory": "8192",
      "disk": "50",
      "arch": "x86_64",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(awk '{printf "%s\\n", $0}' ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "pm_user": "root"
    },
    {
      "mac": [
        "$(sed -n 3p ironic_list.txt  | awk '{print $1}')"
      ],
      "name":"$(sed -n 3p ironic_list.txt  | awk '{print $2}')",
      "capabilities":"profile:$(sed -n 3p ironic_list.txt  | awk '{print $3}')",
      "cpu": "4",
      "memory": "8192",
      "disk": "50",
      "arch": "x86_64",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(awk '{printf "%s\\n", $0}' ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "pm_user": "root"
    },
    {
      "mac": [
        "$(sed -n 4p ironic_list.txt  | awk '{print $1}')"
      ],
      "name":"$(sed -n 4p ironic_list.txt  | awk '{print $2}')",
      "capabilities":"profile:$(sed -n 4p ironic_list.txt  | awk '{print $3}')",
      "cpu": "4",
      "memory": "8192",
      "disk": "50",
      "arch": "x86_64",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(awk '{printf "%s\\n", $0}' ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "pm_user": "root"
    },
    {
      "mac": [
        "$(sed -n 5p ironic_list.txt  | awk '{print $1}')"
      ],
      "name":"$(sed -n 5p ironic_list.txt  | awk '{print $2}')",
      "capabilities":"profile:$(sed -n 5p ironic_list.txt  | awk '{print $3}')",
      "cpu": "4",
      "memory": "8192",
      "disk": "50",
      "arch": "x86_64",
      "pm_addr": "192.168.122.1",
      "pm_password": "$(awk '{printf "%s\\n", $0}' ~/.ssh/id_rsa)",
      "pm_type": "pxe_ssh",
      "pm_user": "root"
    }
    
  ] 
} 
EOF
```
Run the create-instackenv and import to overcloud. Dont forget #source stackrc
```
chmod 700 create-instackenv.sh
./create-instackenv.sh
source stackrc
```
```
### Dont use baremetal import, it wont work.
# openstack overcloud node import instackenv.json -v
START with options: [u'overcloud', u'node', u'import', u'instackenv.json', u'-v']
command: overcloud node import -> tripleoclient.v1.overcloud_node.ImportNode
Using auth plugin: password
Started Mistral Workflow tripleo.baremetal.v1.register_or_update. Execution ID: 8bfae3a9-7427-4909-b4c0-efa3887ccc45
Waiting for messages on queue 'df197b71-e903-4374-99a5-c6d491398957' with no timeout.
Successfully registered node UUID 8c3ae5bd-841e-42be-9b66-749898de6868
Successfully registered node UUID 7b8b699d-716f-47b4-8a6d-6b51617da8e8
Successfully registered node UUID 19d10dfc-4255-47db-a6e1-71fc69084c68
Successfully registered node UUID 09a19dd2-a52d-4207-b517-cc2211c46cad
Successfully registered node UUID 0eee68e2-f4ae-4587-bca2-6346312b8d8c
END return value: 0

```

Perform introspection on baremetal node list
```
openstack baremetal configure boot
for node in $(openstack baremetal node list -c UUID -f value) ; do openstack baremetal node manage $node ; done
#Ignore this error if show, as long as baremetal node list show 'manageable'
#The requested action "manage" can not be performed on node "01613eb8-ce12-4876-a4cd-31b193a9929b" while it is in state "manageable". (HTTP 400)
```
```
[stack@undercloud ~]$ openstack baremetal node list
+-------------------------------+------------------------------+---------------+-------------+--------------------+-------------+
| UUID                          | Name                         | Instance UUID | Power State | Provisioning State | Maintenance |
+-------------------------------+------------------------------+---------------+-------------+--------------------+-------------+
| a9b39c4e-                     | compute_1                    | None          | power off   | manageable         | False       |
| a7f8-4064-b432-2845fd3ecea5   |                              |               |             |                    |             |
| 5e221c67-0c73-455f-           | control_2                    | None          | power off   | manageable         | False       |
| bac5-fabbfc09adfd             |                              |               |             |                    |             |
| 1e058c87-4314-4e72-9363-e0fac | contrail-controller_3        | None          | power off   | manageable         | False       |
| 58d04e8                       |                              |               |             |                    |             |
| ee5352b0-ce8a-                | contrail-analytics_4         | None          | power off   | manageable         | False       |
| 4d5f-b576-b1f6fdf18d0b        |                              |               |             |                    |             |
| f98ea941-dfad-                | contrail-analyticsdatabase_5 | None          | power off   | manageable         | False       |
| 485b-8676-93e6661a1c57        |                              |               |             |                    |             |
+-------------------------------+------------------------------+---------------+-------------+--------------------+-------------+
```

```
openstack overcloud node introspect --all-manageable --provide
#Watch virt-manager, VM should be powered on and executing boot sequence thru PXE, then powered off. 

![Imgur](https://i.imgur.com/E53CZ2W.png)
[stack@undercloud ~]$ openstack overcloud node introspect --all-manageable --provide
Started Mistral Workflow tripleo.baremetal.v1.introspect_manageable_nodes. Execution ID: 3afd8d05-8224-4372-8cde-62256beccd4f
Waiting for introspection to finish...
Waiting for messages on queue '5e3589be-48b5-409d-a4c3-d67e430c49c2' with no timeout.
Introspection for UUID 1e058c87-4314-4e72-9363-e0fac58d04e8 finished successfully.
Introspection for UUID 5e221c67-0c73-455f-bac5-fabbfc09adfd finished successfully.
Introspection for UUID f98ea941-dfad-485b-8676-93e6661a1c57 finished successfully.
Introspection for UUID a9b39c4e-a7f8-4064-b432-2845fd3ecea5 finished successfully.
Introspection for UUID ee5352b0-ce8a-4d5f-b576-b1f6fdf18d0b finished successfully.
Introspection completed.

#Node list should show state = available.
[stack@undercloud ~]$ openstack baremetal node list
+-------------------------------+------------------------------+---------------+-------------+--------------------+-------------+
| UUID                          | Name                         | Instance UUID | Power State | Provisioning State | Maintenance |
+-------------------------------+------------------------------+---------------+-------------+--------------------+-------------+
| 01613eb8-ce12-4876-a4cd-      | compute_1                    | None          | power off   | available          | False       |
| 31b193a9929b                  |                              |               |             |                    |             |
| 04309846-e21c-                | control_2                    | None          | power off   | available          | False       |
| 4a2e-a621-e72d9a36c2f7        |                              |               |             |                    |             |
| 5350fa22-51ea-                | contrail-controller_3        | None          | power off   | available          | False       |
| 472f-b852-2d45342ef18a        |                              |               |             |                    |             |
| fe2c0afd-94ae-                | contrail-analytics_4         | None          | power off   | available          | False       |
| 48b6-bf96-a9e875757f87        |                              |               |             |                    |             |
| 0d31510c-4990-4d17-89b4-de78e | contrail-analyticsdatabase_5 | None          | power off   | available          | False       |
| 136c498                       |                              |               |             |                    |             |
+-------------------------------+------------------------------+---------------+-------------+--------------------+-------------+
```


Create OpenStack Flavor
```
#Delete old compute/controller flavor first (due to default only 1cpu/2gb RAM. Change flavor as desired.
openstack flavor delete compute
openstack flavor delete control

# vim node-profile.sh
for i in compute control contrail-controller contrail-analytics contrail-database contrail-analytics-database
do
  openstack flavor create $i --ram 8192 --vcpus 4 --disk 50;
  openstack flavor set --property "cpu_arch"="x86_64" --property "capabilities:boot_option"="local" --property "capabilities:profile"="${i}" ${i};
done

#Run the script.
chmod 700 node-profile.sh
./node-profile.sh

#openstack flavor list
+--------------------------------------+-----------------------------------------+------+------+-----------+-------+-----------+
| ID                                   | Name                                    |  RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+-----------------------------------------+------+------+-----------+-------+-----------+
| 10c5be43-e4ae-4b1e-ac5c-97d67b900d64 | compute                                 | 8192 |   50 |         0 |     4 | True      |
| 25b489d3-791c-412e-9968-b8009a04c601 | block-storage                           | 4096 |   40 |         0 |     1 | True      |
| 2ad8a937-0ed4-4423-84d3-f7445d80a873 | ceph-storage                            | 4096 |   40 |         0 |     1 | True      |
| 47c10143-845f-4576-a88e-31b52884cd29 | contrail-analytics                      | 8192 |   50 |         0 |     4 | True      |
| 4ed838b6-18b3-481b-ae5e-cc5a18ca2b24 | /usr/share/rhosp-director-images        | 8192 |   50 |         0 |     4 | True      |
|                                      | /ironic-python-agent-latest-11.0.tar    |      |      |           |       |           |
| 6967b7a3-f5a9-4fdc-8f71-2c51b76c26f1 | contrail-analytics-database             | 8192 |   50 |         0 |     4 | True      |
| 704d8524-b190-4e96-b284-e09e31de712c | control                                 | 8192 |   50 |         0 |     4 | True      |
| 7425cc76-2fe8-44f9-b609-677b208412a3 | contrail-database                       | 8192 |   50 |         0 |     4 | True      |
| c3c74956-5419-444a-84f1-517f9c9fe6dc | contrail-controller                     | 8192 |   50 |         0 |     4 | True      |
| c6ede8c5-ab0e-4a09-a942-6d3a7c7c956d | swift-storage                           | 4096 |   40 |         0 |     1 | True      |
| eb95ea01-41c3-4ee0-8bdf-0920dd51531c | baremetal                               | 4096 |   40 |         0 |     1 | True      |
+--------------------------------------+-----------------------------------------+------+------+-----------+-------+-----------+

```

Before continue, make sure those return non-zero value
```
[stack@undercloud ~]$ nova hypervisor-stats
+----------------------+-------+
| Property             | Value |
+----------------------+-------+
| count                | 5     |
| current_workload     | 0     |
| disk_available_least | 195   |
| free_disk_gb         | 195   |
| free_ram_mb          | 81740 |
| local_gb             | 195   |
| local_gb_used        | 0     |
| memory_mb            | 81740 |
| memory_mb_used       | 0     |
| running_vms          | 0     |
| vcpus                | 20    |
| vcpus_used           | 0     |
+----------------------+-------+
[stack@undercloud ~]$ nova hypervisor-list
+----+--------------------------------------+-------+---------+
| ID | Hypervisor hostname                  | State | Status  |
+----+--------------------------------------+-------+---------+
| 1  | 04309846-e21c-4a2e-a621-e72d9a36c2f7 | up    | enabled |
| 2  | 01613eb8-ce12-4876-a4cd-31b193a9929b | up    | enabled |
| 3  | fe2c0afd-94ae-48b6-bf96-a9e875757f87 | up    | enabled |
| 4  | 5350fa22-51ea-472f-b852-2d45342ef18a | up    | enabled |
| 5  | 0d31510c-4990-4d17-89b4-de78e136c498 | up    | enabled |
+----+--------------------------------------+-------+---------+
```

## Provisioning Overcloud
###Those are done on 'stack' user on Undercloud VM.
Install Puppet modules:
```
mkdir -p ~/usr/share/openstack-puppet/modules
git clone https://github.com/Juniper/contrail-tripleo-puppet -b stable/ocata ~/usr/share/openstack-puppet/modules/tripleo
git clone https://github.com/Juniper/puppet-contrail -b master ~/usr/share/openstack-puppet/modules/contrail
tar czvf puppet-modules.tgz usr/
upload-swift-artifacts -f puppet-modules.tgz
```
Get Overcloud images
```
source ~/stackrc
sudo yum install rhosp-director-images rhosp-director-images-ipa
mkdir ~/images
cd ~/images
for i in /usr/share/rhosp-director-images/overcloud-full-latest-11.0.tar /usr/share/rhosp-director-images/ironic-python-agent-latest-11.0.tar; do tar -xvf $i; done
openstack overcloud image upload --image-path /home/stack/images/
cd ~
```
Get Contrail RPM package
```
sudo mkdir /var/www/html/contrail
#Download tgz file on Juniper website, Redhat-Contrail-Networking-4.1.tgz
sudo tar zxvf contrail-install-packages_4.1.0.0-8-ocata_redhat7.tgz -C /var/www/html/contrail/.
```
Install Contrail RPM package from tgz file  (don't know what it's for, since we will get the templates directly from github)
```
cd /var/www/html/contrail/
sudo yum localinstall -y contrail-tripleo-heat-templates-4.1.0.0-8.el7.noarch.rpm 
```

Get TripleO Heat Templates:
```
cp -r /usr/share/openstack-tripleo-heat-templates/ ~/tripleo-heat-templates
git clone https://github.com/Juniper/contrail-tripleo-heat-templates -b stable/ocata
cp -r contrail-tripleo-heat-templates/environments/contrail ~/tripleo-heat-templates/environments
cp -r contrail-tripleo-heat-templates/puppet/services/network/* ~/tripleo-heat-templates/puppet/services/network
cp -r contrail-tripleo-heat-templates/extraconfig ~/tripleo-heat-templates
cp -r contrail-tripleo-heat-templates/network ~/tripleo-heat-templates
```

## Sample template ##
``` 
TODO
```
