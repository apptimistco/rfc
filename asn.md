# App-Server Protocol #
This RFC describes the protocol between an Apptimist Social Networking App and
its front-end servers.

<!--
         1         2         3         4         5         6         7         8
12345678901234567890123456789012345678901234567890123456789012345678901234567890
-->

**Status:** proposed

## Description ##
The Apptimist Social Network (ASN) protocol is an exchange of encapsulated data
units between an App and Server along with a few, server resident access control
files. The Apps themselves extend this protocol with further definition of the
encapsulated `message` and other server resident data files.

## Notation ##
In this document, *"device"* refers to the App running on IOS or Android device
whereas *"service"* refers to programs running on an Apptimist (potentially
cloud based) server.

This document describes a protocol data unit (PDU) as concatenated components.

    xyz = X Y Z

The double-quote encapsulated components denote UTF-8 strings. Unlike the
C-language, such strings are not null terminated.

Comments are introduced with a C++ style double slash.

Numeric literals may be C like decimal or "0x" prefaced hexadecimal digits.

Types are described as:

    [ "[" [ number ] "]" ] uint{8,16,32,64} [ "{" value "}" ]`
or
    [ "[" [ number ] "]" ] float{32,64} [ "{" value "}" ]`

The elemental component type is an 8, 16, 32, or 64-bit, big-endian, unsigned
integer with keywords `uint8`, `uint16`, `uint32` and `uint64` respectively.  A
`float` is either a 32 or 64 bit, big-endian, IEEE-754 floating point number.

An array component has a bracket-bound numbered preface indicating a sequence
of a single element type. Successive bracket-bound number prefaces indicate
multi-dimensional arrays. The number may be absent indicating a possible zero
length array defined by the number in another component.  Examples:

    to = [32]uint8
    recipients = uint8
    recipientList = [][32]uint8

A composite literal component appends a brace-bond list of composite elements.
Examples:

    version = uint8{1}
    nonce = [24]uint8{ id seq }
    id = [64]uint8
    seq = uint64
    nameLen = uint8{ name.Length() }
    name = []uint8
    lat = float64{37.82022}
    lon = float64{-122.47732}

## Users ##
Apptimist Social Networks have three types of users: actual, forum, and bridge.
As the name implies, an actual user is a member or administrator of the social
network. A forum or event is created by and actual or administrative user to
distribute messages to all subscribers. A bridge may also be created by a
member or administrator to multi-cast messages to all signed-in subscribers.

## WebSockets ##
ASN runs through non-secure WebSockets described in [RFC-6455][1] . These are
Web vs. TCP or UDP Sockets to provide the most flexible service through carrier
proxy servers and enterprise firewalls. Also, these are non-secure HTTP sockets
instead of the SSL secured HTTPS because ASN includes [ED25519][2] authenticated
signatures and [NaCL][3]  authenticated encryption to provide a user assigned
web-of-trust rather than a centralized certificate authority.

### URI ###

    URI = "ws://" host [ ":" port ] path [ "?" query ]

The initial `host` for all Apptimist services has this registered domain name
format.

    host = app ".apptimist.co"

Where `app` is the service name; for example, "`siren`".

Some service features may direct the client to establish simultaneous
WebSocket to another service engine; for example,
* "`marks.siren.apptimist.co`"
* "`bridge.siren.apptimist.co`"
* "`fe1.siren.apptimist.co`"
* "`fe2.siren.apptimist.co`"

The `port` component is optional; the default is 80, but may be different for
testing.

The `path` component for most services is simply a slash prefaced,
WebSocket protocol name; e.g. `/ws/asn`.

The `query` component is a 64 digit, hexadecimal string representing the
device ephemeral, 32-octet, public encryption key; e.g.
"`key=1234567890123456789012345678901234567890123456789012345678901234`"

#### Example ####

    ws://siren.apptimist.co/ws/asn?key=1234567890123456789012345678901234567890123456789012345678901234

## Cryptography ##
The ASN PDU has separately encrypted header and data components.

