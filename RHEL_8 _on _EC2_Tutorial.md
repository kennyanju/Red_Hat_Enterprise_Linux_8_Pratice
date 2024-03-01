
# RHEL 8 on Amazon EC2 

## Prerequisites

- An AWS account.
- Basic knowledge of AWS EC2, RHEL, and terminal usage.

## Step 1: Launch EC2 Instance

1. Sign in to the AWS Management Console.
2. Navigate to EC2 and click "Launch Instance".
3. Choose "Red Hat Enterprise Linux 8" as the Amazon Machine Image (AMI).
4. Select an instance type, then click "Next: Configure Instance Details".
5. Configure instance details as required, ensuring your instance is within the desired subnet.
6. Add storage if needed, then click "Next: Add Tags".
7. Add relevant tags, then click "Next: Configure Security Group".
8. Create a new security group allowing SSH access.
9. Review and launch the instance.

## Prepare NFS Server on Amazon EC2

### 1. Update System Software

```shell
sudo yum update -y
```

### 2. Install Linux Guest Additions

For NFS setup:

```shell
sudo yum install -y nfs-utils nfs4-acl-tools
```

### 3. Gather Statistics from System

```shell
vmstat
```

### 4. Generate Report on System Utilization

```shell
sar -u 5 5
```

### 5. Install and Configure NFS Server

```shell
sudo yum install -y nfs-utils
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
```

### 6. Create NFS Export Directory

```shell
sudo mkdir /var/nfs_share
sudo chown nfsnobody:nfsnobody /var/nfs_share
sudo chmod 755 /var/nfs_share
```

### 7. Configure NFS Exports

Edit `/etc/exports` and add:

```text
/var/nfs_share <client-IP>(rw,sync,no_root_squash,no_all_squash)
```

Apply changes:

```shell
sudo exportfs -arv
```

### 8. Create Access Control

Configure `/etc/exports` for specific client IP addresses or networks for access control.

### 9. Configure Firewalls

```shell
sudo firewall-cmd --permanent --zone=public --add-service=nfs
sudo firewall-cmd --permanent --zone=public --add-service=mountd
sudo firewall-cmd --permanent --zone=public --add-service=rpc-bind
sudo firewall-cmd --reload
```

### 10. Mount NFS Share on Client

On client:

```shell
sudo mount -t nfs <server-IP>:/var/nfs_share /mnt
```

To mount automatically at boot, add to `/etc/fstab` on the client:

```text
<server-IP>:/var/nfs_share /mnt nfs defaults 0 0
```

## Prepare NFS Server with NTP, LDAP, and Kerberos

### 1. Install and Configure NTP Services

```shell
sudo yum install -y chrony
sudo systemctl enable chronyd
sudo systemctl start chronyd
```

### 2. Configure Chrony Server and Client

Server Configuration:

- Edit `/etc/chrony.conf` to add server settings.

Client Configuration:

- Add server IP to `/etc/chrony.conf`, e.g., `server <server-IP> iburst`.

## 3. Configure Time and Date

```shell
timedatectl set-timezone <Your_Timezone>
```

### 4. Install and Configure LDAP Server

```shell
sudo yum install -y openldap openldap-servers openldap-clients
sudo systemctl start slapd
sudo systemctl enable slapd
```

### 5. Setup LDAP Database

```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f backend.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f frontend.ldif
```

### 6. Create LDAP User

```shell
ldapadd -x -W -D "cn=admin,dc=example,dc=com" -f adduser.ldif
```

### 7. Install and Configure LDAP Client

```shell
sudo yum install -y openldap-clients nss-pam-ldapd
authconfig --enableldap --enableldapauth --ldapserver=<server-IP> --ldapbasedn="dc=example,dc=com" --update
```

### 8. Install and Configure Kerberos Server Packages

```shell
sudo yum install -y krb5-server krb5-libs krb5-workstation
```

### 9. Create Kerberos Database

```shell
kdb5_util create -s
```

### 10. Configure Kerberos Client Authentication

Server Configuration:

- Edit `/etc/krb5.conf` with your realm and server details.

Client Configuration:

- Install necessary packages and configure `/etc/krb5.conf` similarly.

