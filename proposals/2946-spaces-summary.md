## MSC2946: Spaces Summary

This MSC depends on [MSC1772](https://github.com/matrix-org/matrix-doc/pull/1772), which
describes why a Space is useful:

> Collecting rooms together into groups is useful for a number of purposes. Examples include:
>
> * Allowing users to discover different rooms related to a particular topic: for example "official matrix.org rooms".
> * Allowing administrators to manage permissions across a number of rooms: for example "a new employee has joined my company and needs access to all of our rooms".
> * Letting users classify their rooms: for example, separating "work" from "personal" rooms.
>
> We refer to such collections of rooms as "spaces".

This MSC attempts to solve how a member of a space discovers rooms in that space. This
is useful for quickly exposing a user to many aspects of an entire community, using the
examples above, joining the "official matrix.org rooms" space might suggest joining a few
rooms:

* A room to discuss development of the Matrix Spec.
* An announcements room for news related to matrix.org.
* An off-topic room for members of the space.

## Proposal

A new client-server API (and corresponding server-server API) is added which allows
for querying for the rooms and spaces contained within a space. This allows a client
to efficiently display a hierarchy of rooms to a user (i.e. without having
to walk the full state of the space).

### Client-server API

An endpoint is provided to walk the space tree, starting at the provided room ID
("the root room"), and visiting other rooms/spaces found via `m.space.child`
events. It recurses into the children and into their children, etc.

Note that there is no requirement for any of the rooms to be of have a `type` of
`m.space`, any room with `m.space.child` events is considered.

This endpoint requires authentication and is not subject to rate-limiting.

Example request:

```text
GET /_matrix/client/r0/rooms/{roomID}/spaces?
    max_rooms_per_space=5&
    suggested_only=true
```

Example response:

```jsonc
{
    "rooms": [
        {
            "room_id": "!ol19s:bleecker.street",
            "avatar_url": "mxc://bleecker.street/CHEDDARandBRIE",
            "guest_can_join": false,
            "name": "CHEESE",
            "num_joined_members": 37,
            "topic": "Tasty tasty cheese",
            "world_readable": true,
            "room_type": "m.space"
        },
        { ... }
    ],
    "events": [
        {
            "type": "m.space.child",
            "state_key": "!efgh:example.com",
            "content": {
                "via": ["example.com"],
                "suggested": true
            },
            "room_id": "!ol19s:bleecker.street",
            "sender": "@alice:bleecker.street"
        },
        { ... }
    ]
}
```

Request params:

* **`suggested_only`**: Optional. If `true`, return only child events and rooms where the
  `m.space.child` event has `suggested: true`.  Must be a  boolean, defaults to `false`.
* **`max_rooms_per_space`**: Optional: a client-defined limit to the maximum
  number of children to return per space. Doesn't apply to the root space (ie,
  the `room_id` in the request). Must be a non-negative integer.

  Server implementations should impose a maximum value to avoid resource
  exhaustion.

Response fields:

