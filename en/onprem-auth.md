Title: Log files

# Authentication

Landscape On-Premises supports a few methods of authentication for its web interface.

## PAM Support

If you want to use Pluggable Authentication Modules (PAM) to authenticate users in your new Landscape server you must create the file /etc/pam.d/landscape with the appropriate PAM configuration.

The simplest possible file is:

```
#%PAM-1.0
auth    required pam_permit.so
account required pam_permit.so
```

**Do NOT use this PAM setup on a production environment**

This allows any user to login without validating the password.

We have tested PAM authentication against an LDAP server running on Ubuntu, and against Windows AD authentication.

If you use PAM to authenticate, the user details stored in Landscape are associated with the PAM identity supplied.

For more information on PAM authentication see [PAM Tutorial](http://wpollock.com/AUnix2/PAM-Help.htm).


## OpenID Support

There is also support for authenticating Landscape users with an external OpenID provider. To enable OpenID support, please add `openid-provider-url` and `openid-logout-url` to `/etc/landscape/service.conf` in the `[landscape]` section. For example:

```
[landscape]
[…]
openid-provider-url = https://login.ubuntu.com/
openid-logout-url = https://login.ubuntu.com/+logout
```

After making these changes, restart all Landscape services:

```
sudo lsctl restart
```

There is no provision yet to upgrade current users to OpenID authentication. If you want to change your existing installation to use OpenID, you will have to migrate the existing users manually.

## Migrating existing users to OpenID authentication

To change the authentication mechanism of existing users to OpenID, you will need to insert each user's OpenID URL into the user entry in the database. Let's see an example.

Let's suppose we have an existing user called John Smith and we want to migrate him to OpenID. After changing `/etc/landscape/service.conf` and restarting the Landscape services, connect to the `landscape-standalone-main` database as an administrator:

```
ubuntu@ubuntu:~$ sudo -u postgres psql landscape-standalone-main
psql (8.4.11)
Type "help" for help.

landscape-standalone-main=# 
```

We now need to update the identity column of John's entry in the person table with his OpenID URL. Given John's email and his OpenID URL, the following SQL will do it:

```
UPDATE person SET identity = 'https://login.ubuntu.com/+id/FooBar' WHERE identity IS NULL AND email = 'john@example.com';
UPDATE 1
landscape-standalone-main=# 
```

This needs to be done for all users.


## OpenID-Connect Support

Landscape can now use OpenID-Connect (OIDC) to authenticate users. To enable OpenID-Connect support, please add `oidc-issuer`, `oidc-client-id` and `oidc-client-secret` to `/etc/landscape/service.conf` in the `[landscape]` section. For example:

```
[landscape]
[…]
oidc-issuer = https://accounts.google.com/
oidc-client-id = 000000000000-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.apps.googleusercontent.com
oidc-client-secret = a4sDFAsdfA4F52as-asDfAsd
```

The issuer is the URL of the issuer. That URL should also have a discovery configuration file available by appending `.well-known/openid-configuration`, such as [https://accounts.google.com/.well-known/openid-configuration](https://accounts.google.com/.well-known/openid-configuration). The client-id and client-secret should be provided by the OpenID-Connect provider when you create a client credentials. The provider may require setting an authorization redirect URI. This should look like `https://your_landscape/login/handle-openid`. If your provider also requires a logout redirect URL, this should be the address of your landscape server such as `https://your_landscape/` .

After making these changes, restart all Landscape services:

```
sudo lsctl restart
```

There is no provision yet to upgrade current users to OpenID-Connect authentication. Most providers return obfuscated subject identifiers which are not easily available. For this reason, we do not provide a user migration method and recommend than re-creating users.