## NFS Server Preparation with Network Configurations

### 1. Gather Network Information

```shell
ip addr show
```

### 2. Configure IP and Subnet Mask

```shell
sudo nmcli con mod <connection-name> ipv4.addresses <IP-address>/<subnet-mask>
sudo nmcli con up <connection-name>
```

### 3. Configure Interface Bonding Using nmcli

```shell
sudo nmcli con add type bond con-name bond0 ifname bond0 mode active-backup
sudo nmcli con add type ethernet con-name bond-slave-1 ifname eth0 master bond0
sudo nmcli con add type ethernet con-name bond-slave-2 ifname eth1 master bond0
```

### 4. Configure Interface Teaming Using nmcli

```shell
sudo nmcli con add type team con-name team0 ifname team0 config '{"runner": {"name": "activebackup"}}'
sudo nmcli con add type ethernet con-name team-slave-1 ifname eth2 master team0
sudo nmcli con add type ethernet con-name team-slave-2 ifname eth3 master team0
```

### 5. Configure IPv6 and Perform Basic Troubleshooting

Configure IPv6:

```shell
sudo nmcli con mod <connection-name> ipv6.addr-gen-mode eui64
sudo nmcli con mod <connection-name> ipv6.method auto
```

Troubleshooting:

```shell
ping6 ipv6.google.com
```

### 6. Use firewalld for Packet Filtering

```shell
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload
```

### 7. Use firewalld Zones

```shell
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --reload
```

### 8. Use firewalld for NAT

Masquerading:

```shell
sudo firewall-cmd --zone=public --add-masquerade --permanent
```

### 9. Use firewalld Rich Rules

```shell
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.0.0/24" accept'
sudo firewall-cmd --reload
```

### 10. Route IP Traffic and Create Static Routes

```shell
sudo ip route add <destination-subnet> via <gateway-ip>
```
## Prepare NFS Server with MariaDB

### 1. Install and Configure MariaDB

```shell
sudo yum install -y mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation
```

### 2. Manage SELinux for Database Services

```shell
sudo setsebool -P mysqld_disable_trans 1
```

### 3. Perform Logical Database Backups

```shell
mysqldump -u root -p [database_name] > backup.sql
```

### 4. Create a Database with Tables

```sql
CREATE DATABASE sample_db;
USE sample_db;
CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255), email VARCHAR(255));
```

### 5. Restore Logical Database Backups

```shell
mysql -u root -p [database_name] < backup.sql
```

### 6. Perform Simple SQL Queries

```sql
SELECT * FROM users;
INSERT INTO users (name, email) VALUES ('John Doe', 'john@example.com');
```

### 7. Recover the MariaDB Root Password

Stop the MariaDB service, then start with `--skip-grant-tables`, and reset the password:

```shell
sudo systemctl stop mariadb
sudo mysqld_safe --skip-grant-tables &
mysql -u root
FLUSH PRIVILEGES;
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('new_password');
```

### 8. Set the Root Database Password

If not set during `mysql_secure_installation`, use:

```sql
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('new_password');
```

## Prepare NFS Server with Apache

### 1. Install and Configure Apache

```shell
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
```

### 2. Manage SELinux for Web Services

```shell
sudo setsebool -P httpd_unified 1
```

### 3. Configure a Basic Apache Web Server

Edit `/etc/httpd/conf/httpd.conf`, then restart Apache.

### 4. Configure Access Control on Directories

```apache
<Directory "/var/www/html">
    Require all granted
</Directory>
```

### 5. Configure Private Access Using Basic Auth

```shell
sudo htpasswd -c /etc/httpd/.htpasswd user1
```

Add to your directory configuration:

```apache
<Directory "/var/www/html/private">
    AuthType Basic
    AuthName "Restricted Content"
    AuthUserFile /etc/httpd/.htpasswd
    Require valid-user
</Directory>
```

### 6. Configure Group-Managed Content

Create a group, add users, and set permissions on the content directory.

### 7. Configure Virtual Host

```apache
<VirtualHost *:80>
    ServerName www.example.com
    DocumentRoot /var/www/example
</VirtualHost>
```

### 8. Configure Virtual Host on a Non-Standard Port

