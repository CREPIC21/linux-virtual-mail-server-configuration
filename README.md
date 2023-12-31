# Linux Virtual mail Server Configuration
 - VM, SSH, Domain Name, DNS, VPS, WEB, OpenSSH, MySql, Postfix, SMTP AUTH, Dovecot (POP3/IMAP), Amavis, ClamAV, Redis, Rspamd, Pflogsumm 

💡 In the configuration steps replace all occurrences of:
> - `wordpresslinux.xyz` with your chosen domain name
> - `wordpress` with your own chosen word
> - `vm_ip_address` with your VM IP address


## **1. Create VM in Azure or any other cloud provider(Digital Ocean, Linode, AWS...)**
- use Ubuntu Server 20.04 image
- use password for authentication
- once VM is created, connect to it using SSH: 
```
ssh -p 22 <user_name>@<vm_ip_address>
```
- as root run: 
```
apt update && apt full-upgrade -y
```

## **2. Secure SSH with Key Authentication**
- on your local machine(laptop/PC) that you will use to SSH to the VM create key pairs in `/home/wordpress/` directory:
```
ssh-keygen -t rsa -b 2048 -C 'linux server VM keys'
```
- add created public key to your VM machine `home` directory by copy/pasting it to the VM `.ssh/authorized_keys` file:
```
vim .ssh/authorized_keys
```
- disable password authentication on the server(`PasswordAuthentication no`) and restart the SSH service:
```
vim /etc/ssh/sshd_config
```
```
// set PasswordAuthentication to no in the configuration file
PasswordAuthentication no
```
- restart the service:
```
systemctl restart ssh
```
- now you can SSH to the VM without password(you will not be asked for a password anymore): 
```
ssh -p 22 <user_name>@<vm_ip_address>
```