The App uses an ephemeral key pair for the header component cryptography. These
keys are generated before each connection with with the public key encoded
within the GET request query string as described above. The App loads user's key
pair stored within the IOS cloud for the data component cryptography. The App
registers the user's public key with the `UserAddReq` described below.

The service uses a proprietary key pair for the header component cryptography.
It uses the keys of a service specific, administration user for the data
component cryptography.

## Format ##
Each ASN PDU has this format:

    pdu = headerLen dataLen eHeader eData

    headerLen = uint32{ eHeader.Length() }
    dataLen = uint32{ eData.Length() }

    eHeader = crypto_box(header, headerNonce, rcvrPubEncrKey, sndrSecEncrKey)

    header = []uint8{ version id ... }
    vsersion = uint8{ 0 }
    id = uint8

    headerNonce = [24]uint8{ headerNoncePrefix seq }
    headerNoncePrefix = [22]uint8{ dataNonce[:22] }
    seq = uint16

    rcvrPubEncrKey = [32]uint8
    sndrSecEncrKey = [32]uint8

    eData = crypto_box(data, dataNonce, toPubEncrKey, fromSecEncrKey)
    dataNonce = [24]uint8{ ed25519(serviceSecAuthKey, servicePubEncrKey)[:24] }

    toPubEncrKey = [32]uint8
    fromSecEncrKey = [32]uint8

Every header begins with `version` and `id` components. As indicated, the
most recent `version` is 0. Future versions will be numeric increments and,
when possible, may be backwards compatible.

The `seq` component of the `headerNonce` is a sequential counter following the
NaCL ["Security model"][4] whereby this increments by 2 for each PDU sent
between socket endpoints for the connection duration.  The side with the
lexicographically smaller public key sends its first PDU with a 1 `seq`, 3 on
the second PDU, 5 on the third, etc.; meanwhile, the lexicographically larger
public key peer uses `seq` 2, 4, 6, etc.

The `headerNoncePrefix` is the first 22 bytes of `dataNonce`.

The `dataNonce` is the first 24 bytes of the service self signed public
encryption key.

## States ##
The ASN protocol has very few states.

1. `Open`, prior to an acknowledged `LoginReq`
2. `Logged-in`, after an acknowledged `LoginReq`
3. `Suspended`, after an acknowledged `PauseReq` and before `ResumeReq`
4. `Quitting`, awaiting `QuitReq` acknowledgment
5. `Close`, received `QuitReq` acknowledgment and terminated connection


## Identifiers ##
To support flexible protocol evolution, the App and Service use lookup
tables to map PDU identifier and error codes. Here are the most recent
identifiers:
<!-- go test -v -run Ids github.com/apptimistco/asn/pdu -->
                              Version
                                 0
       0.                RawId   0
       1.                AckId   1
       2.               EchoId   2
       3.        FileLockReqId   3
       4.        FileReadReqId   4
       5.      FileRemoveReqId   5
       6.       FileWriteReqId   6
       7.            HeadRptId   7
       8.            MarkReqId   8
       9.            MarkRptId   9
      10.         MessageReqId  11
      11.    SessionLoginReqId  12
      12.    SessionPauseReqId  13
      13.     SessionQuitReqId  14
      14. SessionRedirectReqId  15
      15.   SessionResumeReqId  16
      16.           TraceReqId  17
      17.         UserAddReqId  18
      18.         UserDelReqId  19
      19.      UserSearchReqId  20
      20.       UserVouchReqId  21

## Acknowledgment ##

Each `Req` suffixed request sent by either device or service shall be
acknowledged with this `Ack`.

    Ack = Version Id Req Err Desc
    Version = uint8{ 0 }
    Id = uint8{ AckId }
    Req = uint8
    Err = uint8

Where `Req` is the identifier of the request and `Err` is one of these error
codes.
<!-- go test -v -run Errs github.com/apptimistco/asn/pdu -->
                              Version
                                 0
       0.              Success   0
       1.            DeniedErr   1
       2.           FailureErr   2
       3.          IlFormatErr   3
       4.      IncompatibleErr   4
       5.          RedirectErr   5
       6.             ShortErr   6
       7.        UnexpectedErr   7
       8.           UnknownErr   8
       9.       UnsupportedErr   9

