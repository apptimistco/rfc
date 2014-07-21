# App-Server Protocol #
This RFC describes the protocols between an Apptimist App and its cloud
server(s).

<!--
         1         2         3         4         5         6         7         8
12345678901234567890123456789012345678901234567890123456789012345678901234567890
-->

**Status:** This is in development and may likely be rambling thoughts for a
while.

## Description ##
The Apptimist App-Server protocol is an exchange of encapsulated data units
between the App and Server along with a few, server resident access control
files. The Apps themselves extend this protocol with further definition of the
encapsulated `message` and other server resident data files.

## Notation ##
In this document, *"device"* refers to the App running on the user's IOS or
Android device whereas *"service"* refers to programs running on an Apptimist
(potentially cloud based) server.

This document describes a protocol data unit (PDU) as concatenated components.

    xyz = X Y Z

The double-quote encapsulated components denote UTF-8 strings. Unlike the
C-language, such strings are not null terminated.

Comments are introduced with a C++ style double slash.

Numeric literals may be C like decimal or "0x" prefaced hexadecimal digits.

Types are described as:

    [ "[" [ number ] "]" ] u{8,16,32,64} [ "{" value "}" ]`

The elemental component type is an 8, 16, 32, or 64-bit, big-endian, unsigned
integer with keyword u8, u16, u32 and u64 respectively.

An array component has a bracket-bound numbered preface indicating a sequence
of a single element type. Successive bracket-bound number prefaces indicate
multi-dimensional arrays. The number may be absent indicating a possible zero
length array defined by the number in another component.  Examples:

    to = [32]u8
    recipients = u8
    recipientList = [][32]u8

A composite literal component appends a brace-bond list of composite elements.
Examples:

    version = u8{1}
    nonce = [24]u8{ id seq }
    id = [64]u8
    seq = u64
    nameLen = u8{ name.Length() }
    name []u8

## WebSockets ##
The Apptimist protocol runs through non-secure WebSockets described in
(http://tools.ietf.org/html/rfc6455). These are Web vs. TCP or UDP Sockets to
provide the most flexible service through carrier proxy servers and enterprise
firewalls. Also, these are non-secure HTTP sockets instead of the SSL secured
HTTPS because the Apptimist protocol includes (http://ed25519.cr.yp.to/)
authenticated signatures and (http://nacl.cr.yp.to/box.html) authenticated
encryption to provide a user assigned web-of-trust rather than a centralized
certificate authority.

### URI ###

    URI = "ws://" host [ ":" port ] path [ "?" query ]

The initial `host` for all Apptimist services has this registered domain name
format.

    host = app ".apptimist.co"

Where `app` is the service name; for example, "siren".

After login, the service may be redirected to a client subset `host`; for
example, "alpha.siren.apptimist.co". Some service features may direct the
client to establish simultaneous WebSockets to shared services; for example,
"video-conference.apptimist.co"

The `port` component is optional; the default is 80, but may be different for
testing.

The `path` component for most services is simply a slash prefaced service
name; i.e "/siren".

The `query` component is a 64 digit, hexadecimal string representing the
device ephemeral, 32-octet, public encryption key.

## PDU ##
The Apptimist PDU has separately encrypted header and data components.
Generally, the data is encrypted by the originator with the key of the
recipient; whereas the header is encrypted with the immediate sender/receiver
key pair and may be re-encrypted at each hop of the server network.

    pdu = headerLen dataLen eHeader eData

    headerLen = u32{ eHeader.Length() }
    dataLen = u32{ eData.Length() }

    eHeader = crypto_box(header, headerNonce, rcvrPubEncrKey, sndrSecEncrKey)

    header = []u8{ version type ... }
    vsersion = u8{ 1 }
    type = u8

    headerNonce = [24]u8{ headerNoncePrefix seq }
    headerNoncePrefix = [22]u8{ dataNonce[:22] }
    seq = u16

    rcvrPubEncrKey = [32]u8
    sndrSecEncrKey = [32]u8

    eData = crypto_box(data, dataNonce, toPubEncrKey, fromSecEncrKey)
    dataNonce = [24]u8{ ed25519(serviceSecAuthKey, servicePubEncrKey)[:24] }

    toPubEncrKey = [32]u8
    fromSecEncrKey = [32]u8

Every header begins with `version` and `type` components. As indicated, the
most recent `version` is 1. Future versions will be numeric increments and,
when possible, may be backwards compatible.

The `seq` component of the `headerNonce` is a sequential counter following the
NaCL *"Security model"* whereby this increments by 2 for each PDU sent between
socket endpoints for the connection duration.  The side with the
lexicographically smaller public key sends its first pdu with a 1 `seq`, 3 on
the second pdu, 5 on the third, etc.; meanwhile, the lexicographically larger
public key peer uses `seq` 2, 4, 6, etc.

The `headerNoncePrefix` is initially the first 22 bytes of `dataNonce`.

**TBD** whether we then reset the `headerNoncePrefix` from a signed PIN.

The `dataNonce` is the first 24 bytes of the service self signed public
encryption key.

### Types ###

    LOGIN_PDU            1
    REDIRECT_PDU         2
    NEW_PDU              3
    DEL_PDU              4
    MESSAGE_PDU          5
    TRUNCATE_PDU         6
    PAUSE_PDU            7
    RESUME_PDU           8
    VOUCH_PDU            9
    VOUCHERS_PDU        10
    SEARCH_BY_NAME_PDU  11
    SEARCH_BY_UID_PDU   12
    SEARCH_BY_KEY_PDU   13
    USER_PDU            14
    READ_PDU            15
    LOCK_PDU            16
    WRITE_PDU           17
    REMOVE_PDU          18

### login ###
The device sends this PDU to restart service for an existing user.

    loginHeader = version type userPubEncrKey signature max head

    version = u8{ 1 }      // see above
    type = u8{ LOGIN_PDU }

    userPubEncrKey = [32]u8
    signature = [64]u8{ ed25519(userSecAuthKey, userPubEncrKey) }

    max = u16
    head = [64]u8

The `head` is the last message received by the device. A zero value `head`
requests that the service to send all messages.

The message history may be limited by `max`, whereby a non-zero `max` governs
the number of queued messages the service searches backward for device's given
`head`.

* This PDU has no `data`.

### redirect ###
The service may send this PDU at anytime to direct the device to immediately
terminate the current WebSocket and reconnect to the given URI.

    redirectHeader = version type uriLen uri

    version = u8{ 1 }      // see above
    type = u8{ REDIRECT_PDU }

    uriLen = u16{ uri.Length() }
    uri = []u8

* This PDU has no `data`.

### new ###
The device sends a `new` PDU to create a service account for the given user.
They must then follow with a `login` PDU to start service.

An existing, logged-in user may also send a `new` PDU to create an event or
forum.

    newHeader = version type pubEncrKey pubAuthKey nameLen uidLen tokenLen name uid token

    version = u8{ 1 }      // see above
    type = u8{ NEW_PDU }

    pubEncrKey = [32]u8
    pubAuthKey = [32]u8

    nameLen = u8{ name.Length() }
    uidLen = u8{ uid.Length() }
    tokenLen = u8{ token.Length() }

    name = []u8
    uid = []u8
    token= []u8

If the device hasn't yet logged-in, the service creates a new account with the
given `name` and `pubEncrkey` for FaceBook authenticated `uid`, `token` pairs.
If FaceBook doesn't authenticate the user, the service ignores the socket for
a random period before closing.

If the device has already has logged-in with a valid`login` PDU, it may send
one or more `new` PDUs to create events and forums. In this case, both `uid`
and `token` are ignored.

* This PDU has no `data`.

### del ###
The device sends this PDU to delete an event or forum. They may only delete
those event in which they are a registered author or editor.

    delHeader = version type pubEncrKey
    version = u8{ 1 }      // see above
    type = u8{ DEL_PDU }

    pubEncrKey = [32]u8

* This PDU has no `data`.
* This PDU is ignored by the service unless the device has previously
  logged-in.

### message ###
This `message` header prefaces encrypted data sent by a user to one or more
other users; by a user to a forum; and by the service delivering a user
message or forum message.

    messageHeader = version type recipientPubEncrKey toPubEncrKey fromPubEncrKey

    version = u8{ 1 }      // see above
    type = u8{ MESSAGE_PDU }

    time = u64

    recipientPubEncrKey = [32]u8
    toPubEncrKey = [32]u8
    fromPubEncrKey = [32]u8

For messages sent to events or forums, both the header and data is encrypted
with the public key of the forum. When sent from the user, the header has the
forum public key for both `recipientPubEncrKey` and `toPubEncrKey`. Upon
receipt, the service replicates these messages to each subscriber by
re-encrypting the data with the subscriber's public key, reseting
`recipientPubEncrKey` to the subscriber; then encrypting the message header
with the subscriber key.

The `message` includes its origination `time` in elapsed seconds since January
1, 1970 UTC.

The `data` format is APP specific; it likely includes a header identifying
content type and recipients. It may also include information intended for
other devices of the same user; for example, an APP may send a message to it's
user to forward message flag changes to other devices.  In addition,
event/forum messages may be routed through select moderators.

Messages are identified with a checksum of the concatenated decrypted header
and encrypted data.

    messageID = [64]u8{ sha512(messageHeader eData) }

* This PDU is ignored by the service unless the device has previously
  logged-in.

### truncate ###
This PDU is sent by the device to `truncate` the identified message. It is
then echo by the server to acknowledge truncation and notify the user's other
devices to truncate cached copies.

    truncateHeader = version type messageID

    version = u8{ 1 }      // see above
    type = u8{ TRUNCATE_PDU }

    messageID = [64]u8

* This PDU has no `data`.
* This PDU is ignored by the service unless the device has previously
  logged-in.

### pause ###
The device sends a `pause` PDU to suspend the service message stream.

    pauseHeader = version type
    version = u8{ 1 }      // see above
    type = u8{ PAUSE_PDU }

* This PDU has no `data`.
* This PDU is ignored by the service unless the device has previously
  logged-in.

### resume ###
The device sends a `pause` PDU to resume the service message stream.

    resumeHeader = version type

    version = u8{ 1 }      // see above
    type = u8{ RESUME_PDU }

* This PDU has no `data`.
* This PDU is ignored by the service unless the device has previously
  logged-in.

### vouch ###
The device sends this PDU to `vouch` for given user. To revoke an earlier
voucher, the device sends this PDU with a nil `signature`.

    vouchHeader = version type subjectPubEncrKey signature

    version = u8{ 1 }      // see above
    type = u8{ VOUCH_PDU }

    subjectPubEncrKey = [32]u8
    signature = [64]u8{ ed25519(userSecAuthKey, subjectPubEncrKey) }

* This PDU has no `data`.
* This PDU is ignored by the service unless the device has previously
  logged-in.

### vouchers ###
The device sends this PDU with a zero `count` and empty `voucherPubEncrKeys`
to request that the service respond with the public keys of users that have
vouched for the given subject.

    vouchersHeader = version type subjectPubEncrKey count voucherPubEncrKeys

    version = u8{ 1 }      // see above
    type = u8{ VOUCHERS_PDU }

    subjectPubEncrKey = [32]u8

    count = u32
    voucherPubEncrKeys = [count][32]u8

* This PDU has no `data`.
* This PDU is ignored by the service unless the device has previously logged-in.

### searchByName, searchByUid, searchByKey ###
The device sends these PDUs to search for users with the given name, FB user
identifier, or public encryption key. The service responds with one or more
`user` matching PDUs as limited by `maxUsers` for by-name and by-uid searches
unless this maximum is zero.

    searchByNameHeader = version type nameLen name

    version = u8{ 1 }      // see above
    type = u8{ SEARCH_BY_NAME_PDU }

    maxUsers = u16

    nameLen = u8{ name.Length() }
    name = []u8


    searchByUidHeader = version type uidLen uid

    version = u8{ 1 }      // see above
    type = u8{ SEARCH_BY_UID_PDU }

    maxUsers = u16

    uidLen = u8{ uid.Length() }
    uid = []u8


    searchByKeyHeader = version type pubEncrKey

    version = u8{ 1 }      // see above
    type = u8{ SEARCH_BY_KEY_PDU }

    pubEncrKey = [32]u8

**TBD** syntax for regex name search.

* These PDUs have no `data`.
* These PDUs are ignored by the service unless the device has previously
  logged-in.

### user ###
The service responds to look-up requests with one or more `user` PDUs. The
`count` indicates how many `user` PDUs will follow constituting the matching
set for the requested look-up. The `count` of the last matching `user` is zero
indicating end-of-match. If there are no matching users, `count`, `nameLen`
and `uidLen` are all zero and the respective `name` and `uid` fields are
empty.

    userHeader = version type count pubEncrKey nameLen uidLen name uid

    version = u8{ 1 }      // see above
    type = u8{ USER_PDU }

    count = u16

    pubEncrKey = [32]u8

    nameLen = u8{ name.Length() }
    uidLen = u8{ uid.Length() }

    name = []u8
    uid = []u8

* This PDU has no `data`.
* This PDU is ignored by the service unless the device has previously logged-in.

### read ###
The device sends `read` to request the given user file. If the file is
accessible, the service responds with the contents of this file in the PDU
`data` encrypted with the `userPubEncrKey`; otherwise, it responds with a
non-zero `error` code and description in the attached `data`.

    readHeader = version type pubEncrKey error filenameLen filename

    version = u8{ 1 }      // see above
    type = u8{ READ_PDU }

    error = u16

    filenameLen = u8{ filename.Length() }
    filename = []u8

* `read` PDUs sent by the device should have empty `data`
* `read` PDUs sent by the service have `data` encrypted as follows:
   eData = crypto_box(data, dataNonce, userPubEncrKey, serviceSecEncrKey)
* This PDU is ignored by the service unless the device has previously logged-in.

### lock ###
The device sends this PDU to lock and read given user file pending and
intended `write`.  Note that there isn't an unlock, the device is expected to
follow a lock with `write`.

    lock = version type pubEncrKey error filenameLen filename

    version = u8{ 1 }      // see above
    type = u8{ LOCK_PDU }

    error = u16

    filenameLen = u8{ filename.Length() }
    filename = []u8

* `lock` PDUs sent by the device should have empty `data`
* `lock` PDUs sent by the service have `data` encrypted as follows:
   eData = crypto_box(data, dataNonce, userPubEncrKey, serviceSecEncrKey)
* This PDU is ignored by the service unless the device has previously logged-in.    

### write ###
The device sends `write` to request that the attached encrypted `data` be
written to given user file. If the file is writable, the service responds with
a nil error PDU `data`; otherwise, it responds with a non-zero `error` code
and description in the attached `data`.

    write = version type pubEncrKey error filenameLen filename

    version = u8{ 1 }      // see above
    type = u8{ WRITE_PDU }

    error = u16

    filenameLen = u8{ filename.Length() }
    filename = []u8

* `filename` must be a valid Linux file name and **not** a path.
* `write` PDUs sent by the device have `data` encrypted as follows:
   eData = crypto_box(data, dataNonce, servicePubEncrKey, userSecEncrKey)
* `write` PDUS sent by the service will have empty `data` unless there is a
  non-zero `error`.
* If the device had previously `lock`'d the file, the it's released by the
  `write`.
* This PDU is ignored by the service unless the device has previously
  logged-in.

### remove ###
The device sends this PDU to `remove` the given user file. The server responds
with this PDU having a zero `error` code if successful; otherwiese, `error` in
non-zero and `data` has the error description.

    remove = version type pubEncrKey error filenameLen filename

    version = u8{ 1 }      // see above
    type = u8{ REMOVE_PDU }

    error = u16

    filenameLen = u8{ filename.Length() }
    filename = []u8

* `write` PDUS sent by the service will have empty `data` unless there is a
  non-zero `error`
* This PDU is ignored by the service unless the device has previously
  logged-in.

## Reserved Files ##

The service reserves these file names to control file access and event
replication: `author`, `editors`, `moderators`, and `subscribers`.

The service writes the 32 byte public encryption key of the user that created
the event or forum into the file named `author`.

The `author` may assign one or more `editors` by adding their 32 by public
encryption key to the so named file. The `author` and `editors` are the only
users permitted to write files in the subject user directory.

The `author` and `editors` may also assign one or more (including themselves)
forum message `moderators`. With a non-empty `moderators` file, the service
will route all messages sent by other users through the listed `moderators`.
The service replicates all messages sent by a `moderator` to the listed
`subscribers`. So for a moderator, the App should effectively echo such forum
messages upon operator approval.

***
&copy; 2014 Apptimist, Inc. All rights reserved.