## **3. Getting a Domain name**
- for this example we used [Namecheap](https://ap.www.namecheap.com/) and domain name `wordpresslinux.xyz`
- navigate to: `Domain List -> manage (next to domain wordpresslinux.xyz) -> Advanced DNS` and under `PERSONAL DNS SERVER` add two nameservers:
```
ns1 with VM IP address
ns2 with VM IP address
```
- navigate to the `DOMAIN` tab and under `NAMESERVERS` choose Custom DNS and add created nameservers:
```
ns1.wordpresslinux.xyz
ns2.wordpresslinux.xyz
```
- in can take up to 72 hours for change to take an effect, to check run in terminal:
```
dig -t ns wordpresslinux.xyz
```

## **4. Installing a DNS Server (Bind9)**
- SSH to VM:
```
ssh -p 22 <user_name>@<vm_ip_address>
```
- as root run: 
```
apt update && apt install bind9 bind9utils bind9-doc -y
```
- check if the service is running: 
```
systemctl status bind9
```
- set IPv4 since we are using only IPv4 mode, add `-4` to the end of options parameter in `/etc/default/named` file and restart the service:
```
vim /etc/default/named
```
```
// add `-4` to the end of options parameter in the configuration
OPTIONS="-u bind -4"
```
- restart the service:
```
systemctl restart bind9
```
- testing:
  - sending DNS query to DNS server that is now running on a localhost: `dig -t a @localhost google.com`
  - sending DNS query to DNS server that is public from CloudFlare: `dig -t a @1.1.1.1 google.com`
- add two forwarders to our DNS Server in the `/etc/bind/named.conf.options` file, set public google DNS servers as forwarders for my server and restart the service:
```
vim /etc/bind/named.conf.options
```
- configuration to add
```
forwarders {
      8.8.8.8;
      8.8.4.4;
};
```
- restart the service:
```
systemctl restart bind9
```
- if the server doesn't know the DNS information asked by the clients, it will send a recursive query to those two forwarder servers and wait for a final answer and if it gets the answer, it will pass it to the client that has asked for it in the first place
- testing: 
```
dig -t a @localhost kali.org
```

## **5. Setting up the Authoritative BIND9 DNS Server with an A, MX, SPF and PTR Record**
- open `/etc/bind/named.conf.local` and add the following configuration:
```
vim /etc/bind/named.conf.local
```
- configuration to add
```
zone "wordpresslinux.xyz" {
        type master;
        file "/etc/bind/db.wordpresslinux.xyz";
};
```
- in the above configuration we created a new zone using the zone clause and specifyed that this is the master zone, which means that this is the master of authoritative DNS server for the Domain, the file is db.wordpresslinux.xyz, where we'll add DNS records for the domain
- next step is to create the zone file and instead of creating the zone file from scratch, we will use a zone template file which exists in `/etc/bind` where we will add `A, MX, SPF and PTR` records: https://www.techfry.com/webmaster-tips/dns-records-ns-a-mx-cname-spf
```
cp /etc/bind/db.empty /etc/bind/db.wordpresslinux.xyz
vim /etc/bind/db.wordpresslinux.xyz
```
- configuration to add
```
; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it, edit named.conf, and use that copy.
;
$TTL    86400
@       IN      SOA     ns1.wordpresslinux.xyz. root.localhost. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns1.wordpresslinux.xyz.
ns1     IN      A       139.162.128.80
@       IN      MX      10 mail.wordpresslinux.xyz.
wordpresslinux.xyz.     IN      A       139.162.128.80
www     IN      A       139.162.128.80
mail    IN      A       139.162.128.80
wordpresslinux.xyz. 	IN	TXT	"v=spf1 mx ~all"
```
- before restarting the server, check if there are any syntax errors in the configuration files:
```
named-checkconf
named-checkzone wordpresslinux.xyz /etc/bind/db.wordpresslinux.xyz
``` 
- silent output indicates no errors were found, if there are syntax errors in the zone file, you need to fix them or the zone won't be loaded
- restart the service:
```
systemctl restart bind9
```
- check the service status and if the zone was loaded successfully:
```
systemctl status bind9
```
- always update the SOA serial number when you make changes to a zone file:
```
vim /etc/bind/db.wordpresslinux.xyz
```
- on line 8 we changed SOA serial number from  1 to 2
```
; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it, edit named.conf, and use that copy.
;
$TTL    86400
@       IN      SOA     ns1.wordpresslinux.xyz. root.localhost. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns1.wordpresslinux.xyz.
ns1     IN      A       vm_ip_address
mail    IN      MX 10   mail.wordpresslinux.xyz.
wordpresslinux.xyz.     IN      A vm_ip_address
www     IN      A       vm_ip_address
mail    IN      A       vm_ip_address
```
- to change PTR record to point to our domain name `mail.wordpresslinux.xyz.`, we need to:
  - first check where does it point to: `dig -x 139.162.128.80`
    - in my case it points to Linode `139-162-128-80.ip.linodeusercontent.com.` where I created my VM and all I need to do is manually change the Reverse DNS in the Linode UI `Network -> Edit RDNS`, add `wordpresslinux.xyz` and reboot the VM
    - check the hostname in your terminal by running `hostnamectl` and change it to `wordpresslinux.xyz` using command `hostnamectl set-hostname wordpresslinux.xyz`
  - for other providers maybe you need to:
    - if necessary, modify the `/etc/hosts` file as below by adding your domain `wordpresslinux.xyz`, restart/reboot the VM and test again with following command `dig -x 139.162.128.80`:
```
# /etc/hosts
127.0.0.1       localhost
127.0.1.1       mail.wordpresslinux.xyz

# The following lines are desirable for IPv6 capable hosts
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
```
- testing: 
```
dig -t a wordpresslinux.xyz
dig -t mx wordpresslinux.xyz
dig -t txt wordpresslinux.xyz
dig -x 139.162.128.80 -> checking PTR record
dig @localhost -t ns wordpresslinux.xyz
dig @localhost -t a www.wordpresslinux.xyz
```

## **6. Installing a Web Server(Apache2)**
```
apt update && apt install apache2 -y
systemctl status apache2
systemctl enable apache2
```
- if ufw is active run:
```
ufw status
ufw allow 'Apache Full'
```
- testing
  - open web browser and go to the VM IP address, trick to get IP address from terminal: `curl -4 ident.me`
  - open web browser and go to the VM domain name `wordpresslinux.xyz`

## **7. Securing Apache with OpenSSH and Digital Certificates**
```
apt update && apt install certbot python3-certbot-apache -y
certbot -d wordpresslinux.xyz
certbot -d www.wordpresslinux.xyz
systemctl status certbot.timer
certbot renew --dry-run
```

## **8. Installing and securing MySQL Server**
```
apt update && apt install mysql-server -y
```
- once the installation is complete, the MySQL server will start automatically:
```
systemctl status mysql
```
- the deamon process that’s running is called `mysqld`, you can check it by running: 
```
ps -ef | grep mysql
```
- MySQL is not very secure so it’s recommended to run a security script that comes pre-installed with MySQL, the script will remove some insecure default settings and lock down access to the database server by removing some MySQL accounts and setting the admin password, run command:
```
mysql_secure_installation
```
- in case of error when providing the password see this article: https://devanswe.rs/how-to-fix-failed-error-set-password-has-no-significance-for-user-rootlocalhost/?utm_content=cmp-true
- once completed login to mysql: 
```
mysql -u root -p
```

## **9. Installing Software Packages**
```
apt update && apt install postfix postfix-mysql postfix-doc dovecot-common dovecot-imapd dovecot-pop3d libsasl2-2 libsasl2-modules libsasl2-modules-sql sasl2-bin libpam-mysql mailutils dovecot-mysql dovecot-sieve dovecot-managesieved
```
### IMPORTANT ###
- when prompted, select `Internet site` as the type of the mail server the postfix installer should configure
- the system mail name should be the fully qualified domain name, which is in my case `mail.wordpresslinux.xyz`
- all installed services are automatically started and enabled at boot time

## **10. Configuring MySql and Connect it with Postfix**
- connect to the MySql and create a database that will store virtual domains, users(and their passwords) and virtual aliases or forwarding table to specify all the emails that are going to be forwarded to other emails
```
mysql -u root -p
mysql> CREATE DATABASE mail;
mysql> USE mail;
mysql> CREATE USER 'mail_admin'@'localhost' IDENTIFIED BY 'mail_admin_password';  
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON mail.* TO 'mail_admin'@'localhost';
mysql> FLUSH PRIVILEGES;
mysql> CREATE TABLE domains (domain varchar(50) NOT NULL, PRIMARY KEY (domain));
mysql> CREATE TABLE users (email varchar(80) NOT NULL, password varchar(128) NOT NULL, PRIMARY KEY (email));
mysql> CREATE TABLE forwardings (source varchar(80) NOT NULL, destination TEXT NOT NULL, PRIMARY KEY (source));
mysql> exit
```
- creating files that will be included in the main.cf, the Postfix configuration file, to tell Postfix how to connect to MySQL:

A) this file indicate Postfix how to get the domains from the mysql database 
```
vim /etc/postfix/mysql_virtual_domains.cf
```
```
user = mail_admin
password = mail_admin_password
dbname = mail
query = SELECT domain FROM domains WHERE domain='%s'
hosts = 127.0.0.1
```

