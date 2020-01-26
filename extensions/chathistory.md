---
title: IRCv3 chathistory extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: "Evan Magaliff"
    period: "2017"
    email: "evan@muffinmedic.net"
  -
    name: "Darren Whitlen"
    period: "2018-2019"
    email: "darren@kiwiirc.com"
---
## Description
This document describes the format of the `chathistory` extension. This enables clients to request messages that were previously sent if they are still available on the server.

The server as mentioned in this document may refer to either an IRC server or an IRC bouncer.

## Implementation
The `chathistory` extension uses the [chathistory][batch/chathistory] batch type and introduces a client command, `chathistory`.

To fully support this extension, clients MUST support the [`batch`][batch], [`server-time`][server-time] and [`message-tags`][message-tags] capabilities.

The `chathistory` capability MUST be negotiated. This allows the server and client to act differently when delivering message history on connection.

An ISUPPORT token MUST be sent to the client to state the maximum number of messages a client can request in a single command, represented by an integer. `CHATHISTORY=50`. If `0`, the client SHOULD assume that there is no maximum number of messages.

The `event-playback` capability MAY be negotiated. This allows the client to signal that it is capable of receiving and correctly processing lines that would normally produce a local state change (such as `JOIN` or `MODE`) in its history batches.

### `CHATHISTORY` Command
`CHATHISTORY` content can be requested by the client by sending the `CHATHISTORY` command to the server. A `batch` MUST be returned by the server. If no content exists to return, an empty batch SHOULD be returned to avoid the client waiting for a reply and to indicate that no content is available.

The `chathistory` command uses the following general syntax structure:

    CHATHISTORY <subcommand> <target> <timestamp | msgid> <limit>

The `target` parameter specifies a single buffer (channel or nickname) from which history SHOULD be retrieved. Any `timestamp` values or parameters MUST be in the format of `YYYY-MM-DDThh:mm:ss.sssZ`.

If a nickname is given as the `target` then the server SHOULD include history sent between both the current user and the target nickname as to give a full conversation. The server SHOULD attempt to include history involving other nicknames if either the current user or the target nickname has changed during the requested timeframe.

#### Subcommands

The following subcommands are used to describe how the server should return messages relative to the `timestamp` or `msgid` given. In all cases, at most one timestamp or msgid MUST be given, never both. The number of messages returned must be equal to or less than `limit`.

#### `BEFORE`
    CHATHISTORY BEFORE <target> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ> <limit>
Request messages before and including the given `timestamp`.

#### `AFTER`
    CHATHISTORY AFTER <target> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ> <limit>
Request number of messages after and including the given `timestamp`.

#### `LATEST`
    CHATHISTORY LATEST <target> <* | timestamp=YYYY-MM-DDThh:mm:ss.sssZ> <limit>
Request the most recent messages that have been sent. If a `timestamp` or `msgid` is given, restrict to messages sent after and including that time or message; if a `*` is given, no such restriction applies.

This is useful for retrieving the latest conversation when first joining a channel or opening a query buffer.

#### `AROUND`
    CHATHISTORY AROUND <target> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ | msgid=1234> <limit>
Request a number of messages before, including, and after the `timestamp` or `msgid`. The implementation may decide how many messages to include before and after (up to a total of `limit`).

This is useful for retrieving conversation context around a single message.

#### `BETWEEN`
    CHATHISTORY BETWEEN <target> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ> <timestamp=YYYY-MM-DDThh:mm:ss.sssZ> <limit>
Request up to `limit` number of messages between the given `timestamp` or `msgid` values. The returned messages MUST start from the inclusive first message selector, while including and finishing on the second - this may be forwards or backwards in time.

#### Returned message notes
The returned messages MUST be in ascending time order and the `server-time` tag SHOULD be the time at which the message was received by the IRC server. The `msgid` tag that identifies each individual message in a response MUST be the `msgid` tag as originally sent by the IRC server.

If the client has not negotiated the `event-playback` capability, the server MUST NOT send any lines other than `PRIVMSG` and `NOTICE` in the reply batch. If the client has negotiated `event-playback`, the server SHOULD send additional lines relevant to the chat history, including but not limited to `TAGMSG`, `JOIN`, `PART`, `QUIT`, `MODE`, `TOPIC`, and `NICK`.

#### Errors and Warnings
Errors are returned using the standard replies syntax.

If the server receives a `CHATHISTORY` command with an unknown subcommand, the `UNKNOWN_COMMAND` error code MUST be returned.
> FAIL CHATHISTORY UNKNOWN_COMMAND the_given_command :Unknown command

