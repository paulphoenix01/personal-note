## Installing RHEL 7.4 and OpenStack Platform 11 with Contrail 4.1 

After OS install
### On Director

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

Set hostname and hosts
```
sudo hostnamectl set-hostname undercloud.jnpr.net
sudo hostnamectl set-hostname --transient undercloud.jnpr.net

# vim /etc/hosts
192.168.250.5   undercloud.jnpr.net
```
Set static IP and gw
```
# vim /etc/resolv.conf
search jnpr.net
nameserver 192.168.122.1

# vim /etc/sysconfig/network-scripts/ifcfg-eth0
<...>
TYPE="Ethernet"
BOOTPROTO="none"
NAME="eth0"
DEVICE="eth0"
ONBOOT="yes"
IPADDR=192.168.122.111
NETMASK=255.255.255.0
GATEWAY=192.168.122.1

# vim /etc/sysconfig/network-scripts/ifcfg-eth1
<...>
TYPE=Ethernet
BOOTPROTO=none
NAME=eth1
DEVICE=eth1
ONBOOT=yes
IPADDR=192.168.250.1
NETMASK=255.255.255.0
```

Update and Reboot
```
sudo yum update -y
sudo reboot
```

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

###On KVM Host, provision contrail-vm
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
# openstack overcloud node import ~/instackenv.json
Started Mistral Workflow tripleo.baremetal.v1.register_or_update. Execution ID: 7775657a-3b82-4a62-b5a1-6f77e32e7591
Waiting for messages on queue '989fdf89-6963-458e-b599-4e8455317c1f' with no timeout.
Successfully registered node UUID b4d8d83b-c667-431e-ae97-8bf6b7eb4500
Successfully registered node UUID b909e770-12d4-4a03-b0af-4765fb231d7e

# openstack baremetal node list
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| b4d8d83b-c667-431e-ae97-8bf6b7eb4500 | None | None          | None        | manageable         | False       |
| b909e770-12d4-4a03-b0af-4765fb231d7e | None | None          | None        | manageable         | False       |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
```
