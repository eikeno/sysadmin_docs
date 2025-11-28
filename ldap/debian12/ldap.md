
This page is mostly following examples based on this series of Videos:

- [Getting Familiar with OpenLDAP, by Rajesh Rajasekharan](https://www.youtube.com/playlist?list=PLfO6SFqcY2PrDR5yct96n4qfgMmh6g0eP)


# Openldap on Debian 12

- Prepare a lab on single VM: Generic VM, 2 CPU 2G RAM will do just fine. 
- Perform minimal install with [netinst ISO](https://cdimage.debian.org/cdimage/archive/12.0.0/amd64/iso-cd/), only select *SSH Server* and *Standard system utilities* when asked and remove Desktop environment, not needed here.
- During install, define host: d12sa (or anything else)  and domain: homelab.local - These values will be used in this document instead of *fedji.com* used in the YT Videos:

- **org**: Homelab
- **domain**: homelab.local

**Goal**: adapt steps shown in the Videos on a Debian 12 system, instead of Solaris, show adapted syntax, and add a few notes when relevant. It is suggested to watch the Videos first, then use this document as companion when trying examples on Debian.

## Memo
### Ports

- **389**   Standard, non encrypted port, with StartTLS (a new port is negotiated in that case)
- **636**   Port used with ldap over SSL/TLS  (LDAPS)

### Core attributes meaning
- **cn** is the common name.
- **sn** is the surname.
- **o**   specifies organization or company name.
- **ou** specifies the organizational unit or department name.
- **dc** specifies the domain component.
- **l**    specifies the locality or city of the organization.
- **st**  specifies state or province.
- **c**   specifies the two-letter ISO code for the country.

To see all attributes of the **core** schema, you can check */etc/ldap/slapd.d/cn=config/cn=schema/cn={0}core.ldif* where it is defined.

### Concepts

- [DIT / DSE](https://ldap.com/dit-and-the-ldap-root-dse/)
- DSA: Directory System Agent.

## Installation
```
apt update
apt install slapd ldap-utils
```

This will ask to define LDAP **admin** password. Store it for later use.


## Configuring OpenLDAP
```
dpkg-reconfigure slapd
```

Answer the configuration questions as follows:


- *Omit OpenLDAP server configuration?* **No**
- *DNS domain name:* **homelab.local**
- *Enter your organization's name* **Homelab**
- *Administrator password:* **Same as defined during installation, or a new one**
- *Do you want the database to be removed when slapd is purged?* **No**
- *Move old database?* **Yes**

After reconfiguring **slapd**, you should have a basic LDAP structure in place.

To be noted the configuration is generated under */etc/ldap/slapd.d/* using the **olc** format, replacing *ldap.conf* or *slapd.conf* files. Check [here](https://www.zytrax.com/books/ldap/ch6/slapd-config.html) for more details on **olc** format and its advantages. This must not be hand edited, **ldapmodify** must be used instead.

Thanks to *dpkg-reconfigure slapd* You don'y need to insert initial record for organization in the directory - it's been done for us. See result of
*ldapsearch -x -LLL -H ldap:/// -b dc=homelab,dc=local objectclass=* below.


## Testing the Configuration - ldapsearch syntax

To ensure everything is working, attempt a search on the base DN of your LDAP directory:

```ldif
# ldapsearch -x -LLL -H ldap:/// -b dc=homelab,dc=local dn
dn: dc=homelab,dc=local
```

Memo:

- **-b**    base __dn__ where to start search
- **-LLL**  ldif format specification, see manpage
- **-x**    use simple authentication instead of SASL
- **-H**    ldap URI

If configured correctly, this should return the base DN and any objects under it.

Other example:
```ldif
# ldapsearch -x -LLL -H ldap:/// -b dc=homelab,dc=local objectclass=*
dn: dc=homelab,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: Homelab
dc: homelab
```





## Adding data: organizationalUnit

002-addFinanceOU.ldif
```ldif
dn: ou=finance,dc=homelab,dc=local
objectClass: organizationalUnit
ou: finance
```

```shell
# ldapadd -x  -H ldap:/// -D cn=admin,dc=homelab,dc=local -w secret -f 002-addFinanceOU.ldif 
adding new entry "ou=finance,dc=homelab,dc=local"
```

You can check the result:
```ldif
# ldapsearch -x -LLL -H ldap:/// -b dc=homelab,dc=local ou=finance
dn: ou=finance,dc=homelab,dc=local
objectClass: organizationalUnit
ou: finance
```
or with 
```ldif
# ldapsearch -x -LLL -H ldap:/// -b dc=homelab,dc=local objectClass=organizationalUnit
dn: ou=finance,dc=homelab,dc=local
objectClass: organizationalUnit
ou: finance
```

Now add another **ou** eng with:

003-addEngOU.ldif
```ldif
dn: ou=eng,dc=homelab,dc=local
objectClass: organizationalUnit
ou: eng
```






## Adding data: persons

Add a user named **Sam Carter** in the **finance** ou:

004-addSamCarterUSER.ldif
```ldif
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
objectClass: person
sn: Carter
cn: Sam Carter
```

```shell
# ldapadd -x  -H ldap:/// -D cn=admin,dc=homelab,dc=local -w secret -f 004-addSamCarterUSER.ldif
adding new entry "cn=Sam Carter,ou=finance,dc=homelab,dc=local"
```

Add users named **Harry Miller** and  **John doe**in the **eng** ou:

005-addEngUSERS.ldif
```ldif
dn: cn=Harry Miller,ou=eng,dc=homelab,dc=local
objectClass: person
sn: Miller
cn: Harry Miller

dn: cn=John Doe,ou=eng,dc=homelab,dc=local
objectClass: person
sn: Doe
cn: John Doe
```

```shell
# ldapadd -x  -H ldap:/// -D cn=admin,dc=homelab,dc=local -w secret -f 005-addEngUSERS.ldif
adding new entry "cn=Harry Miller,ou=eng,dc=homelab,dc=local"

adding new entry "cn=John Doe,ou=eng,dc=homelab,dc=local"
```


## Modify entries

### Adding a telephoneNumber

#### In interactive mode
```ldif
# ldapmodify -x  -H ldap:/// -D cn=admin,dc=homelab,dc=local -w secret
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
changetype: modify
add: telephoneNumber
telephoneNumber: +33 1 12 34 56 78

modifying entry "cn=Sam Carter,ou=finance,dc=homelab,dc=local"


^C
```
Use Ctrl-C to end interactive session, or keep adding commands.

You can check the change was applied:
```ldif
# ldapsearch -x -LLL -H ldap:/// -b ou=finance,dc=homelab,dc=local cn=*
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
objectClass: person
sn: Carter
cn: Sam Carter
telephoneNumber: +33 1 12 34 56 78
```

#### Using an LDIF file

006-modifySamCarterTEL.ldif:
```ldif
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
changetype: modify
replace: telephoneNumber
telephoneNumber: +33 1 23 23 23
```

Apply with:
```shell
# ldapmodify -x  -H ldap:/// -D cn=admin,dc=homelab,dc=local -w secret -f 006-modifySamCarterTEL.ldif
modifying entry "cn=Sam Carter,ou=finance,dc=homelab,dc=local"
```

Verify with:
```ldif
# ldapsearch -x -LLL -H ldap:/// -b ou=finance,dc=homelab,dc=local cn=*
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
objectClass: person
sn: Carter
cn: Sam Carter
telephoneNumber: +33 1 23 23 23
```

### Adding a userPassword

```ldif
# ldapmodify -x  -H ldap:/// -D cn=admin,dc=homelab,dc=local -w secret
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
changetype: modify
add: userPassword
userPassword: megasecret

modifying entry "cn=Sam Carter,ou=finance,dc=homelab,dc=local"
```

Check with:
```ldif
# ldapsearch -x -LLL -H ldap:/// -D cn=admin,dc=homelab,dc=local -w foobar1977 -b ou=finance,dc=homelab,dc=local  userpassword=*
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
objectClass: person
sn: Carter
cn: Sam Carter
telephoneNumber: +33 1 23 23 23
userPassword:: bWVnYXNlY3JldA==
```

**Note**: Here we're using **-D** and **-w**, otherwise the restricted attribute **userPassword** wouldn't be listed to do policy and/or ACL restrictions.

check password:
```shell
# echo bWVnYXNlY3JldA==  | base64 -d - ; echo
megasecret
```

To use **Sam Carter** credentials to perform the search use a syntax like:
```shell
ldapsearch -x -H ldap:/// -D cn="Sam Carter,ou=finance,dc=homelab,dc=local" -w megasecret -b ou=finance,dc=homelab,dc=local  objectClass=*
```

#### Modify password to be hashed

Create a password hash with the **slappasswd** command:

```shell
# slappasswd 
New password: (type foobar)
Re-enter new password: (type foobar)
{SSHA}d9Yi/lvvTDi+CQbCXmWn+4KwLZypxATR
```

007-modifySamCarterPASSWORD.ldif:
```ldif
# setting password to foobar
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
changetype: modify
replace: userPassword
userPassword: {SSHA}d9Yi/lvvTDi+CQbCXmWn+4KwLZypxATR
```

Apply with:
```shell
# ldapmodify -x  -H ldap:/// -D cn=admin,dc=homelab,dc=local -w secret -f 007-modifySamCarterPASSWORD.ldif:
modifying entry "cn=Sam Carter,ou=finance,dc=homelab,dc=local"
```

**Note**: Beware of possible trailing spaces at the end of  userPassword line in the LDIF file, that would cause errors.

Check:
```ldif
# ldapwhoami -x -H ldap:/// -D cn="Sam Carter,ou=finance,dc=homelab,dc=local" -w foobar
dn:cn=Sam Carter,ou=finance,dc=homelab,dc=local
```

This confirms that the operation works, the password is no longer stored in clear text, and works when providing clear text password.


## Dealing with the cn=config access

Starting with video #7 in the playlist: **OpenLDAP Online Configuration (OLC)** it is shown how to define a password for **cn=config** suffix by modifying **slapd.conf**, however on recent Debian this file doesn't exist and the post-installation script defines password for the DN **cn=admin,dc=homelab,dc=local** but not for the DN **cn=config**. One way to overcome this is:

To change the password use ldapmodify as root. 

rootpw_cnconfig.ldif:
```ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: foobar123
```

- change *olcDatabase={0}mdb*  by what's relevant for your setup (check content of */etc/ldap/slapd.d/cn=config/* first)
- use a better password

You can apply as root user, with:

```shell
# ldapmodify -Y EXTERNAL -H ldapi:/// -f rootpw_cnconfig.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}mdb,cn=config"
```

You should now be able to use examples on 'cn=config', for example with:
```
ldapsearch -x -LLL -H ldap:/// -D cn=admin,cn=config  -w foobar123 -b cn=config
```

This page is mostly following examples based on this series of Videos:

- [Getting Familiar with OpenLDAP, by Rajesh Rajasekharan](https://www.youtube.com/playlist?list=PLfO6SFqcY2PrDR5yct96n4qfgMmh6g0eP)

## [Adding an overlay](ldap_overlay)

## Making the OpenLDAP Online Configuration (OLC) Permanent

Related to Video #11

The Debian setup already created **/etc/ldap/slapd.d**  with correct **cn=config** related entries, so we have nothing more to do to achieve permanent OLC, as it's already implemented.

## [Acess Control](ldap_access_control)

## [Password Policy](ldap_password_policy)



## Tips and Tricks 

### Check loaded modules
```shell
# slapcat -n 0 | grep olcModuleLoad
olcModuleLoad: {0}back_mdb
olcModuleLoad: {1}accesslog
```

### Check loaded schemas
```shell
ldapsearch -x -D cn=admin,cn=config -w foobar123 -b cn=schema,cn=config objectClass=* dn
```

### Validate a user password
```ldif
# ldapwhoami -D cn="Sam Carter",ou=finance,dc=homelab,dc=local -W 
Enter LDAP Password: 
dn:cn=Sam Carter,ou=finance,dc=homelab,dc=local
```


## Queries examples

List entries starting at **finance** ou:
```ldif
# ldapsearch -x -LLL -H ldap:/// -b ou=finance,dc=homelab,dc=local
dn: ou=finance,dc=homelab,dc=local
objectClass: organizationalUnit
ou: finance

dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
objectClass: person
sn: Carter
cn: Sam Carter
```

Same as above but filter entries with a **cn**
```ldif
# ldapsearch -x -LLL -H ldap:/// -b ou=finance,dc=homelab,dc=local cn=*
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
objectClass: person
sn: Carter
cn: Sam Carter
```








<!--

## Securing OpenLDAP with TLS // FIXME: outdated

For security reasons, it's important to configure LDAP over TLS. Generate a self-signed certificate or obtain one from a Certificate Authority (CA), and then configure slapd to use it.

Edit the LDAP configuration file to include your TLS settings:

```shell
vi /etc/ldap/ldap.conf
```

Add the following lines, replacing the paths with the actual locations of your certificates:

    TLS_CACERT /etc/ssl/certs/ca-certificates.crt
    TLS_REQCERT demand

Restart the OpenLDAP service to apply the changes:

```shell
systemctl restart slapd
```

With TLS configured, your LDAP communications will be encrypted, enhancing the security of your directory service.

-->

#### See also 
- [LDAP for Rocket Scientists @Zytrax.com](https://www.zytrax.com/books/ldap/) Online book covering many topics. MUST READ !
- [Ldap search guide debutant](https://cyberinstitut.fr/utiliser-ldapsearch-guide-debutants/)
- [Example of using Modrdn, the LDIF changeType directive for ModifyDNRequest](https://ldapwiki.com/wiki/Wiki.jsp?page=Modrdn)




