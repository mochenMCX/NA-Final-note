## install LDAP server
TLS and force TLS search
```
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/server.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/server.key
-
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/ca.crt
```
Add Organizational Unit
```
dn: ou=Group,dc=16,dc=nasa
objectClass: top
objectClass: organizationalUnit
ou: Group

dn: ou=Ppolicy,dc=16,dc=nasa
objectClass: top
objectClass: organizationalUnit
ou: Ppolicy

dn: ou=SUDOers,dc=16,dc=nasa
objectClass: top
objectClass: organizationalUnit
ou: SUDOers

dn: ou=Fortune,dc=16,dc=nasa
objectClass: top
objectClass: organizationalUnit
ou: Fortune
```
加入 SSH
```
dn: cn=openssh-lpk,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: openssh-lpk
olcAttributeTypes: ( 1.3.6.1.4.1.24552.500.1.1.1.13 NAME 'sshPublicKey'
    DESC 'MANDATORY: OpenSSH Public key'
    EQUALITY octetStringMatch
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 )
olcObjectClasses: ( 1.3.6.1.4.1.24552.500.1.1.2.0 NAME 'ldapPublicKey' SUP top AUXILIARY
    DESC 'MANDATORY: OpenSSH LPK objectclass'
    MAY ( sshPublicKey $ uid )
    )
```
加入 role TA
```
dn: cn=ta_sudo,ou=SUDOers,dc=16,dc=nasa
objectClass: top
objectClass: sudoRole
cn: ta_sudo
sudoUser: %ta
sudoHost: ALL
sudoCommand: ALL
```
add-fortune-schema.ldif
```
dn: cn=fortune,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: fortune
# 1) author attribute: Directory String syntax for full matching support
olcAttributeTypes: ( 2.25.123456789012345678901234567890123456
  NAME 'author'
  DESC 'Person who authored the fortune'
  EQUALITY caseIgnoreMatch
  SUBSTR   caseIgnoreSubstringsMatch
  ORDERING caseIgnoreOrderingMatch
  SYNTAX  1.3.6.1.4.1.1466.115.121.1.15{256} )
# 2) id attribute: integer
olcAttributeTypes: ( 2.25.223456789012345678901234567890123456
  NAME 'id'
  DESC 'Numeric identifier for the fortune'
  EQUALITY integerMatch
  ORDERING integerOrderingMatch
  SYNTAX  1.3.6.1.4.1.1466.115.121.1.27 SINGLE-VALUE )
# 3) fortune objectClass
olcObjectClasses: ( 2.25.323456789012345678901234567890123456
  NAME 'fortune'
  DESC 'Fortune entry: must have author & id, may have description'
  SUP top
  STRUCTURAL
  MUST ( cn $ author $ id )
  MAY  ( description ) )
```
overlay.ldif
```
dn: olcOverlay=sssvlv,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcSssvlvConfig
olcOverlay: sssvlv
```
add-member.ldif
```
#dn: cn=ta,ou=Group,dc=16,dc=nasa
#changetype: modify
#add: memberUid
#memberUid: generalta
dn: cn=stu,ou=Group,dc=16,dc=nasa
changetype: modify
add: memberUid
memberUid: stu16
```
generalta.ldif
```
dn: uid=generalta,ou=People,dc=16,dc=nasa
objectClass: top
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
objectClass: ldapPublicKey
cn: General TA
sn:TA
uid: generalta
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/generalta
loginShell: /bin/bash
userPassword: {SSHA}yhsU7L4Ux0SKtP2tXCkqLtJ0ORbZZrO/
sshPublicKey: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFfg2DMY3DfBBvZCnqN8Az5tUnVQca+qXkJ9HceOcRAy 2025-na-hw4
```
modify.ldif
```
dn: uid=stu16,ou=People,dc=16,dc=nasa
changetype: modify
add: objectClass
objectClass: inetOrgPerson
-
replace: cn
cn: stu16
-
add: sn
sn: stu16
-
delete: objectClass
objectClass: account
```
stu16.ldif
```
dn: uid=stu16,ou=People,dc=16,dc=nasa
objectClass: top
objectClass: account
objectClass: posixAccount
cn: stu16
uid: stu16
uidNumber: 20016
gidNumber: 20000
homeDirectory: /home/stu16
userPassword: {SSHA}yhsU7L4Ux0SKtP2tXCkqLtJ0ORbZZrO/
```
add-stu-group.ldif
```
dn: cn=stu,ou=Group,dc=16,dc=nasa
objectClass: top
objectClass: posixGroup
cn: stu
gidNumber: 20000
```
grant1.ldif
```
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to dn.subtree="dc=16,dc=nasa" by dn.exact="uid=generalta,ou=People,dc=16,dc=nasa" write by * break
olcAccess: {1}to attrs=userPassword,shadowLastChange by self write by anonymous auth by * none
olcAccess: {2}to * by * read
```
moduleload.ldif
```
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: sssvlv
```
rootdn.ldif
```
{SSHA}yhsU7L4Ux0SKtP2tXCkqLtJ0ORbZZrO/dn: olcDatabase={1}mdb,cn=config
changetype: modify
# Add the new rootDN
add: olcRootDN
olcRootDN: uid=generalta,ou=People,dc=16,dc=nasa
-
# Store its password
add: olcRootPW
olcRootPW: {SSHA}yhsU7L4Ux0SKtP2tXCkqLtJ0ORbZZrO/
```
SUDOers.ldif
```
dn: cn=ta,ou=SUDOers,dc=16,dc=nasa
objectClass: top
objectClass: sudoRole
cn: ta
sudoUser: %ta
sudoHost: ALL
sudoRunAsUser: ALL
sudoRunAsGroup: ALL
sudoCommand: ALL

dn: cn=stu,ou=SUDOers,dc=16,dc=nasa
objectClass: top
objectClass: sudoRole
cn: stu
sudoUser: %stu
sudoHost: ALL
sudoRunAsUser: ALL
sudoRunAsGroup: ALL
sudoCommand: /usr/bin/ls
```
add-ta-group.ldif
```
dn: cn=ta,ou=Group,dc=16,dc=nasa
objectClass: top
objectClass: posixGroup
cn: ta
gidNumber: 10000
```
sudo.ldif
```
dn: cn=sudo,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: sudo
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.1 NAME 'sudoUser' DESC 'User(s) who may  run sudo' EQUALITY caseExactIA5Match SUBSTR caseExactIA5SubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.2 NAME 'sudoHost' DESC 'Host(s) who may run sudo' EQUALITY caseExactIA5Match SUBSTR caseExactIA5SubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.3 NAME 'sudoCommand' DESC 'Command(s) to be executed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.4 NAME 'sudoRunAs' DESC 'User(s) impersonated by sudo (deprecated)' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.5 NAME 'sudoOption' DESC 'Options(s) followed by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.6 NAME 'sudoRunAsUser' DESC 'User(s) impersonated by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.7 NAME 'sudoRunAsGroup' DESC 'Group(s) impersonated by sudo' EQUALITY caseExactIA5Match SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcObjectClasses: ( 1.3.6.1.4.1.15953.9.2.1 NAME 'sudoRole' SUP top STRUCTURAL DESC 'Sudoer Entries' MUST ( cn ) MAY ( sudoUser $ sudoHost $ sudoCommand $ sudoRunAs $ sudoRunAsUser $ sudoRunAsGroup $ sudoOption $ description ) )
```
