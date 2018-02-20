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
Update and Reboot
```
sudo yum update -y
sudo reboot
```

Configuring Undercloud, with 'stack' user
```
sudo yum install -y python-tripleoclient
cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf
# vim ~/undercloud.conf
local_ip = 192.168.250.5/24
undercloud_public_vip = 192.168.250.6
undercloud_admin_vip = 192.168.250.7
local_interface = eth1
masquerade_network = 192.168.250.0/24
dhcp_start = 192.168.250.10
dhcp_end = 192.168.250.50
enable_ui = true
network_cidr = 192.168.250.0/24
network_gateway = 192.168.250.5
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

Enable fake PXE (Power Management) for Ironic. Poweron/off will be done manually.
```
# sudo vi /etc/ironic/ironic.conf
# Add fake_pxe to enabled_drivers
enabled_drivers = .....,fake_pxe

#Restart service
sudo systemctl restart openstack-ironic-conductor   openstack-ironic-api
```

```
#Create instackenv.json, keep no need change MAC (it's fake)
{
    "nodes":[
        {
            "mac":[
                "00:50:56:80:de:bb"
            ],
            "cpu":"4",
            "memory":"24576",
            "disk":"200",
            "arch":"x86_64",
            "pm_type":"fake_pxe",
            "pm_addr":"192.168.250.10"
        },
        {
            "mac":[
                "00:50:56:80:c8:4b"
            ],
            "cpu":"8",
            "memory":"32768",
            "disk":"250",
            "arch":"x86_64",
            "pm_type":"fake_pxe",
            "pm_addr":"192.168.250.11"
        }
    ]
}

#####
openstack overcloud node import ~/instackenv.json
openstack baremetal node list
```
