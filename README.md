# Easy sssd Active Directory integration on ArchLinux @08.2025

- install packages `sssd samba pam-krb5` 
- clone, build and install `git clone && makepkg -sci` adcli & realmd from AUR `https://aur.archlinux.org/adcli.git` & `https://aur.archlinux.org/realmd.git`
- Join realm
  - `realm join DOMAIN.TLD -U domain_admin@DOMAIN.TLD`
+ Prepare smb.conf
**/etc/samba/smb.conf** (From: https://wiki.archlinux.org/title/User:Tbw/Active_Directory_Integration_With_SSSD)
```
[global]
	workgroup = DOMAIN
	password server = pdc.domain.tld
	realm = DOMAIN.TLD
	security = ads
	server string = %h ArchLinux Host
``` 
- Join Domain
  - `net ads join -U domain_admin@DOMAIN.TLD`
- realmd already modified /etc/krb5.conf, created /etc/sssd/sssd.conf and /etc/pam.d/sssd_arch
- in /etc/sssd.conf `[domain/domain.tld]` section
	- set `use_fully_qualified_names = False` for short names instead of account_name@domain.tld
	- set `ldap_id_mapping = False` if you want to use 'rfc2307' properties, if rfc2307 properties (uid, gid etc.) are not set for user accounts and groups on AD PDC sssd auth wont work with this setting. Leave this as `ldap_id_mapping = True` for easy config
- if you change sssd settings dont forget to clear sssd cache
	`sssctl cache-remove`
- Now lest test if authentication is working
	- `sssctl user-checks user_name -a auth`
- Or test with pamtester
	build and install pamtester from AUR
	- `pamtester -v login user_name authenticate` (From: https://pamtester.sourceforge.net/)
	
- If auth fails with errors in /var/log/sssd/krb5_child.log -> sssd kerberos authentication failing, edit /etc/pam.d/sssd-arch like this (inspired by: https://unix.stackexchange.com/a/545498)
**/etc/pam.d/sssd-arch**
  ```
	auth sufficient pam_krb5.so minimum_uid=1000
	#auth sufficient pam_sss.so forward_pass
	password sufficient pam_sss.so use_authtok
	session required pam_mkhomedir.so skel=/etc/skel/ umask=0077
	session optional pam_sss.so
	```
