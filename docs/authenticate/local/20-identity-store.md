# Identity Store

The `user_name` and `user_secret` are password for the `superuser` in the database.

The plugin creates the following a file having the following structure.

```json
{
  "revision": 1,
  "users": [
    {
      "id": "cd5f647a-cc04-4ae2-9d0a-2d5e9b95cf98",
      "username": "webadmin",
      "email_addresses": [
        {
          "address": "webadmin@localdomain.local",
          "domain": "localdomain.local"
        }
      ],
      "passwords": [
        {
          "purpose": "generic",
          "type": "bcrypt",
          "hash": "$2a$10$B67nHY0PEdxLYdyoLk1YLOomvs.T/dSIyzPuoX9vWULrsD3PRf/sq",
          "cost": 10,
          "expired_at": "0001-01-01T00:00:00Z",
          "created_at": "2020-05-05T18:47:51.513552501Z",
          "disabled_at": "0001-01-01T00:00:00Z"
        }
      ],
      "created": "2020-05-05T18:47:51.513552066Z",
      "last_modified": "2020-05-05T18:47:51.513552175Z",
      "roles": [
        {
          "name": "superadmin"
        }
      ]
    }
  ]
}
```

Finally, browse to `/auth` and login with the username and password:

![](../images/basic_login.png)