B) this file will tell Postfix how to access the mail aliases which are in the forwarding table 
```
vim /etc/postfix/mysql_virtual_forwardings.cf
```
```
user = mail_admin
password = mail_admin_password
dbname = mail
query = SELECT destination FROM forwardings WHERE source='%s'
hosts = 127.0.0.1
```

C) creating virtual mailbox configuration file for Postfix 
```
vim /etc/postfix/mysql_virtual_mailboxes.cf
```
```
user = mail_admin
password = mail_admin_password
dbname = mail
query = SELECT CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/') FROM users WHERE email='%s'
hosts = 127.0.0.1
```

D) creating virtual e-mail mapping file
```
vim /etc/postfix/mysql_virtual_email2email.cf
```
```
user = mail_admin
password = mail_admin_password
dbname = mail
query = SELECT email FROM users WHERE email='%s'
hosts = 127.0.0.1
```
- removing all the permissions for the others so that only the owner and the group can read the files and changing the group owner of the files to postfix, the group under which it runs
```
chmod o-rwx /etc/postfix/mysql_virtual_*
chown root.postfix /etc/postfix/mysql_virtual_*
```
- since the e-mail users are virtual and not system users, because they don't actually exist in `/etc/password`, I need a system user for mail handling, so I'm creating a new user and group called "vmail" for this purpose where I'm setting the group and user ID statically to be 5000, its home directory will be created as well as `/var/vmail`, all the emails will be stored there
```
groupadd -g 5000 vmail
useradd -g vmail -u 5000 -d /var/vmail -m vmail
```
## **11. Configuring Postfix**
```
postconf -e "myhostname = mail.wordpresslinux.xyz"
postconf -e "mydestination = mail.wordpresslinux.xyz, localhost, localhost.localdomain"
postconf -e "mynetworks = 127.0.0.0/8"
postconf -e "message_size_limit = 31457280"
postconf -e "virtual_alias_domains ="
postconf -e "virtual_alias_maps = proxy:mysql:/etc/postfix/mysql_virtual_forwardings.cf, mysql:/etc/postfix/mysql_virtual_email2email.cf"
postconf -e "virtual_mailbox_domains = proxy:mysql:/etc/postfix/mysql_virtual_domains.cf"
postconf -e "virtual_mailbox_maps = proxy:mysql:/etc/postfix/mysql_virtual_mailboxes.cf"
postconf -e "virtual_mailbox_base = /var/vmail"
postconf -e "virtual_uid_maps = static:5000"
postconf -e "virtual_gid_maps = static:5000"
postconf -e "smtpd_sasl_auth_enable = yes"
postconf -e "broken_sasl_auth_clients = yes"
postconf -e "smtpd_sasl_authenticated_header = yes"
postconf -e "smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination"
postconf -e "smtpd_use_tls = yes"
postconf -e "smtpd_tls_cert_file = /etc/letsencrypt/live/wordpresslinux.xyz/fullchain.pem"
postconf -e "smtpd_tls_key_file = /etc/letsencrypt/live/wordpresslinux.xyz/privkey.pem"
postconf -e "virtual_transport=dovecot"
postconf -e 'proxy_read_maps = $local_recipient_maps $mydestination $virtual_alias_maps $virtual_alias_domains $virtual_mailbox_maps $virtual_mailbox_domains $relay_recipient_maps $relay_domains $canonical_maps $sender_canonical_maps $recipient_canonical_maps $relocated_maps $transport_maps $mynetworks $virtual_mailbox_limit_maps'
```
- postfix configuration parameters: https://www.postfix.org/postconf.5.html

