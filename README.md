# Salt Netapi REMOTE_USER auth

This module is used to authenticate to Saltstack REST API via REMOTE_USER. It depends on using Salt's [shared secret auth backend](https://docs.saltstack.com/en/latest/ref/auth/all/salt.auth.sharedsecret.html). It will verify all API requests and make sure the "username" parameter is not set to anything else than REMOTE_USER. This works with any Apache (or other webserver really) module which uses REMOTE_USER to expose the authenticated username to the application. Examples of this is https://github.com/modauthgssapi/mod_auth_gssapi, mod_auth* etc. If header X-Auth-Token is set then Apache should not care about REMOTE_USER, but let Salt handle authentication and authorization.

This has been tested with Apache2 as web frontend, but it should be possible to use with others that can take input and send to output through a filter.

## Usage

### Download auth_remote_user script
Put the auth_remote_user somewhere on the Salt master, e.g /path/to/script/auth_remote_user.

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

## Example run

Since the auth script will set some parameters for us, we don't need to explicitly set them in the client:
```
curl -sS --negotiate -u: https://example.com/run -d tgt='*' -d fun='cmd.run' -d arg='id' -d client='local'
```

We could also use pepper with eauth=kerberos:
```
pepper --client=local_async '*' test.ping -H --password=anythinghere
```