```apache
Listen 8080
<VirtualHost *:8080>
    ServerName www.example.com
    DocumentRoot /var/www/example_port
</VirtualHost>
```

### 9. Configure a Secure Virtual Host

```apache
<VirtualHost *:443>
    ServerName www.example.com
    DocumentRoot /var/www/example_ssl
    SSLEngine on
    SSLCertificateFile "/path/to/www.example.com.cert"
    SSLCertificateKeyFile "/path/to/www.example.com.key"
</VirtualHost>
```

### 10. Deploy a Basic CGI Application

Enable CGI module in Apache, place your CGI script in the `cgi-bin` directory.

### 11. Generate Key Pairs and Self-Signed Certificates

```shell
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/pki/tls/private/www.example.com.key -out /etc/pki/tls/certs/www.example.com.cert
```

## Prepare NFS Server with NFS and SAMBA

### 1. Install and Configure NFS

```shell
sudo yum install -y nfs-utils
sudo systemctl enable --now nfs-server
```

### 2. Manage SELinux for NFS Services

```shell
sudo setsebool -P nfs_export_all_rw 1
sudo setsebool -P nfs_export_all_ro 1
```

### 3. Provide Network Share to Specific Clients

Edit `/etc/exports` to add:

```text
/path/to/share  client1(rw,sync,no_subtree_check) client2(ro,sync,no_subtree_check)
```

Then, apply changes:

```shell
sudo exportfs -arv
```

### 4. Mount a Simple NFS Share

On the client:

```shell
sudo mount nfs-server:/path/to/share /mnt/nfs_share
```

### 5. Create an NFS Share for Group Collaboration

Same as above, but ensure directory permissions allow group access.

### 6. Mount an NFS Share for Group Collaboration

Similar to mounting a simple NFS share, ensure client users are part of the appropriate group.

### 7. Install and Configure SAMBA Services

```shell
sudo yum install -y samba samba-client
sudo systemctl enable --now smb nmb
sudo smbpasswd -a username
```

### 8. Manage SELinux for SMB Services

```shell
sudo setsebool -P samba_export_all_rw 1
```

### 9. Create a Simple Public Share

Edit `/etc/samba/smb.conf` to add:

```ini
[public]
   path = /path/to/public
   public = yes
   writable = no
   printable = no
```

Restart Samba:

```shell
sudo systemctl restart smb nmb
```

### 10. Provide Network Shares to Specific Client

Add to `/etc/samba/smb.conf` under share definitions:

```ini
   hosts allow = client1 client2
```

### 11. Automount Using a Credentials File

On the client, edit `/etc/fstab` to include:

```text
//samba-server/public /mnt/samba_share cifs credentials=/path/to/credentials,file_mode=0660,dir_mode=0770 0 0
```

### 12. Provide Network Shares Suitable for Group Collaboration

Configure as public share but with `writable = yes` and appropriate group permissions.


## Prepare NFS Server with Mail and SSH

### 1. Install and Configure Mail Services

```shell
sudo yum install -y postfix
sudo systemctl enable --now postfix
```

### 2. Manage SELinux for SMTP Services

```shell
sudo setsebool -P allow_postfix_local_write_mail_spool 1
```

### 3. Configure a Local Mail Server

Edit `/etc/postfix/main.cf` for local delivery only.

### 4. Create a Null-Client Mail Relay

Configure `/etc/postfix/main.cf` to relay all emails to a central mail server.

### 5. Create a Mail Gateway

Configure Postfix to handle incoming and outgoing mail for multiple domains.

### 6. Install SSH Additional Packages

```shell
sudo yum install -y openssh-clients openssh-server
```

### 7. Configure SSH Client

Edit `/etc/ssh/ssh_config` to customize client settings.

### 8. Configure SSH Server

Edit `/etc/ssh/sshd_config` to set server preferences.

### 9. Configure Per-User Client

Use `~/.ssh/config` for user-specific configurations.

### 10. Configure Key-Based Authentication

```shell
ssh-keygen
ssh-copy-id user@hostname
```

### 11. Configure Host and User-Based Security

Adjust `/etc/ssh/sshd_config` for host-based and user-based restrictions.