## **12. Configuring SMTP AUTH (SASLAUTHD with PAM and MySql)**
A) Creating a directory where saslauthd will save its information:  
```
mkdir -p /var/spool/postfix/var/run/saslauthd
```

B) Editing the configuration file of saslauthd: 
```
vim /etc/default/saslauthd
```
```
START=yes
DESC="SASL Authentication Daemon"
NAME="saslauthd"
MECHANISMS="pam"
MECH_OPTIONS=""
THREADS=5
OPTIONS="-c -m /var/spool/postfix/var/run/saslauthd -r"
```

C) Creating a new PAM file: 
- Pam, which means pluggable authentication modules, provides authentication for saslauthd; it practically says how to access the MySQL backend
```
vim /etc/pam.d/smtp
```
```
auth required pam_mysql.so user=mail_admin passwd=mail_admin_password host=127.0.0.1 db=mail table=users usercolumn=email passwdcolumn=password crypt=3
account sufficient pam_mysql.so user=mail_admin passwd=mail_admin_password host=127.0.0.1 db=mail table=users usercolumn=email passwdcolumn=password crypt=3
```

D) last required file smtpd.conf
```
vim /etc/postfix/sasl/smtpd.conf
```
```
pwcheck_method: saslauthd 
mech_list: plain login 
log_level: 4
```

E) Setting the permissions
```
chmod o-rwx /etc/pam.d/smtp
chmod o-rwx /etc/postfix/sasl/smtpd.conf
```
F) Adding the postfix user to the sasl group for group access permissions: 
```
usermod  -aG sasl postfix
```

G) Restarting the services:
```
systemctl restart postfix
systemctl restart saslauthd
```
- SASL reference: https://www.rfc-editor.org/rfc/rfc4422

## **13. Configuring Dovecot (POP3/IMAP)**
A) At the end of `/etc/postfix/master.cf` add:
```
dovecot   unix  -       n       n       -       -       pipe
    flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -d ${recipient}
```

