# cloudstack-guid-unbutu-22.04
## Cloudstack Installation Tutorial
#### 1. step 1: remote server 
ssh ip server login super account : sudo -i
#### 2. Modify The Network Interface Configuration ( Ignore if you do not need to configure your Network)
##### Step 1: 
```cd /etc/netplan```
```nano 01-netplan.yaml```
##### Step 2: modify:
  ```yaml
  network:
   version: 2
   renderer: networkd
   ethernets:
     eno1:
       dhcp4: false
       dhcp6: false
       optional: true
     eno2:
       dhcp4: false
       dhcp6: false
       optional: true
   bridges:
     cloudbr0:
       addresses: [192.168.10.33/24]
       routes:
        - to: default
          via: 192.168.10.1
       nameservers:
         addresses: [1.1.1.1,8.8.8.8]
       interfaces: [eno1]
       dhcp4: false
       dhcp6: false
       parameters:
         stp: false
         forward-delay: 0
```
#### Step 3: Apply your changes in the netplan configuration by using this
```
sudo -i
netplan generate
netplan apply
reboot
```
#### 3. Turn on NTP for time synchronization.
Install chrony.
```apt-get install chrony```
#### 4. Add DEB package repository
Type:<br>
```nano /etc/apt/sources.list.d/cloudstack.list```
<br>
write :<br> `deb http://download.cloudstack.org/ubuntu focal 4.17`
<br>
Crtl+ X -> exit => Y to save
<br>
We now have to add the public key to the trusted keys.
```wget -O - http://download.cloudstack.org/release.asc |sudo apt-key add -```

Now update your local apt cache.
<br>
```sudo apt-get update```
<br>
Install 
<br>
```sudo apt-get install cloudstack-management```
#### 5. Install the Database on the Management Server Node
```sudo apt-get install mysql-server```
<br>
Config database
<br>
```nano /etc/mysql/mysql.conf.d/mysqld.cnf```
<br>
Write code before [mysqld] 
```yaml
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'

```
<br>
Then restart mysql service to apply changes.

```systemctl restart mysql```
<br>
#### 6.Deploy database
```cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:password -i 192.168.10.33 ```
- password: password for user root ubuntu
- i: ip server

#### 7. Storage Setup
```yaml
apt-get install nfs-kernel-server quota

echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```
#### 8. Configure NFS Server
```yaml
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
```
Restart service
```service nfs-kernel-server restart```
#### 9. Setup KVM
```yaml
apt-get install qemu-kvm cloudstack-agent
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

nano /etc/default/libvirtd
#On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.
#uncomment LIBVIRTD_ARGS="--listen"

```
#### 10. Configure Default Libvirtd Config
```yaml
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf

systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
#### 11. Generate host id

```yaml
apt-get install uuid
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

#### 12. Disable apparmour on libvirtd
```yaml
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper

```
#### 13. Launch Management Server
```yaml
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log

```
After management server is UP, proceed to http://192.168.10.33(i.e. the cloudbr0-IP):8080/client and log in using the default credentials - username admin and password password.

##### 14. Enable XRDP
```yaml
apt update
apt install xfce4 xfce4-goodies -y
apt install xrdp -y
nano /etc/xrdp/xrdp.ini
systemctl restart xrdp
systemctl status xrdp

```
