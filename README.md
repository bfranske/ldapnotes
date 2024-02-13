# Introduction
This page is intended to provide a tutorial for the setup and configuration of OpenLDAP on a Debian system complete with Argon2 based password hashing and memberOf dynamic lists. It is surprisingly hard to find this information online as there have been many changes to OpenLDAP which seem to be both poorly documented in official documentation and much outdated information in third party documentation.

Modern OpenLDAP configuration is done as "online configuration" (olc) within a special `cn=config` branch of the directory though many sources online only talk about the old slapd.conf configuration method. All of the configuration here will use the new online configuration style.

The aim here is to give you enough information about and experience setting up OpenLDAP that you understand the basics and are able to do adapt this to your specific use and configuration cases.

# Background Information on LDAP and Directories

## About Distinguished Names (DNs)

Distinguished names (DNs) are the path to an item or location within the hierarchical directory information tree structure. DNs are read from right (the root of the tree) to left (the leaf) and are comma-seperated key/value pairs. DNs and all attributes are case-insensitive by default though it is possible to configure an attribute as case sensitive in the schema. 

Examples of keys you may see in a DN:
* **CN** - commonName
* **L** - localityName
* **ST** - stateOrProvinceName
* **O** - organizationName
* **OU** - organizationalUnitName
* **C** - countryName
* **STREET** - streetAddress
* **DC** - domainComponent
* **UID** - userid