B) Edit Dovecot config file, erase everything and add the below configuration(in real world scenario make a copy of the original file instead of erasing it): 
```
vim /etc/dovecot/dovecot.conf
```
```
log_timestamp = "%Y-%m-%d %H:%M:%S "
mail_location = maildir:/var/vmail/%d/%n/Maildir
managesieve_notify_capability = mailto
managesieve_sieve_capability = fileinto reject envelope encoded-character vacation subaddress comparator-i;ascii-numeric relational regex imap4flags copy include variables body enotify environment mailbox date
namespace {
  inbox = yes
  location = 
  prefix = INBOX.
  separator = .
  type = private
}
passdb {
  args = /etc/dovecot/dovecot-sql.conf
  driver = sql
}
protocols = imap pop3

service auth {
  unix_listener /var/spool/postfix/private/auth {
    group = postfix
    mode = 0660
    user = postfix
  }
  unix_listener auth-master {
    mode = 0600
    user = vmail
  }
  user = root
}

userdb {
  args = uid=5000 gid=5000 home=/var/vmail/%d/%n allow_all_users=yes
  driver = static
}

protocol lda {
  auth_socket_path = /var/run/dovecot/auth-master
  log_path = /var/vmail/dovecot-deliver.log
  mail_plugins = sieve
  postmaster_address = postmaster@example.com
}

protocol pop3 {
  pop3_uidl_format = %08Xu%08Xv
}

service stats {
  unix_listener stats-reader {
    user = dovecot
    group = vmail
    mode = 0660
  }
  unix_listener stats-writer {
    user = dovecot
    group = vmail
    mode = 0660
  }
}

ssl = yes
ssl_cert = </etc/letsencrypt/live/wordpresslinux.xyz/fullchain.pem
ssl_key = </etc/letsencrypt/live/wordpresslinux.xyz/privkey.pem
```

C) creating a file for Dovecot to use MySql to extract the users and their passwords
```
vim /etc/dovecot/dovecot-sql.conf
```
```
driver = mysql
connect = host=127.0.0.1 dbname=mail user=mail_admin password=mail_admin_password
default_pass_scheme = PLAIN-MD5
password_query = SELECT email as user, password FROM users WHERE email='%u';
```
D) Restart Dovecot
```
systemctl restart dovecot
```

## **14. Adding Domains and Virtual Users, Testing The System**
```
mysql -u root
msyql>USE mail;
mysql>INSERT INTO domains (domain) VALUES ('wordpresslinux.xyz');
mysql>insert into users(email,password) values('u1@wordpresslinux.xyz', md5('pass123'));
mysql>insert into users(email,password) values('u2@wordpresslinux.xyz', md5('pass123'));
mysql>insert into users(email,password) values('u3@wordpresslinux.xyz', md5('pass123'));
mysql>quit;
```
- testing:
  - `testsaslauthd -u u1@wordpresslinux.xyz -p pass123 -f /var/spool/postfix/var/run/saslauthd/mux -s smtp`
  - set up a mail client like Mozilla Thunderbird, send and receive mail to both local and external accounts

- troubleshooting:
  - ping the mail server: `ping mail.wordpresslinux.xyz`
  - test if port 25 is open on a mail server from another machine: `nmap -p 25 mail.wordpresslinux.xyz -Pn` 
  - test if port 465 is open on a mail server from another machine: `nmap -p 465 mail.wordpresslinux.xyz -Pn` 
  - test the ports from Windows by using telnet: `telnet mail.wordpresslinux.xyz 25`
  - check the logs in real time: `tail -f /var/log/mail.log`
    - to increase the logging level open `/etc/postfix/master.cf` and add the `-v` at the end of the line that starts with `SMTP`
    - restart the service: `systemctl restart postfix`
  - check for authentication issues: `tail -f /var/log/auth.log`
    - to see more details regarding authentication issues and to see how it connects to the MySQL backend and how it is running the select SQL statement add `debug` word option at the end of the each line of `/etc/pam.d/smtp` file
    - restart the service: `systemctl restart saslauthd`
  - check open/listening ports on the VM: `netstat -tupan` or `netstat -tupan | grep master`
  - check port 25 on a mail server of a specific domain: 
- enable SMTPS Port 465(because it's SMTPS or SMTP over SSL port) in Postfix for email submission by uncommenting `smtps     inet  n       -       y       -       -       smtpd` line and restart the service:
```
vim /etc/postfix/master.cf
```
```
# Postfix master process configuration file.  For details on the format
# of the file, see the master(5) manual page (command: "man 5 master" or
# on-line: http://www.postfix.org/master.5.html).
#
# Do not forget to execute "postfix reload" after editing this file.
#
# ==========================================================================
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (no)    (never) (100)
# ==========================================================================
smtp      inet  n       -       y       -       -       smtpd
#smtp      inet  n       -       y       -       1       postscreen
#smtpd     pass  -       -       y       -       -       smtpd
#dnsblog   unix  -       -       y       -       0       dnsblog
#tlsproxy  unix  -       -       y       -       0       tlsproxy
#submission inet n       -       y       -       -       smtpd
#  -o syslog_name=postfix/submission
#  -o smtpd_tls_security_level=encrypt
#  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_tls_auth_only=yes
#  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING
smtps     inet  n       -       y       -       -       smtpd
#  -o syslog_name=postfix/smtps
#  -o smtpd_tls_wrappermode=yes
#  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING

```
```
systemctl restart postfix
```
## **15. Virus Scanning Using Amavis and ClamAV**
A) Installing Amavis
```
apt update && apt install amavisd-new
```
**Note: if there's an error set `$myhostname` in `/etc/amavis/conf.d/05-node_id`**