* **`rooms`**: for each room/space, starting with the root room, a
  summary of that room. The fields are the same as those returned by
  `/publicRooms` (see
  [spec](https://matrix.org/docs/spec/client_server/r0.6.0#post-matrix-client-r0-publicrooms)),
  with the addition of:
  * **`room_type`**: the value of the `m.type` field from the
    room's `m.room.create` event, if any.
* **`events`**: `m.space.child` events of the returned rooms. For each event, only the
  following fields are returned: `type`, `state_key`, `content`, `room_id`,
  `sender`.<sup id="a1">[1](#f1)</sup>

Errors:

403 with an error code of `M_FORBIDDEN`: if the user doesn't have permission to
view/peek the root room (including if that room does not exist). This matches the
behavior of other room endpoints (e.g.
[`/_matrix/client/r0/rooms/{roomID}/aliases`](https://matrix.org/docs/spec/client_server/latest#get-matrix-client-r0-rooms-roomid-aliases)).
To not divulge whether the user doesn't have permission vs whether the room
does not exist.

#### Algorithm

A rough algorithm follows:

1. Start at the "root" room (the provided room ID).
2. Generate a summary and add it to `rooms`.
3. Add any `m.space.child` events in the room to `events`.
4. Recurse into the targets of the `m.space.child` events.
   1. If the room is not accessible (or has already been processed), do not
      process it.
   2. Generate a summary for the room and add it to `rooms`.
   3. Add any `m.space.child` events of the room to `events`.
5. Recurse into any newly added targets of `m.space.child` events (i.e. repeat
   step 4), until either all discovered rooms have been inspected, or the
   server-side limit on the number of rooms is reached.

   In the case of the homeserver not having access to the state of a room, the
   server-server API (see below) can be used to query for this information over
   federation.

Other notes:

* Any inaccessible children are omitted from the result, but the `m.space.child`
  events that point to them are still returned.
* There could be loops in the returned child events - clients should handle this
  gracefully.
* Similarly, note that a child room might appear multiple times (e.g. also be a
  grandchild). Clients and servers should handle this appropriately.
* `suggested_only` applies transitively.

  For example, if a space A has child space B which is *not* suggested, and space
  B has suggested child room C, and the client makes a summary request for A with
  `suggested_only=true`, neither B **nor** C will be returned.

  Similarly, if a space A has child space B which is suggested, and space B has
  suggested child room C which is suggested, and the client makes a summary request
  for A with `suggested_only=true`, both B and C will be returned.
* The current implementation doesn't honour `order` fields in child events, as
  suggested in [MSC1772](https://github.com/matrix-org/matrix-doc/pull/1772).
* `m.space.child` with an invalid `via` (invalid is defined as missing, not an
  array or an empty array) are ignored.

### Server-server API

The Server-Server API has almost the same interface as the Client-Server API.
It is used when a homeserver does not have the state of a room to include in the
summary.

Example request:

```jsonc
GET /_matrix/federation/v1/spaces/{roomID}?
    exclude_rooms=%21a%3Ab&
    exclude_rooms=%21b%3Ac&
    max_rooms_per_space=5&
    suggested_only=true&
```

The response has the same shape as the Client-Server API.

Request parameters are the same as the Client-Server API, with the addition of:

* **`exclude_rooms`**: Optional. A list of room IDs that can be omitted
  from the response.

This is largely the same as the Client-Server API, but differences are:

* The calling server can specify a list of spaces/rooms to omit from the
  response (via `exclude_rooms`).
* `max_rooms_per_space` applies to the root room as well as any returned
  children.
* If the target server is not a member of any discovered children (so
  would have to send another request over federation to inspect them), no
  attempt is made to recurse into them - they" are simply omitted from the
  `rooms` key of the response. (Although they will still appear in the `events`
  key).
  * If the target server is not a member of the root room, an empty
    response is returned.
* Currently, no consideration is given to room membership: the spaces/rooms
  must be world-readable (ie, peekable) for them to appear in the results.

## Potential issues

To reduce complexity, only a limited number of rooms are returned for a room,
no effort is made to paginate the results. Proper pagination is left to a future
MSC.

### MSC1772 Ordering

[MSC1772](https://github.com/matrix-org/matrix-doc/pull/1772) defines the ordering
of "default ordering of siblings in the room list" using the `order` key:

> Rooms are sorted based on a lexicographic ordering of the Unicode codepoints
> of the characters in `order` values. Rooms with no `order` come last, in
> ascending numeric order of the `origin_server_ts` of their `m.room.create`
> events, or ascending lexicographic order of their `room_id`s in case of equal
> `origin_server_ts`. `order`s which are not strings, or do not consist solely
> of ascii characters in the range `\x20` (space) to `\x7F` (~), or consist of
> more than 50 characters, are forbidden and the field should be ignored if
> received.

Unfortunately there are situations when a homeserver comes across a reference to
a child room that is unknown to it and must decide the ordering. Without being
able to see the `m.room.create` event (which it might not have permission to see)
no proper ordering can be given.

Consider the following case of a space with 3 child rooms:

```
         Space A
           |
  +--------+--------+
  |        |        |
Room B   Room C   Room D
```

Space A, Room B, and Room C are on HS1, while Room D is on HS2. HS1 has no users
in Room D (and thus has no state from it). Room B, C, and D do not have an
`order` field set (and default to using the ordering rules above).

When a user asks HS1 for the space summary with a `max_rooms_per_space` equal to
`2` it cannot fulfill this request since it is unsure how to order Room B, Room
C, and Room D, but it can only return 2 of them. It *can* reach out over
federation to HS2 and request a space summary for Room D, but this is undesirable:

* HS1 might not have the permissions to know any  of the state of Room D, so might
  receive a 403 error.
* If we expand the example above to many rooms than this becomes expensive to
  query a remote server simply for ordering.

This proposes changing the ordering rules from MSC1772 to the following:

* Rooms are sorted based on a lexicographic ordering of the Unicode codepoints
  of the characters in `order` values.

  `order`s which are not strings, or do not consist solely  of ascii characters
  in the range `\x20` (space) to `\x7F` (~), or consist of more than 50
  characters, are forbidden and the field should be ignored if received.
* Rooms with no `order` come last, in ascending lexicographic order of their
  `room_id`s.

This removes the clauses discussing using the `origin_server_ts` of the
`m.room.create` event to allow a defined sorting of siblings based purely on the
information available in the `m.space.child` event.

## Alternatives

An initial version of this followed both `m.space.child` and `m.space.parent` events,
but this is unnecessary to provide the expected user experience.

## Security considerations

A space with many rooms on different homeservers could cause multiple federation
requests to be made. A carefully crafted room with inadequate limits on the maximum
rooms per space (or a maximum total number of rooms) could be used in a denial
of service attack.

## Unstable prefix

During development of this feature it will be available at unstable endpoints.

The client-server API will be:
`/_matrix/client/unstable/org.matrix.msc2946/rooms/{roomID}/spaces`

The server-server API will be:
`/_matrix/federation/unstable/org.matrix.msc2946/spaces/{roomID}`

## Footnotes

<a id="f1"/>[1]: The rationale for including stripped events here is to reduce
potential dataleaks (e.g. timestamps, `prev_content`, etc.) and to ensure that
clients do not treat any of this data as authoritative (e.g. if it came back
over federation). The data should not be persisted as actual events.[↩](#a1)