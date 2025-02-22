# LDAP Configuration

It is recommended reading the documentation for Local backend, because
it outlines important principles of operation of all backends.

Additionally, the LDAP backend works in conjunction with Local backend.
As you will see later, the two can be used together by introducing a
dropdown in UI interface to choose local versus LDAP domain authentication.

The reference configuration for the backend is in
[`assets/conf/ldap/Caddyfile`](https://github.com/greenpau/caddy-auth-portal/blob/main/assets/conf/ldap/Caddyfile).

The following Caddy endpoint at `/auth` authentications users
from `contoso.com` domain.

There is a single LDAP server associated with the domain: `ldaps://ldaps.contoso.com`.

The plugin DOES NOT ignore certificate errors when connecting to the servers.
However, one may ignore the errors by appending `ignore_cert_errors` to the
ldap server address.

```
          servers {
            ldaps://ldaps.contoso.com ignore_cert_errors
          }
```

As a better alternative to ignoring certificate errors, the plugin allows
adding trusted certificate authorities via `trusted_authority` Caddyfile directive:

```
          servers {
            ldaps://ldaps.contoso.com
          }
          trusted_authority /etc/gatekeeper/tls/trusted_authority/contoso_com_root1_ca_cert.pem
          trusted_authority /etc/gatekeeper/tls/trusted_authority/contoso_com_root2_ca_cert.pem
          trusted_authority /etc/gatekeeper/tls/trusted_authority/contoso_com_root3_ca_cert.pem
```

The following commands allow you connecting LDAPS server, e.g. `ldaps.localhost.local:636` and
collecting certificates for the `trusted_authority` directive.

```bash
mkdir -p certs && cd certs
openssl s_client -showcerts -verify 5 -connect ldaps.localhost.local:636 < /dev/null | \
    awk '/BEGIN/,/END/{ if(/BEGIN/){a++}; out="cert"a".crt"; print >out}' && \
    for cert in *.crt; do \
        newname=$(openssl x509 -noout -subject -in $cert | sed -n 's/^.*CN=\(.*\)$/\1/; s/[ ,.*]/_/g; s/__/_/g; s/^_//g;p').pem;
        mv $cert $newname;
    done
```

The LDAP attribute mapping to JWT fields is as follows.

| **JWT Token Field** | **LDAP Attribute** |
| --- | --- |
| `name` | `givenName` |
| `surname` | `sn` |
| `username` | `sAMAccountName` |
| `member_of` | `memberOf` |
| `email` | `mail` |

The plugin uses `authzsvc` domain user to perform LDAP bind.

The base search DN is `DC=CONTOSO,DC=COM`.

The plugin accepts username (`sAMAccountName`) or email address (`mail`)
and uses the following search filter: `(&(|(sAMAccountName=%s)(mail=%s))(objectclass=user))`.

For example:

```json
      {
        "Name": "sAMAccountName",
        "Values": [
          "jsmith"
        ]
      },
      {
        "Name": "mail",
        "Values": [
          "jsmith@contoso.com"
        ]
      }
```

Upon successful authentication, the plugin assign the following rules
to a user, provided the user is a member of a group:

| **JWT Role** | **LDAP Group Membership** |
| --- | --- |
| `admin` | `CN=Admins,OU=Security,OU=Groups,DC=CONTOSO,DC=COM` |
| `editor` | `CN=Editors,OU=Security,OU=Groups,DC=CONTOSO,DC=COM` |
| `viewer` | `CN=Viewers,OU=Security,OU=Groups,DC=CONTOSO,DC=COM` |

The security of the `password` could be improved by the following techniques:

* pass the password via environment variable `LDAP_USER_SECRET`
* store the password in a file and pass the file inside the `password`
  field with `file:` prefix, e.g. `file:/path/to/password`.

The following `Caddyfile` secures Prometheus/Alertmanager services. Users may access
using local and LDAP credentials.

```
{
  http_port     8080
  https_port    8443
  debug
}

127.0.0.1:8443 {
  route /auth* {
    authp {
      backends {
        crypto key sign-verify 0e2fdcf8-6868-41a7-884b-7308795fc286
        local_backend {
          method local
          path assets/conf/local/auth/user_db.json
          realm local
        }
        ldap_backend {
          method ldap
          realm contoso.com
          servers {
            ldaps://ldaps.contoso.com
          }
          trusted_authority /etc/gatekeeper/tls/trusted_authority/contoso_com_root1_ca_cert.pem
          trusted_authority /etc/gatekeeper/tls/trusted_authority/contoso_com_root2_ca_cert.pem
          trusted_authority /etc/gatekeeper/tls/trusted_authority/contoso_com_root3_ca_cert.pem
          attributes {
            name givenName
            surname sn
            username sAMAccountName
            member_of memberOf
            email mail
          }
          username "CN=authzsvc,OU=Service Accounts,OU=Administrative Accounts,DC=CONTOSO,DC=COM"
          # password "P@ssW0rd123"
          password "file:/etc/gatekeeper/auth/ldap.secret"
          search_base_dn "DC=CONTOSO,DC=COM"
          search_filter "(&(|(sAMAccountName=%s)(mail=%s))(objectclass=user))"
          groups {
            "CN=Admins,OU=Security,OU=Groups,DC=CONTOSO,DC=COM" admin
            "CN=Editors,OU=Security,OU=Groups,DC=CONTOSO,DC=COM" editor
            "CN=Viewers,OU=Security,OU=Groups,DC=CONTOSO,DC=COM" viewer
          }
        }
      }
      ui {
        logo url "https://caddyserver.com/resources/images/caddy-circle-lock.svg"
        logo description "Caddy"
        links {
          "Prometheus" /prometheus
          "Alertmanager" /alertmanager
          "My App" /myapp
        }
      }
    }
  }

  route /prometheus* {
    authorize {
      primary yes
      crypto key verify 0e2fdcf8-6868-41a7-884b-7308795fc286
      set auth url /auth
      allow roles authp/admin authp/user authp/guest
      allow roles superadmin
      allow roles admin editor viewer
      allow roles AzureAD_Administrator AzureAD_Editor AzureAD_Viewer
    }
    uri strip_prefix /prometheus
    reverse_proxy http://127.0.0.1:9080
  }

  route /alertmanager* {
    authorize
    uri strip_prefix /alertmanager
    reverse_proxy http://127.0.0.1:9083
  }

  route /myapp* {
    authorize
    respond * "myapp" 200
  }

  route /version* {
    respond * "1.0.0" 200
  }

  route {
    redir https://{hostport}/auth 302
  }
}
```
