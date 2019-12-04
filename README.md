# Salt Netapi REMOTE_USER auth

This module is used to authenticate to Saltstack REST API via REMOTE_USER. It supports multiple [auth backends](https://docs.saltstack.com/en/latest/ref/auth/all/index.html) but uses [shared secret auth backend](https://docs.saltstack.com/en/latest/ref/auth/all/salt.auth.sharedsecret.html) by default. It will verify all API requests and make sure the "username" parameter is not set to anything else than REMOTE_USER. This works with any Apache (or other webserver really) module which uses REMOTE_USER to expose the authenticated username to the application. Examples of this is https://github.com/modauthgssapi/mod_auth_gssapi, mod_auth* etc. If header X-Auth-Token is set then Apache should not care about REMOTE_USER, but let Salt handle authentication and authorization.

This has been tested with Apache2 as web frontend, but it should be possible to use with others that can take input and send to output through a filter.

## Usage

### Install into a virtualenv
```
$ git clone https://github.com/stockholmuniversity/salt-netapi-remoteuser-auth/
$ python3 -m virtualenv salt-netapi-remoteuser-auth
$ cd salt-netapi-remoteuser-auth && source bin/activate
$ pip3 install -r requirements.txt
$ ./auth_remote_user --test # Run tests validate that everything works
```

### Shared Secret External Auth
Setup sharedsecret in your Salt master config, see: https://docs.saltstack.com/en/latest/ref/auth/all/salt.auth.sharedsecret.html

Example:
```yaml
external_auth:
  sharedsecret:
    'foo@EXAMPLE.COM':
      - '.*'

sharedsecret: <your shared secret>
```

### Apache configuration
Enable the [ext_filter](https://httpd.apache.org/docs/2.4/mod/mod_ext_filter.html) apache module.
Set environment variable SHARED_SECRET_CONFIG and add ext filter to your apache site config:
```apache
SetEnvIf Request_Method POST SHARED_SECRET_CONFIG=/path/to/salt/master.d/sharedsecret.conf
ExtFilterDefine setRemoteUserAsUsername mode=input cmd="/path/to/script/auth_remote_user"

<LocationMatch />
    <If "-z %{HTTP:X-Auth-Token}">
        <RequireAll>
          AuthType GSSAPI
          GssapiCredStore keytab:/etc/krb5.keytab
          # Only allow Kerberos
          GssapiAllowedMech krb5
          # Only try SPNEGO once
          GssapiNegotiateOnce On
          require valid-user
        </RequireAll>
        SetInputFilter setRemoteUserAsUsername
    </If>
</LocationMatch>
```

### Switching eauth backend
You can switch which eauth backend salt should use, so e.g. if you want to use the rest auth backend:
```apache
SetEnvIf Request_Method POST EAUTH=rest
```

### Optional authorization and username rewrite

Using the environment variable `REMOTE_USER_PATTERN` you can do authorization
and also username rewrite.

#### authZ
Say you only want to allow users from the `@SU.SE`
realm you can do so with:

```apache
SetEnvIf Request_Method POST REMOTE_USER_PATTERN=.*@SU\.SE$
```

#### Username rewrite
If you want to rewrite the username from e.g. an administrative principal to a
regular user because the regular user is a member of an LDAP-group that you
want to reuse (but still use a different set of credentials) you can do like
this:
```apache
SetEnvIf Request_Method POST REMOTE_USER_PATTERN=^(.+?)\/root@SU\.SE$
```
The first group matched will be used.

## Example run

Since the auth script will set some parameters for us, we don't need to explicitly set them in the client:
```
curl -sS --negotiate -u: https://example.com/run -d tgt='*' -d fun='cmd.run' -d arg='id' -d client='local'
```

We could also use pepper with eauth=kerberos:
```
pepper --client=local_async '*' test.ping -H --password=anythinghere
```
