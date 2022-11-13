# MSC3381: Chat Polls

Polls are additive functionality in a room, allowing someone to pose a question and others to answer
until the poll is closed. In chat, these are typically used for quick questionares such as what to
have for lunch or when the next office party should be, not elections or anything needing truly
secret ballot.

[MSC2192](https://github.com/matrix-org/matrix-doc/pull/2192) does introduce a different way of doing
polls (originally related to inline widgets, but diverged into `m.room.message`-based design). That
MSC's approach is discussed at length in the alternatives section for why it is inferior.

## Proposal

Polls are intended to be handled completely client-side and encrypted when possible in a given room.
They are started by sending an event, responded to using events, and closed using more events - all
without involving the server (beyond being the natural vessel for sending the events). Other MSCs
related to polls might require changes from servers, however this MSC is intentionally scoped so that
it does not need server-side involvement.

The events in this MSC make use of the following functionality:

* [MSC1767](https://github.com/matrix-org/matrix-doc/pull/1767) (extensible events & `m.markup`)
* [Event relationships](https://spec.matrix.org/v1.4/client-server-api/#forming-relationships-between-events)
* [Reference relations](https://github.com/matrix-org/matrix-spec/pull/1206) (**TODO:** Link to final spec)

To start a poll, a user sends an `m.poll` event into the room. An example being:

```json5
{
  "type": "m.poll",
  "sender": "@alice:example.org",
  "content": {
    "m.markup": [
      // Markup is used as a fallback for text-only clients which don't understand polls. Specific formatting is
      // not specified, however something like the following is likely best.
      {
        "mimetype": "text/plain",
        "body": "What should we order for the party?\n1. Pizza 🍕\n2. Poutine 🍟\n3. Italian 🍝\n4. Wings 🔥"
      }
    ],
    "m.poll": {
      "kind": "m.disclosed",
      "max_selections": 1,
      "question": {
        "m.markup": [{"body": "What should we order for the party?"}]
      },
      "answers": [
        {"m.id": "pizza", "m.markup": [{"body": "Pizza 🍕"}]},
        {"m.id": "poutine", "m.markup": [{"body": "Poutine 🍟"}]},
        {"m.id": "italian", "m.markup": [{"body": "Italian 🍝"}]},
        {"m.id": "wings", "m.markup": [{"body": "Wings 🔥"}]},
      ]
    }
  }
}
```

!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!! TODO: EDIT BEYOND THIS LINE                                                              !!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

As mentioned above, this is already making use of Extensible Events: The fallback for clients which don't
know how to render polls is to just post the message to the chat. Some of the properties also make use of
extensible events within them, such as the `question` and the elements of `answers`: these are essentially
nested events themselves. For example, the following can represent the same `question`:

```json
{
  "question": {
    "m.text": "How are you?"
  }
}
```
```json5
{
  "question": {
    // a plaintext format is always required
    "m.text": "How are you?",
    "m.html": "<b>How are you?</b>"
  }
}
```
```json5
{
  "question": {
    "m.message": [
      // a plaintext format is always required
      {"body": "How are you?", "mimetype": "text/plain"},
      {"body": "<b>How are you?</b>", "mimetype": "text/html"},
    ]
  }
}
```

The `kind` refers to whether the poll's votes are disclosed while the poll is still open. `m.poll.undisclosed`
means the results are revealed once the poll is closed. `m.poll.disclosed` is the opposite: the votes are
visible up until and including when the poll is closed. Custom values are permitted using the standardized
naming convention are supported. Unknown values are to be treated as `m.poll.undisclosed` for maximum
compatibility with theoretical values. More specific detail as to the difference between two polls come up
later in this MSC.

`answers` must be an array with at least 1 option and are truncated to 20 options. Polls with fewer than 1
option should not rendered, and only the first 20 options are considered for rendering. Most polls are
expected to have 2-8 options. The answer `id` is an arbitrary string used within the poll events. Clients
should not attempt to parse or understand the `id`.

`max_selections` is optional and denotes the maximum number of responses a user is able to select. Users
can select fewer options, but not more. This defaults to `1`. Cannot be less than 1.

The `m.message` fallback should be representative of the poll, but is not required and has no mandatory
format. Clients are encouraged to be inspired by the example above when sending poll events.

To respond to a poll, the following event is sent:

```json5
{
  "type": "m.poll.response",
  "sender": "@bob:example.org",
  "content": {
    "m.relates_to": { // from MSC2674: https://github.com/matrix-org/matrix-doc/pull/2674
      "rel_type": "m.reference", // from MSC3267: https://github.com/matrix-org/matrix-doc/pull/3267
      "event_id": "$poll"
    },
    "m.poll.response": {
      "answers": ["poutine"]
    }
  },
  // other fields that aren't relevant here
}
```

Like `m.poll.start`, this `m.poll.response` event supports Extensible Events. However, it is strongly discouraged
for clients to include renderable types like `m.text` and `m.message` which could impact the usability of
the room (particularly for large rooms with lots of responses).

The response event forms a reference relationship with the poll start event. This kind of relationship doesn't
easily allow for server-side aggregation, however the alternatives section goes into detail as to why this
isn't a requirement for Polls.

Only a user's latest response event (by `origin_server_ts`) will be considered by clients. If that response
is after the poll has closed, the user is considered to have not voted. Votes are accepted until the poll
is closed (according to the `origin_server_ts` on the end/closure event).

The `answers` array in the response is the user's selection(s) for the poll. The array length is truncated
to `max_selections` length during processing. The entries are the `id` of each answer from the original poll
start event. If *any* of the supplied answers is unknown, or the field is otherwise invalid, then the user's
vote is spoiled. Spoiled votes are also how users can "un-vote" from a poll - redacting, or setting `answers`
to an empty array, will spoil that user's vote.

Votes are accepted until the poll is closed according to timestamp: servers/clients which receive votes
which are timestamped before the close event's timestamp (or, when no close event has been sent) are valid.
Late votes should be ignored. Early votes (from before the start event) are considered to be valid for the
sake of handling clock drift as gracefully as possible.

Only the poll creator or anyone with a suitable power level for redactions can close the poll. The rationale
for using the redaction power level is to help aid moderation efforts: while moderators can just redact the
original poll and invalidate it entirely, they might prefer to just close it and leave it on the historical
record.

Closure events which are sent by users without appropriate permission are ignored. A poll is considered
closed once the first valid closure event is received - repeated closures are ignored.

Closing a poll is done as follows:

```json5
{
  "type": "m.poll.end",
  "sender": "@bob:example.org",
  "content": {
    "m.relates_to": {
      "rel_type": "m.reference", // from MSC3267: https://github.com/matrix-org/matrix-doc/pull/3267
      "event_id": "$poll"
    },
    "m.poll.end": {},
    "m.text": "The poll has ended. Top answer: Poutine 🍟"
  },
  // other fields that aren't relevant here
}
```

Once again, Extensible Events make an appearance here. There's nothing in particular metadata wise that
needs to appear in the `m.poll.end` property of `content`, though it is included for future capability. The
backup `m.text` representation is for fallback purposes and is completely optional with no strict format
requirements: the example above is just that, an example of what a client *could* do. Clients should be
careful to include a "top answer" in the end event as server lag might allow a few more responses to get
through while the closure is sent. Votes sent on or before the end event's timestamp are valid votes - all
others must be disregarded by clients.

**Rationale**: Although clock drift is possible, as is clock manipulation, it is not anticipated that
polls will be closed while they are still receiving high traffic. There are some cases where clients might
apply local timers to auto-close polls, though these are typically used in extremely high traffic cases
such as Twitch-style audience polls - rejecting even 100 responses is unlikely to significantly affect
the results. Further, if a server were to manipulate its clock so that poll responses are sent after the
poll was closed, but timestamped for when it was open, the server is violating a social contract and likely
will be facing a ban from the room. This MSC does not propose a mitigation strategy beyond telling people
not to ruin the fun. Also, don't use polls for things that are important.

Clients should disable voting interactions with polls once they are closed. Events which claim to close
the poll from senders other than the creator are to be treated as invalid and thus ignored.

### Disclosed versus undisclosed polls

Disclosed polls are most similar to what is seen on Twitch and often Twitter: members of the room are able
to see the results and vote accordingly. Clients are welcome to hide the poll results until after the user
has voted to avoid biasing the user.

Undisclosed polls do track who voted for what, though don't reveal the results until the poll has been
closed, even after a user has voted themselves. This is enforced visually and not by the protocol given
the votes are sent to the room for local tallying - this is considered more of a social trust issue than
a technical one. This MSC expects that rooms (and clients) won't spoil the results of an undisclosed poll
before it is closed.

In either case, once the poll ends the results are shown regardless of kind. Clients might wish to avoid
disclosing who voted for what in an undisclosed poll, though this MSC leaves that at just a suggestion.

### Client implementation notes

Clients should rely on [MSC3523](https://github.com/matrix-org/matrix-doc/pull/3523) and
[MSC2675](https://github.com/matrix-org/matrix-doc/pull/2675) for handling limited ("gappy") syncs. The
relations endpoint can give (paginated) information about which results have been selected and when the
poll has closed, overriding any stale local state the client might have.

For clarity: clients using [MSC3523](https://github.com/matrix-org/matrix-doc/pull/3523) should use the
time-based shape of the endpoint, not the event ID shape, in order to honour the poll rules.

### Notifications

In order to have polls behave similar to message events, the following underride push rules are defined.
Note that the push rules are mirrored from those available to `m.room.message` events.

```json
{
  "rule_id": ".m.rule.poll_start_one_to_one",
  "default": true,
  "enabled": true,
  "conditions": [
    {"kind": "room_member_count", "is": "2"},
    {"kind": "event_match", "key": "type", "pattern": "m.poll.start"}
  ],
  "actions": [
    "notify",
    {"set_tweak": "sound", "value": "default"}
  ]
}
```

```json
{
  "rule_id": ".m.rule.poll_start",
  "default": true,
  "enabled": true,
  "conditions": [
    {"kind": "event_match", "key": "type", "pattern": "m.poll.start"}
  ],
  "actions": [
    "notify"
  ]
}
```

```json
{
  "rule_id": ".m.rule.poll_end_one_to_one",
  "default": true,
  "enabled": true,
  "conditions": [
    {"kind": "room_member_count", "is": "2"},
    {"kind": "event_match", "key": "type", "pattern": "m.poll.end"}
  ],
  "actions": [
    "notify",
    {"set_tweak": "sound", "value": "default"}
  ]
}
```

```json
{
  "rule_id": ".m.rule.poll_end",
  "default": true,
  "enabled": true,
  "conditions": [
    {"kind": "event_match", "key": "type", "pattern": "m.poll.end"}
  ],
  "actions": [
    "notify"
  ]
}
```

Servers should keep these rules in sync with the `m.room.message` rules they are based upon. For
example, if the `m.room.message` rule gets muted in a room then the associated rules for polls would
also get muted. Similarly, if either of the two poll rules were to be muted in a room then the other
poll rule and the `m.room.message` rule would be muted as well.

Clients are expected to not require any specific change in order to support these rules. Their user
experience typically already considers an entry for "messages in the room", which is what a typical
user would expect to control notifications caused by polls.

The server-side syncing of the rules additionally means that clients won't have to manually add support
for the new rules. Servers as part of implementation will update and incorporate the rules on behalf
of the users and simply send them down `/sync` per normal - clients which parse the push rules manually
shouldn't have to do anything as the rule will execute normally.

## Potential issues

As mentioned, poll responses are sent to the room regardless of the kind of poll. For open polls this
isn't a huge deal, but it can be considered an issue with undisclosed polls. This MSC strongly considers
the problem a social one: users who are looking to "cheat" at the results are unlikely to engage with the
poll in a productive way in the first place. And, of course, polls should never be used for something
important like electing a new leader for a country.

Poll responses are also de-anonymized by nature of having the sender attached to a response. Clients
are strongly encouraged to demonstrate anonymization by not showing who voted for what, but should consider
warning the user that their vote is not anonymous. For example, saying "22 total responses, including
from TravisR, Matthew, and Alice" before the user casts their own vote.

Limiting polls to client-side enforcement could be problematic if the MSC was interested in reliable
or provable votes, however as a chat feature this should reasonably be able to achieve user expectations.
Bolt-on support for signing, verification, validity, etc can be accomplished as well in the future.

The fallback support relies on clients already knowing about extensible events, which might not be
the case. Bridges (as of writing) do not have support for extensible events, for example, which can
mean that polls are lost in transit. This is perceived to be a similar amount of data loss when a Matrix
user reacts to an IRC user's message: the IRC user has no idea what happened on Matrix. Bridges, and
other clients, can trivially add message parsing support as described by extensible events to work
around this. The recommendations of this MSC specifically avoid the vote spam from being bridged, but
the start of poll and end of poll (results) would be bridged. There's an argument to be made for
surrounding conversation context being enough to communicate the results without extensible events,
though this is slightly less reliable.

Though more important for Extensible Events, clients might get confused about what they should do
with the `m.message` parts of the events. For absolute clarity: if a client has support for polls,
it can outright ignore any irrelevant data from the events such as the message fallback or other
representations that senders stick onto the event (like thumbnails, captions, attachments, etc).

The push rules for this feature are complex and not ideal. The author believes that it solves a short
term need while other MSCs work on improving the notifications system. Most importantly, the author
believes future MSCs which aim to fix notifications for extensible events in general will be a more
preferred approach over this MSC's (hopefully) short-term solution.

## Alternatives

The primary competition to this MSC is the author's own [MSC2192](https://github.com/matrix-org/matrix-doc/pull/2192)
which describes not only polls but also inline widgets. The poll implementation in MSC2192 is primarily
based around `m.room.message` events, using `msgtype` to differentiate between the different states. As
[a thread](https://github.com/matrix-org/matrix-doc/pull/2192/files#r514497274) notes on the MSC, this
is an awful experience on clients which do not support polls properly, leaving an irritating amount of
contextless messages in the timeline. Though not directly mentioned on that thread, polls also cannot be
closed under that MSC which leads to people picking options hours or even days after the poll has "ended".
This MSC instead proposed to only supply fallback on the start and end of a poll, leading to enough context
for unsupporting clients without flooding the room with messages.

Originally, MSC2192 was intended to propose polls as a sort of widget with access to timeline events
and other important information, however the widget infrastructure is just not ready for this sort of
thing to exist. First, we'd need to be able to send events to the widget which reference itself (for
counting votes), and allow the widget to self-close if needed. This is surprisingly difficult when widgets
can be "popped out" or have a link clicked in lieu of rendering (for desktop clients): there's no
communication channel back to the client to get the information back and forth. Some of this can be solved
with scoped access tokens for widgets, though at the time of writing those are a long ways out. In the
end, it's simply more effective to use Extensible Events and Matrix directly rather than building out
the widgets infrastructure to cope - MSC2192 is a demonstration of this, considering it ended up taking
out all the widget aspects and replacing them with fields in the content.

Finally, MSC2192 is simply inferior due to not being able to restrict who can post a poll. Responses
and closures can also be limited arbitrarily by room admins, so clients might want to check to make
sure that the sender has a good chance of being able to close the poll they're about to create just
to avoid future issues.

### Aggregations instead of references?

A brief moment in this MSC's history described an approach which used aggregations (annotations/reactions)
instead of the proposed reference relationships, though this had immediate concerns of being too
complicated for practical use.

While it is beneficial for votes to be quickly tallied by the server, the client still needs to do
post-processing on the data from the server in order to accurately represent the valid votes. The
server should not be made aware of the poll rules as it can lead to over-dependence on the server,
potentially causing excessive network requests from clients.

As such, the reference relationship is maintained by this proposal in order to remain consistent with
how the poll close event is sent: instead of clients having to process two paginated requests they can
use a single request to get the same information, but in a more valuable form.

For completeness, the approach of aggregations-based responses is summarized as:

* `m.annotation` `rel_type`
* `key` is an answer ID
* Multiple response events for multi-select polls. Only the most recent duplicate is considered valid.
* Unvoting is done through redaction.

Additional concerns are how the client needs to ensure that the answer IDs won't collide with a reaction
or other annotation, adding additional complexity in the form of magic strings.

## Security considerations

As mentioned a multitude of times throughout this proposal, this MSC's approach is prone to disclosure
of votes and has a couple abuse vectors which make it not suitable for important or truly secret votes.
Do not use this functionality to vote for presidents.

Clients should apply a large amount of validation to each field when interacting with polls. Event
bodies are already declared as completely untrusted, though not all clients apply a layer of validation.
In general, this MSC aims to try and show something of use to users so they can at least figure out
what the sender intended, though clients are also welcome to just hide invalid events/responses (with
the exception of spoiled votes: those are treated as "unvoting" or choosing nothing). Clients are
encouraged to try and fall back to something sensible, even if just an error message saying the poll
is invalid.

Users should be wary of polls changing their question after they have voted. Considering polls can be
edited, responses might no longer be relevant. For example, if a poll was opened for "do you like
cupcakes?" and you select "yes", the question may very well become "should we outlaw cupcakes?" where
your "yes" no longer applies. This MSC considers this problem more of a social issue than a technical
one, and reminds the reader that polls should not be used for anything important/serious at the moment.

## Future considerations

Some aspects of polls are explicitly not covered by this MSC, and are intended for another future MSC
to solve:

* Allowing voters/room members to add their own freeform options. The edits system doesn't prevent other
  members from editing messages, though clients tend to reject edits which are not made by the original
  author. Altering this rule to allow it on some polls could be useful in a future MSC.

* Verifiable or cryptographically secret polls. There is interest in a truly enforceable undisclosed poll
  where even if the client wanted to it could not reveal the results before the poll is closed. Approaches
  like [MSC3184](https://github.com/matrix-org/matrix-doc/pull/3184) or Public Key Infrastructure (PKI)
  might be worthwhile to investigate in a future MSC.

## Other notes

If a client/user wishes to make a poll statically visible, they should check out
[pinned messages](https://matrix.org/docs/spec/client_server/r0.6.1#m-room-pinned-events).

## Unstable prefix

While this MSC is not eligible for stable usage, the `org.matrix.msc3381.` prefix can be used in place
of `m.`. Note that extensible events has a different unstable prefix for those fields.

The 3 examples above can be rewritten as:

```json5
{
  "type": "org.matrix.msc3381.poll.start",
  "sender": "@alice:example.org",
  "content": {
    "org.matrix.msc3381.poll.start": {
      "question": {
        "org.matrix.msc1767.text": "What should we order for the party?"
      },
      "kind": "org.matrix.msc3381.poll.disclosed",
      "answers": [
        { "id": "pizza", "org.matrix.msc1767.text": "Pizza 🍕" },
        { "id": "poutine", "org.matrix.msc1767.text": "Poutine 🍟" },
        { "id": "italian", "org.matrix.msc1767.text": "Italian 🍝" },
        { "id": "wings", "org.matrix.msc1767.text": "Wings 🔥" }
      ]
    },
    "org.matrix.msc1767.message": [
      {
        "mimetype": "text/plain",
        "body": "What should we order for the party?\n1. Pizza 🍕\n2. Poutine 🍟\n3. Italian 🍝\n4. Wings 🔥"
      },
      {
        "mimetype": "text/html",
        "body": "<b>What should we order for the party?</b><ol><li>1. Pizza 🍕</li><li>2. Poutine 🍟</li><li>3. Italian 🍝</li><li>4. Wings 🔥</li></ol>"
      }
    ]
  },
  // other fields that aren't relevant here
}
```

```json5
{
  "type": "org.matrix.msc3381.poll.response",
  "sender": "@bob:example.org",
  "content": {
    "m.relates_to": { // from MSC2674: https://github.com/matrix-org/matrix-doc/pull/2674
      "rel_type": "m.reference", // from MSC3267: https://github.com/matrix-org/matrix-doc/pull/3267
      "event_id": "$poll"
    },
    "org.matrix.msc3381.poll.response": {
      "answers": ["poutine"]
    }
  },
  // other fields that aren't relevant here
}
```

```json5
{
  "type": "org.matrix.msc3381.poll.end",
  "sender": "@bob:example.org",
  "content": {
    "m.relates_to": {
      "rel_type": "m.reference",
      "event_id": "$poll"
    },
    "org.matrix.msc3381.poll.end": {},
    "org.matrix.msc1767.text": "The poll has ended. Top answer: Poutine 🍟"
  },
  // other fields that aren't relevant here
}
```

Note that the extensible event fallbacks did not fall back to `m.room.message` in this MSC: this
is deliberate to ensure polls are treated as first-class citizens. Client authors not willing/able
to support polls are encouraged to instead support Extensible Events for better fallbacks.
