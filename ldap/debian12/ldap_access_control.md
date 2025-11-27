 [TOC]

# OpenLDAP Access Control 

Related to Video 12, 13, 14

## Check current rules

To check the current access control rules:

```shell
ldapsearch -x -D cn=admin,cn=config -w foobar123 -b cn=config
```

For our **{1}mdb** database (olcSuffix: dc=homelab,dc=local) we get:

```ldif
olcAccess: {0}to attrs=userPassword by self write by anonymous auth by * none
olcAccess: {1}to attrs=shadowLastChange by self write by * read
olcAccess: {2}to * by * read
```

The last one (**{2}**) gives read permissions to all attributes to everyone, which can be confirmed by running:
```shell
ldapsearch -x -b dc=homelab,dc=local objectClass=*
```

### Create or restore initial rules

For reference, if we had to implement this from scratch as in the Videos, we could have used:

```ldif
ldapmodify -D cn=admin,cn=config -w foobar123 
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword by self write by anonymous auth by * none
olcAccess: {1}to attrs=shadowLastChange by self write by * read
olcAccess: {2}to * by * read
```

To restore initial rules after modification, use this simply replacing **add:** by **replace:**



## Modify a rule and observe result

Let's change the permissive last rule to a restrictive version:

```ldif
# ldapmodify -Y EXTERNAL  -H ldapi:/// -D cn=admin,dc=config 
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {2}to * by * none

modifying entry "olcDatabase={1}mdb,cn=config"
```

Now, the following query won't return anything, contrary to the initial rules:

```shell
# ldapsearch -x -b dc=homelab,dc=local objectClass=*
# extended LDIF
#
# LDAPv3
# base <dc=homelab,dc=local> with scope subtree
# filter: objectClass=*
# requesting: ALL
#

# search result
search: 2
result: 32 No such object

# numResponses: 1
```

Now restore the original configuration (see section *Create or restore initial rules* above).



## Add a rule to block access to a specific DN

Let's now try to  block access to user *Sam Carter*. Before proceeding, confirm *Sam Carter* currently has access by running this query:

```shell
ldapsearch -x -H ldap:///   -D cn="Sam Carter",ou=finance,dc=homelab,dc=local -b dc=homelab,dc=local -W
```

If it's not working, check previous steps and make sure to restore original configuration (see section *Create or restore initial rules* above).


Let's add a blocking rule specific to *Sam Carter*:

```ldif
ldapmodify -D cn=admin,cn=config -w foobar123 
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to attrs=userPassword by self write by anonymous auth by * none
olcAccess: {1}to attrs=shadowLastChange by self write by * read
olcAccess: {2}to * by dn.exact="cn=Sam Carter,ou=finance,dc=homelab,dc=local" none
olcAccess: {3}to * by * read
```

Try again the *ldapsearch* above, it should now fail.

> **Note**: The third rule, granting read to everyone is not evaluated here because second rule is matched first, similarly to how most firewalls evaluate ACL rules.

# See also

- [8. Access Control on openldap.org](https://www.openldap.org/doc/admin24/access-control.html)
- [Les ACL dans OpenLDAP on vincentliefooghe.net](https://www.vincentliefooghe.net/content/les-acl-dans-openldap)


