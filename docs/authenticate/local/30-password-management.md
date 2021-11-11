# Password Management

An administrator may change the password directly in the file called
`/etc/caddy/auth/local/users.json`.

First, download `bcrypt-cli`:

```bash
go get -u github.com/bitnami/bcrypt-cli
```

Then, use it to generate a new password:

```bash
$ echo -n "password123" | bcrypt-cli -c 10
$2a$10$OVnOaHDkcOXfbUZPFh5qt.yJqUt6pl9uJaqEMxxM.vS5fY/cZNmsq
```

Finally, replace the newly generated password in the user database file.
