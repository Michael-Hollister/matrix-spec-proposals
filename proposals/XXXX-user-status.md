# MSCXXXX: User status endpoint

Matrix clients sometimes need a way to display additional information about a
user. For example, when interacting with a Matrix user, it might be useful for
clients to show whether this user has been deactivated, or even exists at all.

Currently clients are forced to resort to hacks to try to derive this
information. Some, for example, check whether a user has a profile (display
name, avatar) set when trying to send an invite, which means that if a user
doesn't have any profile information (which is valid under the Matrix
specification), the inviter might be warned that the user might not exist.

## Proposal

Two new endpoints are added to the specification, one to the client-server API
and one to the server-server API.

### `GET /_matrix/client/v1/user/status`

This endpoint requires authentication via an access token.

This endpoint takes a `user_id` query parameter indicating which user(s) to look
up the status of. This parameter may appear multiple times in the request if the
client wishes to look up the statuses of multiple users at once.

If no error arises, the endpoint responds with a body using the following
format:

```json
{
    "@user1:example.com": {
        "exists": true,
        "deactivated": false
    },
    "@user2:example.com": {
        "exists": false
    }
}
```

Each top-level key in the response maps to one of the `user_id` parameter(s)
in the request. For each user:

* `exists` is a boolean that indicates whether the user exists.
* `deactivated` is a boolean that indicates whether the user has been
  deactivated. Omitted if `exists` is `false`.

If one or more user(s) is not local to the homeserver this request is performed
on, the homeserver must attempt to retrieve user status using the federation
endpoint described below.

Below is how this endpoint behaves in case of errors:

* If no `user_id` parameter is provided, the endpoint responds with a 400 status
  code and a `M_MISSING_PARAM` error code.
* If one or more of the `user_id` parameter(s) provided cannot be parsed as a
  valid Matrix user ID, the endpoint responds with a 400 status code and a
  `M_INVALID_PARAM` error code.

### `GET /_matrix/federation/v1/query/user_status`

This endpoint behaves in an identical way to the client-side endpoint described
above.

### `m.user_status` capability

Some server administrators might not want to disclose too much information about
their users. To support this use case, homeservers must expose a `m.user_status`
capability to tell clients whether they support retrieving user status via the
client-side endpoint described above.

## Potential issues

It is not clear how to show users whose status could not be retrieved because of
federation issue, a missing or blocked federation endpoint, or an incomplete
response from the remote homeserver, in the client-side endpoint's response. I
have considered omitting them from the response entirely, or adding an
additional `success` boolean to each user's status to indicate whether status
for this user could be retrieved, but I am not entirely satisfied with either
solution. I am open to suggestions on this point.

## Security considerations

Should a server administrator not want to disclose their users' statuses through
the federation endpoint described above, they should use a reverse proxy or
similar tool to prevent access to the endpoint. On top of this, homeserver
implementations may implement measures to only respond with an empty JSON object
`{}` in this case.

## Unstable prefixes

Until this proposal is stabilised in a new version of the Matrix specification,
implementations should use the following paths for the endpoints described in
this document:

| Stable path                                | Unstable path                                                       |
|--------------------------------------------|---------------------------------------------------------------------|
| `/_matrix/client/v1/user/status`           | `/_matrix/client/unstable/org.matrix.mscXXXX/user/status`           |
| `/_matrix/federation/v1/query/user_status` | `/_matrix/federation/unstable/org.matrix.mscXXXX/query/user_status` |

Additionally, implementations should use the unstable identifier
`org.matrix.mscXXXX.user_status` instead of `m.user_status` for the client-side
capability.