These keys are actually atributes of an entry in the directory and there are actually many more common attributes in use. It is also possible to define your own custom attributes. However, the attributes listed above are some of the most common ones you may see in a DN. Zytrax has a list of some other [commonly used attributes](https://www.zytrax.com/books/ldap/ape/#attributes).

So an example of a DN might be `"CN=Joelle Zboncak,OU=People,DC=example,DC=com"`. In this case this is most likely a user account belonging to "Joelle Zboncak" in the "People" organizational unit of the "example.com" directory.

Not all DNs are users though, another DN might be `"CN=Developers,OU=Groups,DC=example,DC=com"`. In this case this is probably a group named "Developers" in the "Groups" organizational unit of the "example.com" directory.

Each of these entires is represented by a similar DN (though in a different OU) so how does the directory know one is a user account and the other is a group. It is not, as you might think, related to the OU they are stored in, those are only named People and Groups for convenience and convention, you are free to place any type of entry in any OU. Instead, each entry is assigned one or more **objectClasses** which define the type of entry. The objectClass may itself (and likely does) have a parent objectClass from which it inherits some of it's attributes. ObjectClasses may make certain attributes mandatory in an entry and others optional. 

Examples of objectClasses you are likely to encounter:
* **person**
* **groupOfNames** - This is the typical objectClass for Groups
* **organizationalPerson** - This is a sub-objectClass of Person
* **inetOrgPerson** - This is a sub-objectClass of organizationalPerson and is the standard objectClass for network users
* **organization**
* **organizationalUnit** - This is used to define OUs

There are many more objectClasses and your can define your own as well. Zytrax has a list of some other [common objectClasses and their mandatory and optional attributes](https://www.zytrax.com/books/ldap/ape/#objectclasses).

# OpenLDAP Server Installation

In order to complete this tutorial you will need the following Debian packages installed:
* **slapd** - The OpenLDAP server
* **slapd-contrib** - Additional plugins for the OpenLDAP server including support for the Argon2 password hashing algorithm and dynamic lists
* **ldap-utils** - Command line utilities for adding to, deleting from, modifying, and searching the directory with the LDAP protocol

Install these like:
```
$ sudo apt install slapd slapd-contrib ldap-utils
```

During the initial installation of these packages you may be asked some minimal questions about configuration such as setting up an administrative password and/or a domain name. We'll do further configuration in the following step wich will re-ask those questions so see there for more information.

# Initial Configuration

You may not have been asked about both and administrative password and domain name during the initial package installation but you can re-set these by reconfiguring the package. Note that this will create a brand new OpenLDAP directory with no data so you don't want to do this after you have added a bunch of users, groups, organizational units (OUs), or configuration.

The base of your OpenLDAP directory will be set as the domain name you give in this configuration so if you enter a domain name of `example.com` during package configuration your base distinguished name (dn) will be `dc=example,dc=com` and the dn of your initial administrative user will be `cn=admin,dc=example,dc=com`. If you don't know about DNs yet don't worry, they are explained later in this tutorial.

Complete the initial configuration by reconfiguring the package:
```
sudo dpkg-reconfigure -plow slapd
```

# Creating Organizational Units (OUs)

It's generally a good idea to have some organization in your directory. Even for a small directory you probably want to separate where users and groups are stored so let's create OUs for users and groups.

The `ldapadd` command allows us to add an entry into our directory by passing LDAP Data Interchange Format (LDIF) data to the directory server. Instead of an LDIF file for the tutorial, we will provide the LDIF data directly from the command line through standard input. However, you could create the OUs with an LDIF file or using some other LDAP client instead.

Create a users OU:

```
$ ldapadd -x -D 'cn=admin,dc=example,dc=com' -W << EOF
dn: ou=people,dc=example,dc=com
objectClass: organizationalUnit
ou: People
EOF
```

Note that in the first line of the above command the dn `'cn=admin,dc=example,dc=com'` is used to login to the directory with a user who has permissions to make the change, in our case the original administrative user that was created when the directory was initialized. This is the password you need to provide when the command is issued. In the second line the LDIF data gives the DN (path) in the directory where the OU is to be placed. The third line defines the objectClass to identify the entry as an OU, and the fourth line sets the name of the OU. Note that the name of the OU is also represented in the DN.

Now create a groups OU:

```
$ ldapadd -x -D 'cn=admin,dc=example,dc=com' -W << EOF
dn: ou=groups,dc=example,dc=com
objectClass: organizationalUnit
ou: Groups
EOF
```

# Configuring the Directory for Modern Password Hashing (Argon2)

By directory RFC, and for interoperablity purposes, user passwords are supposed to be stored in plaintext. However, unless you have a very good reason to (and you should not) this is widely understood to be a bad thing. This is why pretty much every directory server, including OpenLDAP supports password hashing and storing passwords as hashes. The best built-in password hash in OpenLDAP is salted SHA but that has some well known weaknesses and is generally no longer considered a best practice password hashing algorithm. Luckily, there is a module available for OpenLDAP to use the Argon2 hash which is [recommended by OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html).

Note that this algorithm will be used for passwords created going forward which is why we're doing it before creating users in our tutorial. It is possible to migrate an existing directory to use Argon2 hashes as users change their passwords, doing so safely is beyond the scope of this tutorial though. Also, note that our administrative user `'cn=admin,dc=example,dc=com'` is not using the Argon2 hash as they were created previously. You should update that user's password to use Argon2 as well (as wel will below).

Selecting the specific parameters of Argon2 to use is outside the scope of this tutorial. However, be aware of the need to balance how frequently your server will be authenticating users, memory and CPU available on the server, and security concerns when selecting parameters. Also, as of this writing the Debian OpenLDAP Argon2 module uses the Argon2i varient of Argon2 when storing passwords and doesn't choose particularly secure parameters. Per [OWASP recommendations](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) Argon2id is the preferred varient. A [patch was submitted and accepted](https://git.openldap.org/openldap/openldap/-/merge_requests/643) to change that so future versions of OpenLDAP server in Debian will use Argon2id by default. The good news is that in the meantime if you use some other software to create the Argon2id hash and then prepend it with `{ARGON2}` like `{ARGON2}$argon2id$v=19$m=16,t=2,p=1$TmpzS0Y0dDZDVnhoaTRBYg$DJO5QlKmmw1X6FW7UdzIBA` and store it as a plaintext password in the directory the directory will still be able to authenticate against it.

Modify the configuration of the directory to load the Argon2 module:
```
$ sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// << EOF
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: "argon2 m=12288"
EOF
```

Change the default password hashing algorithm to use Argon2:
```
$ sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// << EOF
dn: olcDatabase={-1}frontend,cn=config
changetype: modify
add: olcPasswordHash
olcPasswordHash: {ARGON2}
EOF
```

Create a test password hash to see the Argon2 module is loaded:
```
$ slappasswd -h "{ARGON2}" -o module-load="argon2 m=12288"
```

Run the above command again if it was successful and generate a new hashed password for the admin user.

Update the password for the admin user by copying and pasting the hash into the `olcRootPW` attribute of this command before issuing it to change the admin password:
```
$ sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {ARGON2}xxxxxxxxxxxxxx
EOF
```

Test whether this password update worked by getting the admin user password hash back from the directory:
```
$ sudo ldapsearch -LLLQ -Y EXTERNAL -H ldapi:/// -b olcDatabase={1}mdb,cn=config olcRootPW
```

If you see the `olcRootPW` come back starting as `{ARGON2}` then it worked! Just to be sure let's actually try authenticating though:
```
$ ldapwhoami -x -D cn=admin,dc=example,dc=com -W
```

If this is successful it will come back to you with the DN of your user.

# Creating Users

Let's create a sample user as an inetOrgPerson since that is the standard way to store network login users. As a sub-objectClass of person an inetOrgPerson must have a the cn and sn attributes set, other attributes are optional. We'll use CN as the naming attribute in the dn as it is required and uid is not. Some directory systems use uid in the dn instead of cn, but either is a valid choice. Usually applications which connect to the directory server for authentication can have their configuration set to use either cn or uid.

Common attributes for inetOrgPerson objectClass:
* **cn** - commonName (REQUIRED)
* **sn** - surname also known as last name (REQUIRED)
* **gn** - givenName also known as first name
* **displayName**
* **uid** - userid

Create a sample user:
```
$ ldapadd -x -D 'cn=admin,dc=example,dc=com' -W << EOF
dn: cn=nbosco,ou=people,dc=example,dc=com
gn: Natalia
cn: nbosco
displayName: Natalia
sn: Bosco
objectClass: person
objectClass: inetOrgPerson
EOF
```

Set the user's initial password as the admin user:
```
$ ldappasswd -x -D 'cn=admin,dc=example,dc=com' -W cn=nbosco,ou=people,dc=example,dc=com -S
```
Note that you need to enter the admin's password after you enter the user's new password twice to make an administrative password change.

Do an anonymous search on the directory for the user's entry:
```
$ ldapsearch -LLL -x -b 'ou=people,dc=example,dc=com' cn=nbosco
```
Note that some information such as the user's password hash are not available from an anonymous search.

Do an authenticated search on the user, logged in as the user:
```
$ ldapsearch -LLL -x -D 'cn=nbosco,ou=people,dc=example,dc=com' -W -b 'ou=people,dc=example,dc=com' cn=nbosco
```
Note this time that we see the userPassword attribute though it is still Base64 encoded (as could be other attributes) because it is a UTF-8 encoded value. Base64 encoded values are denoted in ldapsearch output by a double colon `::` after the attribute name.

# Adding Support for Dynamic List memberOf User Attribute

Coming Soon!

# Adding Groups and Users to Groups

# Enabling TLS with LDAPS

Coming Soon!

# Resources
1. [fkooman gives an excellent starting point for online configuration of OpenLDAP with Argon2 configuration](https://codeberg.org/fkooman/paste/src/branch/main/LDAP_SETUP.md) though it still uses the deprecated memberof overlay instead of the dynlist replacement. Much of this page is inspired by this work.
2. [Puzzle ITC has some information about configuring additional Argon2 parameters in this post](https://www.puzzle.ch/de/blog/articles/2023/08/08/enhancing-openldap-security-with-argon2) though it is focused on Rocky Linux and pushign configuration via Ansible instead of directly configuring OpenLDAP which is more suitable for a tutorial.
3. [The online book LDAP for Rocket Scientists at zytrax.com](https://www.zytrax.com/books/ldap/) is perhaps the most thorough treatment of OpenLDAP and a great place to go now that you have your feet wet. However, it is a bit of drinking from a firehose  when your're just getting started, still has a lot of slapd.conf style configuration details, and lacks information on things like configuring dynamic lists in an online configuration environment.
