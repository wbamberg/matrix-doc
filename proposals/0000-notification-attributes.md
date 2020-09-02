# MSC0000: Event notification attributes and actions

## Background: the problems with push rules

["Push rules"](https://matrix.org/docs/spec/client_server/r0.6.1#push-rules)
provide an interface which allows Matrix clients and users to
configure various actions to be performed on receipt of certain events, such as
showing a pop-up notification or playing a sound.

Originally, push rules were intended to be interpreted only by homeservers
(they would direct the server how to "push" the event to mobile devices via a
push gateway). Later, as encrypted events became more common, it also became
necessary for clients to be able to process push rules.

Experience has shown that push rules do not have enough flexibility to provide
the features expected by users. This proposal suggests an alternative data
model and API set which could be used for configuring notifications.

## Proposal

We break down the notifications process into two distinct phases. Firstly, the
homeserver (or client, in the case of encrypted events) analyses each event and
assigns a set of _attributes_ to the events. Secondly, the homeserver/client
follows a set of rules which define which _actions_ should be performed for a
given set of attributes - where the rules can vary by room.

### Event notification attributes

For each user on a homeserver, the homeserver analyses each event, and assignes
zero or more *event notification attributes* for that user. For encrypted
events, the analysis is carried out by the client instead.

Event notification attributes consist of a identifiers matching the [common
namespaced identifier
grammar](https://github.com/matrix-org/matrix-doc/pull/2758). The algorithm for
assigning each attribute is defined in detail in following sections; a summary
follows here:

 * `m.keyword`: The event contains one of the user's registered "notification
   keywords".
 * `m.mention`: The event contains a "mention" of the user's userid, etc.
 * `m.invite`: The event is an invitation to a room.
 * `m.room_upgrade`: The event is a notification that a room has been upgraded.
 * `m.voip_call`: The event is an invitation to a VoIP call.
 * `m.dm`: The event was in a Direct Message room.
 * `m.msg`: The event contains a visible body.

### APIs for configuring the assignment of event notification attributes

The assignment of event notification attributes has a limited amount of
configuration. This information is stored in an
[`account_data`](https://matrix.org/docs/spec/client_server/r0.6.1#id125) event
of type `m.notification_attribute_data`, and with content with the following
structure:

XXX case-sensitivity?

```json
{
  "keywords": ["foo", "bar", "longer string"],
  "mentions": {
     "displayname": true,
     "mxid": true,
     "room_notif": true,
  }
}
```

This event can be set at the global level. (Room-level overrides will be a
future extension).

The fields are defined as follows:

 * `keywords`: an array of strings which form "notification keywords". See
   `m.keyword` below. If this field is absent, the list is implicitly empty.

 * `mentions`: an object containing boolans which define which events should
   qualify for `m.mention` attributes. See `m.mention` for more details.

   Each of the defined fields defaults to `false` if not present. If the whole
   `mentions` field is absent, all the inner fields take their default values.

Homeservers should check the content of any uploaded
`m.notification_attribute_data` events, and reject any malformed events with a
400 error and `M_BAD_JSON`. However, implementations should be resilient to
malformed data and behave "sensibly" (for example: the user might upload
malformed data to an older server version which does not yet support the
required checks).

A number of APIs are defined to allow the configuration data to be atomically
manipulated and interrogated at a finer level. These are defined as follows.

#### `GET /_matrix/client/r0/notification_attribute_data/global`

The body of the response to this request is simply the current global
`m.notification_attribute_data` account data event.

#### `GET /_matrix/client/r0/notification_attribute_data/global/keywords`

The body of the response to this request is the user's current global keyword
list, as follows:

```json
{
  "keywords": ["foo", "bar", "longer string"]
}
```

The `keywords` field is always present, even if the list is empty.

#### `PUT /_matrix/client/r0/notification_attribute_data/global/keywords`

This endpoint replaces the user's global keyword list. The request body takes
the form:

```json
{
  "keywords": ["foo", "bar", "longer string"]
}
```

It is invalid to make this request with no `keywords` field.

On success, the response is an empty object:

```json
{}
```


#### `GET /_matrix/client/r0/notification_attribute_data/global/mentions`

The body of the response to this request is the user's current global mentions
settings, as follows:

```json
{
  "mentions": {
     "displayname": true,
     "mxid": true,
     "room_notif": true,
  }
}
```

#### `PUT /_matrix/client/r0/notification_attribute_data/global/mentions`

This endpoint updates the user's global mentions settings. The request body takes
the form:

```json
{
  "mentions": {
     "displayname": true,
     "mxid": true,
     "room_notif": true,
  }
}
```

Any fields that are absent are reset to their defaults (ie, `false`). It is
invalid to omit the entire `mentions` field.

On success, the response is an empty object:

```json
{}
```





## Potential issues

*Not all proposals are perfect. Sometimes there's a known disadvantage to implementing the proposal,
and they should be documented here. There should be some explanation for why the disadvantage is
acceptable, however - just like in this example.*

Someone is going to have to spend the time to figure out what the template should actually have in it.
It could be a document with just a few headers or a supplementary document to the process explanation,
however more detail should be included. A template that actually proposes something should be considered
because it not only gives an opportunity to show what a basic proposal looks like, it also means that
explanations for each section can be described. Spending the time to work out the content of the template
is beneficial and not considered a significant problem because it will lead to a document that everyone
can follow.


## Alternatives

### Extend use of push rules

Before this proposal, we gave considerable thought to how we might extend the
use of Push Rules to meet user expectations.

Whilst in theory they are very flexible,
it is often necessary to use tens of rules to express simple-sounding
requirements. To give some examples:

 * Suppose the user wants to configure certain rooms to be "highlighted" when
   their name is mentioned, but they do not want to hear an audible alert in
   those rooms. (In other rooms, they want both a highlight *and* and audible
   alert.)

   "Mentions" are implemented by eight separate push rules, including rules to
   filter out mentions in `m.room.notice` and edit events. Each "keyword" gets
   its own rule too. So for that one room, where we want to change the
   behaviour of mentions and keywords, we would have to repeat each of those
   8+N rules.

 * Suppose the user configures a particular room so that they receive a visible
   notification and an audible alert for each message. Later, the user disables
   audible alerts at the account level; this means that the account-level
   "mention" and "keyword" rules are reconfigured so that there is no audible
   alert.

   The problem concerns how to handle the user's original room-level
   configuration. There are two unsatisfactory options:

   * Leaving the room configuration in place would mean that they would mean
     that the user would get audible notifications for all messages in that
     room *except* mentions and keywords.

   * Removing the room configuration would at least give consistent behaviour,
     but it will be surprising for the user that their room-level configuration
     is affected by account-level configuration.

Whilst it is *possible* to create sets of push rules that meet the user's
expectations, it then becomes very difficult to go from the long list of rules
back to the user's intentions to populate a configuration interface. This is
further exacerbated by the fact that different clients might take the same set
of user intentions and populate the push rules in slightly different ways.

Long, detailed lists of push rules would also make it very difficult to extend
Matrix in the future without breaking users' existing push rules. For example:
a currently proposed change to the push rules implementation is that
`m.reaction` events should not cause a visible notification by default (see
[MSC2153](https://github.com/matrix-org/matrix-doc/pull/2153)). This would be
an order of magnitude harder to do if each user had hundreds of different
copies of the default push rules, fine-tuning the behaviour for each room.


## Security considerations

*Some proposals may have some security aspect to them that was addressed in the proposed solution. This
section is a great place to outline some of the security-sensitive components of your proposal, such as
why a particular approach was (or wasn't) taken. The example here is a bit of a stretch and unlikely to
actually be worthwhile of including in a proposal, but it is generally a good idea to list these kinds
of concerns where possible.*

By having a template available, people would know what the desired detail for a proposal is. This is not
considered a risk because it is important that people understand the proposal process from start to end.

## Unstable prefix

The following mapping will be used for identifiers in this MSC during
development:

Proposed final identifier       | Purpose | Development identifier
------------------------------- | ------- | ----
`m.notification_attribute_data` | account_data event type |
`/_matrix/client/r0/notification_attribute_data` | API endpoint |
