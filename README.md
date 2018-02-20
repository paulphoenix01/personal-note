## Installing RHEL 7.4 and OpenStack Platform 11 with Contrail 4.1 

After OS install
### On Director

Register Redhat
```
sudo subscription-manager register
sudo subscription-manager list --available --all --matches="*OpenStack*" | grep Pool
sudo subscription-manager attach --pool=<$Pool_ID>      ### Copy Pool_ID from above

sudo subscription-manager repos --disable=*
sudo subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-11-rpms --enable=rhel-7-server-openstack-11-devtools-rpms
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
