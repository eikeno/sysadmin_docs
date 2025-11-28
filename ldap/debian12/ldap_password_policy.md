# Password Policy

Related to Videos #15 and following

## Introduction

Password policies can be used in order to place constraints of passwords at creation or modification time, for example:

- password minimal lenght
- new password cannot match current password
- password expiration
- etc.


## Load the ppolicy schema

This part must be skipped because:

> In OpenLDAP 2.4 the slapo-ppolicy(5) overlay relied on a separate schema file to be included for it to function. This schema is now implemented internally in the slapo-ppolicy module. When upgrading slapd.conf(5) deployments the include statement for the schema must be removed. For slapd-config(5) deployments, the config database must be exported via slapcat and the old ppolicy schema removed from the export. The resulting config database can then be imported.

source: [https://www.openldap.org/doc/admin25/appendix-upgrading.html](https://www.openldap.org/doc/admin25/appendix-upgrading.html)

As Debian 12 uses v2.5 of Openldap, we don't have any schema to load, and can move on directly to load the module and apply the overlay.


## Load the ppolicy module

```ldif 
# ldapmodify -Y EXTERNAL -H ldapi:///
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: ppolicy
```

> **Note**: This step is added compared to the Videos due to changes in Openldap v2.5

## Add the overlay

```ldif
# ldapmodify -Y EXTERNAL -H ldapi:/// -D cn=admin,cn=config
dn: olcOverlay=ppolicy,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcPPolicyConfig
olcOverlay: ppolicy
olcPPolicyDefault: cn=Normal Policy,dc=homelab,dc=local
olcPPolicyHashClearText: TRUE
```

## Create the policy
Create the policy required by the **olcPPolicyDefault**  setting in the previous section:

```ldif
# ldapmodify -x -H ldap:/// -D cn=admin,dc=homelab,dc=local -w secret
dn: cn=Normal Policy,dc=homelab,dc=local
changetype: add
objectClass: device
objectClass: pwdPolicy
cn: Normal Policy
pwdAttribute: userPassword
pwdInHistory: 2
```

> **Note**: The inclusion of **objectClass: device** in an LDAP ppolicy (Password Policy) entry is a common practice in OpenLDAP configurations, and it serves a purely technical function to satisfy the schema requirements for the entry.


## Testing

Now, a user shouldn't be able to set a new pasword equal to the last two previous one. Let's check:

```ldif
# ldapmodify -x -H ldap:/// -D cn="Sam Carter",ou=finance,dc=homelab,dc=local -w newsecret2
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
changetype: modify
replace: userPassword
userPassword: newsecret2

modifying entry "cn=Sam Carter,ou=finance,dc=homelab,dc=local"
ldap_modify: Constraint violation (19)
	additional info: Password is not being changed from existing value
```

The policy works, as we receive the expected error due to trying to set the same password.

We can also confirm that password change will work when using a new value:

```ldif
# ldapmodify -x -H ldap:/// -D cn="Sam Carter",ou=finance,dc=homelab,dc=local -w newsecret2
root@d12sa:/etc/ldap# ldapmodify -x -H ldap:/// -D cn="Sam Carter",ou=finance,dc=homelab,dc=local -w newsecret2
dn: cn=Sam Carter,ou=finance,dc=homelab,dc=local
changetype: modify
replace: userPassword
userPassword: newsecret3

modifying entry "cn=Sam Carter,ou=finance,dc=homelab,dc=local"

^C
```


# See also

- [12.10. Password Policies @openldap.org](https://www.openldap.org/devel/admin/overlays.html)
