# Easy sssd Active Directory integration on ArchLinux @08.2025

- install packages `sssd` 
- clone, build and install `git clone && makepkg -sci` adcli & realmd from AUR `https://aur.archlinux.org/adcli.git` & `https://aur.archlinux.org/realmd.git`
+ Prepare **/etc/realmd.conf** ([https://www.freedesktop.org/software/realmd/docs/realmd-conf.html](https://www.freedesktop.org/software/realmd/docs/realmd-conf.html)) 
 ```
[users]
default-home = /home/%U
default-shell = /bin/bash

[active-directory]
os-name = Arch Linux
os-version = 9.9.9.9.9
default-client = sssd
# default-client = winbind

[service]
automatic-install = no

[serenity.lan]
automatic-join = no
user-principal = yes
manage-system = no
automatic-id-mapping = no
fully-qualified-names = no
``` 
- Join realm
  - `realm join DOMAIN.TLD -U domain_admin@DOMAIN.TLD`
- Join Domain
  - `adcli join domain.tld -U domain_admin@DOMAIN.TLD`
- realmd already modified /etc/krb5.conf, /etc/pam.d/system-auth, created /etc/sssd/sssd.conf and /etc/pam.d/sssd_arch
- in /etc/sssd.conf `[domain/domain.tld]` section
	- set `use_fully_qualified_names = False` for short names instead of account_name@domain.tld
	- set `ldap_id_mapping = False` if you want to use 'rfc2307' properties, if rfc2307 properties (uid, gid etc.) are not set for user accounts and groups on AD PDC sssd auth wont work with this setting. Leave this as `ldap_id_mapping = True` for easy config
- if you change sssd settings dont forget to clear sssd cache `sssctl cache-remove`
```
$ sudo sssctl cache-remove
SSSD must not be running. Stop SSSD now? (yes/no) [yes] 
Creating backup of local data...
SSSD backup of local data already exists, override? (yes/no) [no] yes
Removing cache files...
SSSD needs to be running. Start SSSD now? (yes/no) [yes]
```
- Now lest test if authentication is working `sssctl user-checks user_name -a auth`
```
$ sssctl user-checks user_name -a auth
user: enes
action: auth
service: system-auth

SSSD nss user lookup result:
 - user name: user_name
 - user id: 10003
 - group id: 984
 - gecos: User Name
 - home directory: /home/user
 - shell: /bin/bash

InfoPipe User lookup with [user] failed.
testing pam_authenticate

Password: 
pam_authenticate for user [user]: Success

PAM Environment:
 - KRB5CCNAME=FILE:/tmp/krb5cc_10003
```
- Or test with pamtester
	build and install pamtester from AUR
	- `pamtester -v login user_name authenticate` (From: https://pamtester.sourceforge.net/)

***Troubleshooting***
- Make sure ***/etc/krb5.conf*** doesnt contain `default_ccache_name = FILE:/run/user/%{uid}/krb5cc` if there is, either delete it or replace it with `default_ccache_name = FILE:/tmp/krb5cc_%{uid}` then remove sss cache before trying again.
- If auth fails with errors in /var/log/sssd/krb5_child.log -> sssd kerberos authentication failing, install ***pam_krb5***, edit ***/etc/pam.d/sssd-arch*** like this (inspired by: https://unix.stackexchange.com/a/545498)
  ```
	auth sufficient pam_krb5.so minimum_uid=1000
	#auth sufficient pam_sss.so forward_pass
	password sufficient pam_sss.so use_authtok
	session required pam_mkhomedir.so skel=/etc/skel/ umask=0077
	session optional pam_sss.so
	```