If the server receives a `CHATHISTORY` command with missing parameters, the `NEED_MORE_PARAMS` error code MUST be returned.
> FAIL CHATHISTORY NEED_MORE_PARAMS the_given_command :Missing parameters

If no message history can be returned due to an error, the `MESSAGE_ERROR` error code SHOULD be returned.
> FAIL CHATHISTORY MESSAGE_ERROR the_given_command the_given_target [extra_context] :Messages could not be retrieved

### Examples

Requesting the latest conversation upon joining a channel
~~~~
[c] CHATHISTORY LATEST #channel * 50
[s] :irc.host BATCH +ID chathistory #channel
[s] @batch=ID;msgid=1234;time=2019-01-04T14:33:26.123Z :nick!ident@host PRIVMSG #channel :message
[s] @batch=ID;msgid=1235;time=2019-01-04T14:33:38.123Z :nick!ident@host NOTICE #channel :message
[s] @batch=ID;msgid=1238;time=2019-01-04T14:34:17.123Z;+client_tag=val :nick!ident@host PRIVMSG #channel :ACTION message
[s] :irc.host BATCH -ID
~~~~

Requesting further message history than our client currently has
~~~~
[c] CHATHISTORY BEFORE bob timestamp=2019-01-04T14:34:17.123Z 50
[s] :irc.host BATCH +ID chathistory bob
[s] @batch=ID;msgid=1234;time=2019-01-04T14:34:09.123Z :bob!ident@host PRIVMSG alice :hello
[s] @batch=ID;msgid=1235;time=2019-01-04T14:34:10.123Z :alice!ident@host PRIVMSG bob :hi! how are you?
[s] @batch=ID;msgid=1238;time=2019-01-04T14:34:16.123Z; :bob!ident@host PRIVMSG alice :I'm good, thank you!
[s] :irc.host BATCH -ID
~~~~

## Use Cases
The batch type and supporting command are useful for allowing an "infinite scroll" type capability within the client. A client will, upon scrolling to the top of the active window or a manual trigger, may request `chathistory` from the server and, after receiving returned content, append it to the top of the window. Users can repeat this historic scrolling to retrieve prior history until limitations are met (see below).

Upon joining a channel, a client may request the latest messages for the buffer so that the active conversation context may be retrieved.

### Client pseudocode

A client with full support for BATCH, message IDs, and deduplication can fill in gaps in its history using the following pseudocode. `FUZZ_INTERVAL` is a constant that compensates for clock skew across the IRC network (perhaps 1 to 10 seconds):

    lower_bound = <timestamp of last message relayed to the previous session>
    lower_bound -= FUZZ_INTERVAL
    upper_bound = now() + FUZZ_INTERVAL
    retrieved_count = 0
    while retrieved_count < SANITY_LIMIT and lower_bound < upper_bound:
        messages = CHATHISTORY(BETWEEN, upper_bound, lower_bound)
        messages = deduplicate(messages)
        if len(messages) == 0:
            break
        display(messages)
        upper_bound = min(msg.timestamp for msg in messages)
        retrieved_count += len(messages)

A client without support for BATCH, message IDs, or deduplication can still make use of CHATHISTORY, albeit with the possibility of skipping some messages or seeing some duplicated messages. For example, on initial JOIN, the client can do the following (this implementation errs on the side of missing messages, rather than seeing duplicated messages):

    lower_bound = <timestamp of last message relayed to the previous session>
    lower_bound += ONE_MILLISECOND
    display(CHATHISTORY(LATEST, lower_bound))

Infinite scroll can be implemented as:

    lower_bound = <timestamp of last message relayed to the previous session>
    lower_bound += ONE_MILLISECOND
    upper_bound = <lowest timestamp of any message retrieved during current session>
    upper_bound -= ONE_MILLISECOND
    if lower_bound < upper_bound:
        display(CHATHISTORY(BETWEEN, upper_bound, lower_bound))

## Implementation Considerations

In the typical IRC network, there is no well-defined global linear ordering of messages, since different linked servers may see messages in different orders. Furthermore, due to clock skew between servers and between server and client, messages may be delivered in an order that differs from the timestamp order. Clients should take this into account when requesting and displaying history.

## Security Considerations
Secure identification of users and clients MUST exist in order to ensure that users cannot obtain history they are not authorised to view. Use of account names, internal account identifiers, or certificate fingerprints SHOULD be strongly considered when matching content to users. If a client requests content for a target that they do not have permission for, eg. a channel they are banned from, an empty batch SHOULD be returned as if no content exists.

While an ISUPPORT token value of `0` may be used to indicate no message limit exists, servers SHOULD set and enforce a reasonable maximum and properly throttle `CHATHISTORY` commands to prevent abuse.