A negative acknowledgment shall include a UTF-8 character string in the data
portion of the PDU which describes the error. For `RedirectErr`, this the
redirected URL for the requested service.

The positive acknowledgment of `FileLockReq` and `FileReadReq` shall include
data containing the requested data.

The `UserSearchReq` acknowledgment includes data containing a list of the user
keys whose name or `FBuid` matches the given regular expression.

## Echo ##

The device or service may send an `Echo` request in ether `Open` or `Login`
states to diagnose and benchmark connectivity. The `Echo` receiver shall
return an `Echo` with a non-zero `Reply` along with the original data.

    Echo = Version Id Reply
    Version = uint8{ 0 }
    Id = uint8{ EchoId }
    Reply = uint8

## Files ##

The device manipulates files with these requests.

    LockReq = Version Id Name
    Version = uint8{ 0 }
    Id = uint8{ FileLockReqId }
    Name = []uint8

    ReadReq = Version Id Name
    Version = uint8{ 0 }
    Id = uint8{ FileReadReqId }
    Name = []uint8

    WriteReq = Version Id Name
    Version = uint8{ 0 }
    Id = uint8{ FileWriteReqId }
    Name = []uint8

The successful `LockReq` and `ReadReq` acknowledgment includes the requested
file contents as `data`. This `data` is encrypted with the secret key of the
service and the public key of the requesting user.  Similarly, the data`
attached to the `WriteReq`is encrypted with the secret user and public service
keys.  All or portions of the encrypted data may itself be encrypted as
explained in the *Messaging* below.

The `LockReq` also prevents another user or session from writing until a
`WriteReq` to the named file from the locking session.

### Reserved Files ###

The service stores the following reserved files in directories named
`KEY/asn`; where `KEY` is the 64 symbol, hexadecimal encoded string
representing the subject user's public key that uniquely identifies them
within the service.

    KEY/asn:
        author
        editors
        mark.yaml
        messages/head
        moderators
        subscribers
        subscriptions
        user.yaml

For efficiency, the service may make `KEY` hierarchical like GIT objects but
this is beyond to scope of this document.

Before acknowledging `UserAddReq`, the service records the relevant
components, along with originating author, in the reserved file named
`user.yaml`.  Only administrave users may write this file thereafter.
The file contains mapped strings like these.

    Name: "jane doe"
    FBuid: "jane.doe@facebook.com"
    FBtoken: "..."
    Auth: "..."

If the `UserAddReq` was issued within an established session (see below) the
service records the 32 byte public of that session user in the reservd file
named `author`.

The author may assign one or more editors by adding their user keys to the
reserved `editors` file.

Only the author and editors are permitted to write files in the subject user
directory.

The author and editors may also assign one or more (including themselves) as
forum and bridge moderators by ading their keys to the reserved `moderators`
file.

The author and editors subscribe users to a forum or bridge by adding the
subscriber's key to the `subscribers` file. The service reflects changes to
this file through the subscriber's `subscriptions` file. So, an author or
editor adding `U` user to `F` forums `F/subscribers` will result in the
service adding `F` to `U/subscriptions`.  The service prevents any other
modification of `subscriptions`.

The author and editors may statically mark a forum or bridge location by
writing a `mark.yaml` file with this mapped float format.

    Lat: 37.619002
    Lon: -122.374843
    Ele: 100

Where `Lat[itude]` and `Lon[gitude]` are in degrees and `Ele[vation]` is in
meters.

The service records the keys of checked-in users of a forum or bridge in the
`attendance` file.

The service stores all non-bridge messages in files named by the SHA512
identifier within the reserved sub-directory named `messages/`. The service
writes the 64 byte identifier of the most recent message to the file
`messages/head`. The device may truncate messages with an empty data
`WriteReq` although the service will maintain it's identifier of the previous
message.

The service lists the keys of all validate vouching users in the `references`
file. This may not be modfied in anyother way other than with the `VouchReq`.

The service prevents addition of any other files to the reserved `KEY/asn`
directory.


## Location ##

The device registers it's location or scan for nearby users with this
`MarkReq`.

    MarkReq = Version Id Lat Lon Z Cmd
    Version = uint8{ 0 }
    Id = uint8{ MarkReqId }
    Lat = float64	// Lat[itude] in degrees
    Lon = float64	// Lon[gitude] in degrees
    Z = float64		// Elevation or Radius in meters
    Cmd = uint8

Possible `Cmd` values:

    0	// Set user's anonymous location
    1   // Unset user's anonymous location
    2	// Check-in user's public location
    3	// Check-out user's public location
    4	// Scan nearby users
    5	// Stop report of nearby users

After acknowledging a scan `MarkReq`, the service will continually send this
`MarkRpt` of users within or leaving the location of interest.

    MarkRpt = Version Id Lat Lon
    Version = uint8{ 0 }
    Id = uint8{ MarkRptId }
    Key = [32]uint8
    Lat = float64
    Lon = float64
    Ele = float64

The service obfuscates the keys of users with anonymous location by nulling
the last 6 bytes. If a user has left the location of interest, the latitude
will be > 90 degrees.

An event/forum author or editor may mark the location of the event by writing
the file `KEY/asn/mark.yaml` with this mapped float format.

    Lat: 37.619002
    Lon: -122.374843
    Ele: 100

* Neither `MarkReq` nor `MarkRpt` have data.

## Messaging ##

After login, the service will report the user's most recent message and that
of all their subscribed forums with this `HeadRpt`.

    HeadRpt = Version Id Key Head
    Version = uint8{ 0 }
    Id = uint8{ HeadRptId }
    Key = [32]uint8
    Head = [64]uint8

The service will continue sending these `HeadRpt`s for every change of the
respective heads.

The `Head` references a reserved file of the form:

    KEY/messages/HEAD

Where for efficiency, the service may make `KEY` and `HEAD` hierarchical but
transparently to the device.

The device then reads the `Head` and antecedent messages with `FileReadReq`.

The device sends messages with this `MessageReq`.

    MessageReq = Version Id Time To From
    Version = uint8{ 0 }
    Id = uint8{ MessageId }
    Time = uint64
    To = [32]uint8
    From = [32]uint8

The origination `Time` is elapsed seconds since January 1, 1970 UTC.

`To` is the public encryption key recipient and `From` is the key of the
originating user.

The message `data` is encrypted with the `From` secret and `To` public keys
with an APP specific nonce.  Its format is also APP specific and may likely
contain a header identifying content type, etc.

The device may only send messages to forum or bridge users in a session
logged-in as that user. Again, `From` must be the actual subscriber.

For moderated forums, the service shunts `MessageReq` to the configured
moderators (see Reserved Files). Upon approval, the moderator's APP should
resend these `MessageReq`.

The service stores user and forum destined messages in files within  the
reserved `TO/asn/messages/` directory. These message files have this format.

    Message = Prev EMessageReq Edata
    Prev = [64]uint8
    Len = uint8
    EMessageReq = []uint8
    EData = []uint8

The message files are uniquely named with a 64 byte identifier encoded as 128
hexadecimal characters. This identifier is calculated by a SHA512 sum of the
file contents.

`Prev` is the identifier of the previous message with all zeros representing
the end.

`EMessageReq` is the originating `MessageReq` re-encrypted with the service
secret key and the `To` public key.

`EData` is the originally encrypted data included with the `MessageReq` that
used the `From` secret key and `To` public key.

So, once stored by the service, only `To` may decrypt these files.

The service doesn't store messages sent to bridge users; instead, it forwards
`MessageReq` to all active subscriber sessions with that header encrypted by
the service secret and ephemeral public session keys.

Like forums, the bridge may be moderated with the service shunting messages
through the configured moderators.

## Session Management ##

The device establishes a user session with this `LoginReq`.

    LoginReq = Version Id Key Sig
    Version = uint8{ 0 }
    Id = uint8{ SessionLoginReqId }
    Key = [32]uint8
    Sig = [64]uint8{ ed25519(SecAuthKey, Key) }

Where `Key` is the user's public encryption key and `Sig` is the user's
signature of their own key that the service authenticates with the previously
configured, user authentication key. The service prohibits login with forum or
bridge keys.

After achieving `Login` state with a positive acknowledgment, the device may
suspend the session to maintain the connection in a low power state with a
`PauseReq`; then continue with `ResumeReq`.

    PauseReq = Version Id
    Version = uint8{ 0 }
    Id = uint8{ SessionPauseReqId }

    ResumeReq = Version Id
    Version = uint8{ 0 }
    Id = uint8{ SessionResumeReqId }

The service may instruct the device to terminate the active connection and
reconnect at another URL with this `RedirectReq`.

    RedirectReq = Version Id Url
    Version = uint8{ 0 }
    Id = uint8{ SessionRedirectReqId }
    Url = []uint8

Either device or service may terminate the session at anytime with this
`QuitReq`.

    QuitReq = Version Id Url
    Version = uint8{ 0 }
    Id = uint8{ SessionQuitReqId }

* None of these session management PDUs have `data`.

## User Management ##

Before `Login`, the device may request an account for a new user with this
`UserAddReq`.

    AddReq = Version Id Key Auth NameLen UidLen TokenLen Name Uid Token
    Version = uint8{ 1 }
    Id = uint8{ UserAddReqId }
    User = uint8		// { 0: actual, 1: forum, 2: bridge }
    NameLen = uint8{ Name.Length() }
    UidLen = uint8{ Uid.Length() }
    TokenLen = uint8{ Token.Length() }
    Key = [32]uint8		// Public Encryption Key
    Auth = [32]uint8		// Public Authorization Key
    Name = []uint8		// unique user name
    FBuid = []uint8		// Facebook user identifier
    FBtoken= []uint8		// Facebook authentication token

Upon receiving a positive acknowledgment, the device may proceed to session
establishment with a `SessionLoginReq`.

After `Login` of an actual user, the device may send any of the following user
administration requests. The device may also send the above `UsersAddReq` to
create event/forum and bridge type users. Id addition, administrative users
may add users by proxy.

The device may delete any actual, forum, or bridge user in which they are the
registered author (including themselves) with this `UserDelReq`.

    DelUserReq = Version Id PubEncr
    Version = uint8{ 0 }
    Id = uint8{ UserDelReqId }
    Key = [32]uint8

The device may search for users by public encryption key, service user name,
and Facebook user identifier with this `UserLookupReq`.

    UserSearchReq = Version Id By Thing
    Version = uint8{ 0 }
    Id = uint8{ UserSearchReqId }
    Max = uint16
    By = uint8			// { 0: name, 2: FB uid }
    Regex = []uint8

`Regex` is a [POSIX basic regular expression][5] matching the respective
service user name or Facebook user identifier.

The service acknowledges the search with data containing a key list of
matching users.

The device may vouch for another user by sending an authenticated signature of
the subject user's public key with this `UserVouchReq`. To revoke an earlier
voucher, the device resends this with a null signature.

    UserVouchReq = Version Id PubEncr Signature
    Version = uint8{ 0 }
    Id = uint8{ UserVouchReq }
    Key = [32]uint8
    Signature = [64]uint8{ ed25519(secAuth, Key) }

* None of these user management PDUs have `data`.

# References #
1. http://tools.ietf.org/html/rfc6455
2. http://ed25519.cr.yp.to/
3. http://nacl.cr.yp.to/box.html
4. [NaCL "Security model"](http://nacl.cr.yp.to/box.html)
5. http://en.wikibooks.org/wiki/Regular_Expressions/POSIX_Basic_Regular_Expressions

***
&copy; 2014 Apptimist, Inc. All rights reserved.
