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

Run Director cmd tools with 'stack' user
```
sudo systemctl list-units openstack-*
source ~/stackrc
```