B) Installing required packages for scanning attachments
```
apt install arj bzip2 cabextract cpio rpm2cpio file gzip lhasa nomarch pax rar unrar p7zip-full unzip zip lrzip lzip liblz4-tool lzop unrar-free
```

C) Configuring Postfix (`/etc/postfix/main.cf`)
```
postconf -e 'content_filter = smtp-amavis:[127.0.0.1]:10024'
postconf -e 'smtpd_proxy_options = speed_adjust'
```

D) Add to the end of `/etc/postfix/master.cf`:
```
smtp-amavis   unix   -   -   n   -   2   smtp
    -o syslog_name=postfix/amavis
    -o smtp_data_done_timeout=1200
    -o smtp_send_xforward_command=yes
    -o disable_dns_lookups=yes
    -o max_use=20
    -o smtp_tls_security_level=none


127.0.0.1:10025   inet   n    -     n     -     -    smtpd
    -o syslog_name=postfix/10025
    -o content_filter=
    -o mynetworks_style=host
    -o mynetworks=127.0.0.0/8
    -o local_recipient_maps=
    -o relay_recipient_maps=
    -o strict_rfc821_envelopes=yes
    -o smtp_tls_security_level=none
    -o smtpd_tls_security_level=none
    -o smtpd_restriction_classes=
    -o smtpd_delay_reject=no
    -o smtpd_client_restrictions=permit_mynetworks,reject
    -o smtpd_helo_restrictions=
    -o smtpd_sender_restrictions=
    -o smtpd_recipient_restrictions=permit_mynetworks,reject
    -o smtpd_end_of_data_restrictions=
    -o smtpd_error_sleep_time=0
    -o smtpd_soft_error_limit=1001
    -o smtpd_hard_error_limit=1000
    -o smtpd_client_connection_count_limit=0
    -o smtpd_client_connection_rate_limit=0
    -o receive_override_options=no_header_body_checks,no_unknown_recipient_checks,no_address_mappings
```

E) Installing ClamAV
```
apt install clamav clamav-daemon
```

F) Turning on virus-checking in Amavis.
- in `/etc/amavis/conf.d/15-content_filter_mode` uncomment:
```
@bypass_virus_checks_maps = (
  	\%bypass_virus_checks, \@bypass_virus_checks_acl, \$bypass_virus_checks_re);
```
- the communication between Amavis and ClamAV will be done through a socket file and we need to add the user clamav to the amavis group so that they have the necessary permissions on the socket file
```
usermod -aG amavis clamav
```

G) Restarting Amavis and ClamAv
```
systemctl restart amavis
systemctl restart clamav-daemon
```

H) To check that Amavis communicates with ClamAV run:
```
journalctl -eu amavis
```

I) Testing Amavis and ClamAV
```
tail -f /var/log/mail.log
journalctl -eu amavis
```
- download test virus and send it to via email to check if antivirus detection is working
  - https://www.eicar.org/download-anti-malware-testfile/

## **16. Fighting Against Spam: Postfix Access Restrictions**
- http://www.postfix.org/postconf.5.html

