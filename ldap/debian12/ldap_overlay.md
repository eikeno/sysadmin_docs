[TOC]
# Adding an overlay :  AccessLog, log to a dedicated MDB database

For the demo, following Videos #7, 8, 9 we add an AccessLog overlay to:

- **{1}mdb** database (olcSuffix: dc=homelab,dc=local)
  - so that it logs in a new database:
- **{2}mdb** database (olcSuffix: cn=log)

Setting up the accesslog overlay using OLC (Online Configuration) in OpenLDAP requires three main steps:

- Loading the module
- Creating a separate accesslog database
- Applying the overlay to the target database.


## Load the Accesslog Module

First, ensure the accesslog module is loaded into the server configuration.

Create an LDIF file

load_accesslog.ldif:
```ldif
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: accesslog
```

Apply the change using ldapmodify:
```shell
# ldapmodify -Y EXTERNAL -H ldapi:/// -f load_accesslog.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=module{0},cn=config"
```

> Note: The index **{0}** in **cn=module{0},cn=config** may vary depending on your existing configuration. Check the directory */etc/ldap/slapd.d/cn=config/* to find out.


## Create the log Database

The access log entries are stored in their own, separate database. You must create this database before applying the overlay.

Create new database directory with appropriate owner and permissions:
```shell 
mkdir -p /var/lib/ldap-access
chown openldap:openldap /var/lib/ldap-access
chmod 755 
```

Create an LDIF file for the new database:

009-addDatabaseLOG-minimal.ldif:
```ldif
dn: olcDatabase={2}mdb,cn=config
changetype: add
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: {2}mdb
olcSuffix: cn=log
olcDbDirectory: /var/lib/ldap-access
```

For reference __only__, a more elaborate ldif file would look like:
```ldif
dn: olcDatabase={X}mdb,cn=config
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: mdb
olcDbDirectory: /var/lib/ldap/accesslog 
olcSuffix: cn=accesslog
olcRootDN: cn=admin,cn=accesslog
olcRootPW: {SSHA}password_hash_here
olcDbIndex: default eq
olcDbIndex: entryCSN,objectClass,reqEnd,reqResult,reqStart,reqDN
```
Refer to man page for details.

But for the sake of simplicity and keeping close to the Videos content, let's stick to *009-addDatabaseLOG-minimal.ldif* here.

Apply the change:

```shell
# ldapmodify -x -H ldap:/// -D cn=admin,cn=config -w foobar123 -f 008-addDatabaseLOG.ldif 
adding new entry "olcDatabase={2}mdb,cn=config"
```

Or, if you want to use SASL the equivalent is (you need to be *root*:
```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f  009-addDatabaseLOG-minimal.ldif 
```

You can check creation with:

```ldif
# ldapsearch -x -LLL -H ldap:/// -D cn=admin,cn=config  -w foobar123 -b cn=config objectclass=*
[...truncated output ...]
dn: olcDatabase={2}mdb,cn=config
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: {2}mdb
olcDbDirectory: /var/lib/ldap-access
olcSuffix: cn=log
```

## Apply the Accesslog Overlay to the Target Database

Finally, apply the overlay to the main database ( and point it to the new accesslog database.

010-add_accesslog_overlay.ldif:
```ldif
dn: olcOverlay=accesslog,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcAccessLogConfig
olcOverlay: accesslog
olcAccessLogDB: cn=log
olcAccessLogOps: writes reads session
olcAccessLogSuccess: TRUE
olcAccessLogPurge: 07+00:00 01+00:00
```

- **olcDatabase={1}mdb**: the DN of your target database (the one whose activity you want to log).
- **olcAccessLogDB**: Must match the olcSuffix of the database created in step 2 (cn=log).
- **olcAccessLogOps**: Specifies which operations to log (writes = add, delete, modify, modrdn; reads = compare, search; session = abandon, bind, unbind; all for everything).
- **olcAccessLogPurge**: Defines the purge schedule: 7+00:00 (purge entries older than 7 days) and 1+00:00 (run the purge task daily at 01:00).

```shell
# ldapmodify -Y EXTERNAL -H ldapi:/// -f 010-add_accesslog_overlay.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "olcOverlay=accesslog,olcDatabase={1}mdb,cn=config"
```

After applying the overlay, the configured operations on the target database will be logged as entries in the **cn=log** database, which you can query using standard LDAP search commands.

## Testing

Do a query to generate activity to be logged:
```shell
ldapsearch -x -LLL -H ldap:/// -b ou=finance,dc=homelab,dc=local cn=*
```

Verify that log is fed with new data:
```shell
ldapsearch -Y EXTERNAL -H ldapi:/// -D cn=admin,cn=config  -b cn=log
```

Should output something similar to:
```ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
# extended LDIF
#
# LDAPv3
# base <cn=log> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# log
dn: cn=log
objectClass: auditContainer
cn: log

# 20251125160359.000001Z, log
dn: reqStart=20251125160359.000001Z,cn=log
objectClass: auditSearch
reqStart: 20251125160359.000001Z
reqEnd: 20251125160359.000002Z
reqType: search
reqSession: 1069
reqAuthzID:
reqDN: ou=finance,dc=homelab,dc=local
reqResult: 0
reqScope: sub
reqDerefAliases: never
reqAttrsOnly: FALSE
reqFilter: (cn=*)
reqEntries: 1
reqTimeLimit: 3600
reqSizeLimit: 500

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```





# Adding an overlay :  AuditLog, log to an LDIF file

See Videos #8 and following.

## Load the Accesslog Module

011-load_auditlog.ldif:
```ldif
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: auditlog
```

```shell
# ldapmodify -Y EXTERNAL -H ldapi:/// -f 011-load_auditlog.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=module{0},cn=config"
```


##  Create directory for AuditLog LDIF files

Create new log directory with appropriate owner and permissions:
```shell 
mkdir -p /var/lib/ldap-audit
chown openldap:openldap /var/lib/ldap-audit
chmod 755 /var/lib/ldap-audit
```

## Apply the Overlay
012-add_auditlog_overlay.ldif:
```ldif
dn: olcOverlay=auditlog,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcAuditLogConfig
olcOverlay: auditlog
olcAuditlogFile: /var/lib/ldap-audit/auditlog.ldif
```

```shell
# ldapadd -Y EXTERNAL -H ldapi:/// -f 012-add_auditlog_overlay.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "olcOverlay=auditlog,olcDatabase={1}mdb,cn=config"
```

## Testing 

Do a query to generate activity to be logged, for example:

```ldif 
# ldapmodify -x  -H ldap:/// -D cn=admin,dc=homelab,dc=local -w secret
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
changetype: modify
replace: userPassword
userPassword: newsecret2

modifying entry "cn=Sam Carter,ou=finance,dc=homelab,dc=local"

^C
```


Check file /var/lib/ldap-audit/auditlog.ldif
```ldif
# cat /var/lib/ldap-audit/auditlog.ldif
# modify 1764153842 dc=homelab,dc=local cn=admin,dc=homelab,dc=local IP=127.0.0.1:60354 conn=1116
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
changetype: modify
replace: userPassword
userPassword:: bmV3c2VjcmV0Mg==
-
replace: entryCSN
entryCSN: 20251126104402.078514Z#000000#000#000000
-
replace: modifiersName
modifiersName: cn=admin,dc=homelab,dc=local
-
replace: modifyTimestamp
modifyTimestamp: 20251126104402Z
-
# end modify 1764153842
```

Confirm password value:
```shell
# echo bmV3c2VjcmV0Mg== | base64 -d ; echo
newsecret2
```

# See also

- [12. Overlays @openldap.org admin guide](https://www.openldap.org/doc/admin24/overlays.html)
- [Tyler's Guides : OpenLDAP Audit Log Overlay](https://tylersguides.com/guides/openldap-audit-log-overlay/)
- [man 5 slapo-accesslog](https://www.openldap.org/software/man.cgi?query=slapo-accesslog&apropos=0&sektion=5&manpath=OpenLDAP+2.4-Release)
- [man 5 slapo-auditlog](https://www.openldap.org/software/man.cgi?query=slapo-auditlog&sektion=5&apropos=0&manpath=OpenLDAP+2.4-Release)