## Installing RHEL 7.4 and OpenStack Platform 11 with Contrail 4.1 

After OS install
### On Director
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
