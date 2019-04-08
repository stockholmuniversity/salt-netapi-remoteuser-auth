# Salt Netapi REMOTE_USER auth

This module is used to authenticate to Saltstack REST API via REMOTE_USER. It depends on using Salt's [shared secret auth backend](https://docs.saltstack.com/en/latest/ref/auth/all/salt.auth.sharedsecret.html). It will verify all API requests and make sure the "username" parameter is not set to anything else than REMOTE_USER. This works with SPNEGO, Kerberos, GSSAPI, and anything really that will set the REMOTE_USER variable.

This has been tested with Apache2 as web frontend, but it should be possible to use with others that can take input and send to output through a filter.

## Usage

### Download auth_remote_user script
Put the auth_remote_user somewhere on the Salt master, e.g /path/to/script/auth_remote_user.

### Shared Secret External Auth
Setup sharedsecret in your Salt master config, see: https://docs.saltstack.com/en/latest/ref/auth/all/salt.auth.sharedsecret.html

Example:
```
external_auth:
  sharedsecret:
    'foo@EXAMPLE.COM':
      - '.*'
```

### Apache configuration
Enable the [ext_filter](https://httpd.apache.org/docs/2.4/mod/mod_ext_filter.html) apache module.
Set environment variable SHARED_SECRET_CONFIG and add ext filter to your apache site config:
```
SetEnvIf Request_Method POST SHARED_SECRET_CONFIG=/path/to/salt/master.d/sharedsecret.conf
ExtFilterDefine setRemoteUserAsUsername mode=input cmd="/path/to/script/auth_remote_user"
<LocationMatch />
    SetInputFilter setRemoteUserAsUsername
</LocationMatch>
```

## Example curl

Since the auth script will set some parameters for us, we don't need to explicitly set them in the client:
```
curl -sS --negotiate -u: https://example.com/run -d tgt='*' -d fun='cmd.run' -d arg='id' -d client='local'
```

