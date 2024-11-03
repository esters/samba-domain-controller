# Samba as Microsoft Active Directory Domain Controller

## Microsoft Active Directory Domain Controller on a Debian 12 host

The documentation shows the steps on how to install and deploy a Domain Controller on a Debian 12 host. Virtualization was done by KVM (1 x vCPU, 1 Gb RAM, 12Gb SSD), this documentation does not cover the steps to set up and run a dedicated host.

### Reference documentation

The SAMBA project provides very good and thoughtful documentation with the necessary steps to get a Domain Controller up and running

Introduction - [Setting_up_Samba_as_an_Active_Directory_Domain_Controller](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller)

**NOTE**: The documentation uses "samdom.example.com" as the DC's primary domain. Since I am resident in Latvia. NIC (the domain registar for .lv) provides a free second level domain (*.id.lv) for every private person, and I will be using __dc01.domain.id.lv__ in my case.

Follow the steps in the SAMBA documentation:
* Install distribution (Debian) specific packages:
```
apt-get update && apt-get install acl attr samba winbind libpam-winbind libnss-winbind krb5-config krb5-user dnsutils python3-setproctitle smbclient ldb-tools python3-cryptography
```
* Install `chrony` as the NTP server
```
apt-get update && apt-get install chrony
```
* Set up the Chrony NTP server followed by the documentation here - [Configuring Time Synchronisation on a DC with chrony](https://wiki.samba.org/index.php/Time_Synchronisation#With_chrony)

**NOTE**: The "ntp_signd" directory has to be manually created in `/var/lib/samba/` with the required permissions as per documentation

* Set a static IP address to your DC's network interface. I am using systemd-networkd as the network manager. Debian Wiki tutorial - [SystemdNetworkd](https://wiki.debian.org/SystemdNetworkd)
* Edit the `/etc/hosts` file and add the fully-qualified domain name and the short host name of the DC
```
127.0.0.1  localhost
192.168.99.1 debian-dc01.dc01.domain.lv  debian-dc01
```
* Stop all SAMBA related services, in case they are running
```
systemctl stop samba winbind nmbd smbd
```
* Remove any existing configuration files for SAMBA - `smb.conf` and `*.tdb` and `*.ldb` files - [Reference](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller#Only_Applicable_if_Samba_was_Previously_Installed)
* Provision a SAMBA Active Directory host in Interactive Mode - [Reference](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller#Provisioning_Samba_AD_in_Interactive_Mode)
```
# samba-tool domain provision --use-rfc2307 --interactive
Realm: dc01.domain.id.lv
Domain: dc01
Server Role: dc
DNS backend: SAMBA_INTERNAL
DNS forwarder IP address: 8.8.8.8 (You can set up more than one in smb.conf)
Administrator password: SuperP4ss
...
``` 
* Disable `systemd-resolved` service and create a static `/etc/resolv.conf` file cotaining only the following entries, as SAMBA now will be the DNS resolver:
```
search dc01.domain.id.lv
nameserver 127.0.0.1
```
* Copy the created SAMBA Kerberos configuration file [Reference](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller#Configuring_Kerberos):
```
cp -av /var/lib/samba/private/krb5.conf /etc/krb5.conf
```
* Activate SAMBA AD DC service [Reference](https://samba.tranquil.it/doc/en/samba_config_server/debian/server_install_samba_debian.html)
```
# systemctl mask smbd nmbd winbind
# systemctl disable smbd nmbd winbind
# systemctl unmask samba-ad-dc
# systemctl enable samba-ad-dc
```
* Reboot the DC
* Optionaly - Create a DNS reverse zone [Reference](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller#Create_a_reverse_zone)
* Verify that File Server is operational and verify that AD DNS is working correctly and Kerberos is working as intended [Reference1](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller#Verifying_the_File_Server_(Optional))
  [Reference2](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller#Verifying_DNS_(Optional))
  [Reference3](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller#Verifying_Kerberos_(Optional))
* If there are issues, follow the guidelines here - [Samba_AD_DC_Troubleshooting](https://wiki.samba.org/index.php/Samba_AD_DC_Troubleshooting)