A) Fighting Against Spam: Postfix HELO Restrictions
```
vim /etc/postfix/main.cf
```
```
smtpd_helo_required = yes
smtpd_helo_restrictions =
 permit_mynetworks,
 permit_sasl_authenticated,
 reject_invalid_helo_hostname,
 reject_non_fqdn_helo_hostname,
 reject_unknown_helo_hostname,
 permit
```
B) Fighting Against Spam: Postfix Sender Restrictions
```
vim /etc/postfix/main.cf
```
```
smtpd_sender_restrictions =
 permit_mynetworks,
 permit_sasl_authenticated,
 reject_unknown_sender_domain,
 reject_non_fqdn_sender,
 reject_unknown_reverse_client_hostname,
 reject_unknown_client_hostname,
 permit
```
C) Fighting Against Spam: Postfix Recipient Restrictions
```
vim /etc/postfix/main.cf
```
```
smtpd_recipient_restrictions =
 permit_mynetworks,
 permit_sasl_authenticated,
 reject_unauth_destination,
 reject_unauth_pipelining,
 reject_unknown_recipient_domain,
 reject_non_fqdn_recipient,
 permit
```
D) Fighting Against Spam: Using Public RBLs
- https://www.debouncer.com/
```
vim /etc/postfix/main.cf
```
```
smtpd_recipient_restrictions =
 permit_mynetworks,
 permit_sasl_authenticated,
 reject_unauth_destination,
 reject_unauth_pipelining,
 reject_unknown_recipient_domain,
 reject_non_fqdn_recipient,
 check_client_access hash:/etc/postfix/rbl_override, # file for checking the whitelist
 reject_rhsbl_helo dbl.spamhaus.org,
 reject_rhsbl_reverse_client dbl.spamhaus.org,
 reject_rhsbl_sender dbl.spamhaus.org,
 reject_rbl_client zen.spamhaus.org,
 permit
```
- sometimes, there are legitimate mail servers blacklisted, let's create a whitelist so they won't be blocked, in the file write on each line a white listed domain that will be allowed to send emails even though it's blacklisted in a public RBL
- create a new file `/etc/postfix/rbl_override` and add:
```
gooddomain.com OK
ilikeit.com    OK
```
- Postfix will not use the text file that we just created, but a special lookup file that is created using the postmap command: 
```
postmap /etc/postfix/rbl_override
```
```
systemctl restart postfix
```

## **17. Installing RSPAMD and POSTFIX integration**
### All commands are run as root 

A) Installing Redis as storage for non-volatile data and as a cache for volatile data
```
apt update && apt install redis-server
```

B) Adding the repository GPG key 
```
wget -O- https://rspamd.com/apt-stable/gpg.key | sudo apt-key add -
```

C) Enabling the Rspamd repository
```
echo "deb http://rspamd.com/apt-stable/ $(lsb_release -cs) main" | sudo tee -a /etc/apt/sources.list.d/rspamd.list
```

D) Installing Rspamd
```
apt update && apt install rspamd
```
- check if the rspamd processes are running
```
pgrep -a rspamd
```

E) Configuring the Rspamd normal worker to listen only on localhost interface
```
vim /etc/rspamd/local.d/worker-normal.inc
```
```
bind_socket = "127.0.0.1:11333";
```
F) Enabling the milter protocol to communicate with postfix:
```
vim /etc/rspamd/local.d/worker-proxy.inc
```
```
 bind_socket = "127.0.0.1:11332";
    milter = yes;
    timeout = 120s;
    upstream "local" {
    default = yes;
    self_scan = yes;
    }
```

G) Configure postfix to use Rspamd
- https://rspamd.com/doc/quickstart.html
```
postconf -e "milter_protocol = 6"
postconf -e "milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}"
postconf -e "milter_default_action = accept"
postconf -e "smtpd_milters = inet:127.0.0.1:11332"
postconf -e "non_smtpd_milters = inet:127.0.0.1:11332"
```

H) Restarting Rspamd and Postfix
```
systemctl restart rspamd
systemctl restart postfix
```

I) Configuring and Testing Rspamd
- create a new file with points that will be awarded to email attributes, such as `SUBJECT_HAS_CURRENCY` attribute:
```
vim /etc/rspamd/local.d/groups.conf

symbols {
        "SUBJECT_HAS_CURRENCY"{
                weight = 25.0;
}
}
```
- restart the service:
```
systemctl restart rspamd
```
- check the email scores which determine if the email will be rejected:
```
cat /etc/rspamd/actions.conf
```
- send a test email to your user from gmail which will be rejected

## **18. Postfix Log Monitoring Using pflogsumm**
```
apt install pflogsumm
```
```
pflogsumm -d today /var/log/mail.log
pflogsumm -d today /var/log/mail.log --problems-first
pflogsumm -d today /var/log/mail.log --rej-add-from --verbose-msg-detail
pflogsumm -d today /var/log/mail.log --rej-add-from --verbose-msg-detail | less
pflogsumm -d today /var/log/mail.log --rej-add-from --verbose-msg-detail | mail -s "Postfix log summary for today" u3@wordpresslinux.xyz
```





















