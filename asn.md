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

    owner = [32]uint8
    namelen = uint8
    name = []uint8

A composite literal component appends a brace-bond list of composite elements.
Examples:

    version = uint8{0}
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
Web vs. TCP or UDP Sockets to provide the most flexible service through
carrier proxy servers and enterprise firewalls. Also, these are non-secure
HTTP sockets instead of the SSL secured HTTPS because ASN includes
[ED25519][2] authenticated signatures and [NaCL][3]  authenticated encryption
to provide a user assigned web-of-trust rather than a centralized certificate
authority.

### URI ###
    URI = "ws://" host [ ":" port ] path [ "?" query ]

The initial `host` for Apptimist services has this registered domain name.

    host = app ".apptimist.co"

Where `app` is the service name; for example, "`siren`".

Some service features may direct the client to establish simultaneous
WebSocket to another service engine; for example,

    sf.siren.apptimist.co
    la.siren.apptimist.co
    ny.siren.apptimist.co
    bridge.siren.apptimist.co

The `port` component is optional; the default is 80, but may be different for
testing.

The `path` component for most services is a slash prefaced, WebSocket protocol
name; e.g. `/ws/asn`.

The `query` component is unused and should be empty.

For example,

    ws://siren.apptimist.co/ws/asn

## Cryptography ##
ASN PDUs are encrypted in segments of up to 4096-bytes, including the 16-byte
encryption overhead. Each segment is preceded by a uint16 length with the most
significant bit used as a `More` flag that is set for each but the final PDU
segment.

    segment = moreLen data
    moreLen = uint16
    data = [1..4096]uint8

After connecting, the device (the App or a mirroring server) sends its
32-byte, ephemeral public key to the server. It uses this key along with the
proprietary service key and nonce to encrypt the first `login` request PDU.
The server's acknowledgment includes it's own ephemeral key and nonce to
replace the initial proprietary key/nonce in subsequent segment encryption.

The last two bytes of the initial proprietary, then following ephemeral nonce
are sequential counters following the NaCL ["Security model"][4] whereby these
increment by 2 for each segment sent between socket endpoints for the
connection duration.  The side with the lexicographically smaller public key
sends its first segment with 1, 3 on the second, 5 on the third, etc.;
meanwhile, the lexicographically larger public key peer uses 2, 4, 6, etc.

## Format ##
Each ASN PDU begins with version and identifier components.

    pdu = version id
    version = uint8{ 0 }
    id = uint8

There are two types of PDUs, requests and objects.

Along with the acknowledgment, each request has an 8-byte requester following
the version, id pair.

    requester = [8]uint8

Each object has this 8-byte magic string following the version, id pair.

    magic = [8]uint8{"asnmagic"}

We describe the remaining object and request formats below.

## States ##
The ASN protocol has very few states.

1. `opened`, prior to an acknowledged `login` or subsequent `resume`
2. `provisional`, after first `login`, prior to upload of authentication key
3. `established`, after verified `login`
4. `suspended`, after an acknowledged `pause` and before `resume`
5. `quitting`, awaiting `quit` acknowledgment
6. `closed`, received `quit` acknowledgment and terminated connection


## Identifiers ##
To support flexible protocol evolution, the App and Service use lookup
tables to map PDU identifier and error codes. Here are the most recent
identifiers:
<!-- go test -v -run Ids github.com/apptimistco/asn -->

                         Version
                            0
       1.        AckReqId   1
       2.       ExecReqId   2
       3.      LoginReqId   3
       4.      PauseReqId   4
       5.       QuitReqId   5
       6.   RedirectReqId   6
       7.     ResumeReqId   7
       8.          BlobId   8
       9.         IndexId   9

## Acknowledgment ##
Each request sent by either device or service shall be acknowledged with
this `AckReq`.

    ack = version id requester result data
    version = uint8{ 0 }
    id = uint8{ AckReqId }
    requester = [8]uint8
    result = uint8
    data = []uint8

Where `result` is one of these codes.
<!-- go test -v -run Errs github.com/apptimistco/asn -->

                         Version
                            0
       0.         Success   0
       1.       DeniedErr   1
       2.      FailureErr   2
       3.     IlFormatErr   3
       4. IncompatibleErr   4
       5.     RedirectErr   5
       6.        ShortErr   6
       7.   UnexpectedErr   7
       8.      UnknownErr   8
       9.  UnsupportedErr   9

A negative acknowledgment shall include a UTF-8 character string describing
the error as the `data` component except for `RedirectErr` where it's the
redirected URL for the requested service.

The `data` component of positive acknowledgments, if any, is described below.

## Session Management ##
The device establishes a user session with this `login` request immediately
after sending its ephemeral key.

    login = version id requester user sig
    version = uint8{ 0 }
    id = uint8{ AckReqId }
    requester = [8]uint8
    user = [8]uint8
    sig = uint8

Where `user` is the user's public (not ephemeral) encryption key and `sig` is
the user's signature of their own key. If this is the user's first login, the
session is in `provisional` state until the device uploads the user's
authentication key (see below). Once available, the service verifies the
signed user key before setting the `established` state.  The service prohibits
login with forum or bridge keys.

After session establishment the device may suspend to maintain the connection
in a low power state with this `pause` request; then continue with the
`resume` request.

    pause = version id requester
    version = uint8{ 0 }
    id = uint8{ PauseReqId }
    requester = [8]uint8

    resume = version id requester
    version = uint8{ 0 }
    id = uint8{ ResumeReqId }
    requester = [8]uint8

The service may instruct the device to terminate the active connection and
reconnect (at possibly another URL) with this `redirect` request. With an
empty `url` component, the device should reconnect to the same server after
waiting 10 or more seconds.

    redirect = version id requester url
    Version = uint8{ 0 }
    id = uint8{ RedirectReqId }
    requester = [8]uint8
    url = []uint8

Either device or service may terminate the session at anytime with this `quit`
request.

    quit = version id requester
    version = uint8{ 0 }
    id = uint8{ QuitReqId }
    requester = [8]uint8

## Exec ##
This `exec` request is a command line style interface to several services.

    exec = version id requester argv
    version = uint8{ 0 }
    id = uint8{ ReqAckId }
    requester = [8]uint8
    argv = []string

Where `argv` is a null separated list of ASCII command arguments with the
optional `--` separating command input. All `exec` requests receive an
acknowledgment, in addition, some may also have associated objects sent to the
requester.

### add ###
    add [USER/][INDEX][/NAME] -- DATA

The device may make an `add` request in the `provisional` state to upload and
reference the user's authentication key within the ASN index with this command
where AUTH is the login user's 32-byte, public, ED25519 authentication key or
its ASCII, 64-digit, hexadecimal character representation.

    add asn/auth -- AUTH


The device may also make this request in the `established` state to upload any
named blob within it associate index, or a message in it's implicit index like
this.

    add USER -- MESSAGE

In this case, a message object is created from MESSAGE and it's SHA sum
identifier is added to both USER and login's message index.

### approve ###
    approve SHA...

A moderator's device may make an `approve` request in the `established` state
for the server to add the blobs with the given SHA sums to the respective
indices of the moderated bridge or forum.

### cat ###
    cat [USER/]INDEX/PATTERN

The device may make a `cat` request in the `established` state for the server
to acknowledge with the decoded blob with the matching name within the given
index of the login or specified USER.

### echo ###

    echo [STRING]...

The device may make an `echo` request in `open`, `provisional` or
`established` states.  The data of the server's positive acknowledgment has
the joined, space separated list of arguments without trailing newline.

### fetch ###

    fetch [USER[/]][INDEX]</PATTERN|@EPOCH>...
    fetch SHA...

The device may make a `fetch` request in the `established` state to have the
server send matching blobs after the request acknowledgment.

This requests messages since EPOCH.

    fetch @EPOCH

This requests messages to the ASCII public key, USER since EPOCH.

    fetch USER/msg@EPOCH

This requests the blob containing USER's ASN authentication key.

    fetch USER/asn/auth

This requests blobs with the given SHA sums.

    fetch SHA...

### ls ###
    ls [USER/][INDEX][/PATTERN|@EPOCH]...

The device may make an `ls` request in the `established` state for the server
to acknowledge with a newline separated list of matching names within the
given index of the login or specified USER. Each line contains space
separated, SHA, NAME pairs.

    ls asn/auth*

    0190fb33 auth
    15139977 author

The device may also request the list of messages since a given EPOCH.

    ls @2147483647

    0190fb33
    15139977

### new-event, new-bridge ###
The device may make an `new-event` or `new-bridge` request in the
`established` state for the server to create a new event or bridge user with
random keys. The acknowledgment has a string like this:

    pub:
      encr: 5fb2d5d9552c47f02d4cfc1f3938abd4c5f685b050501e53f6bf545c05982e33
      auth: 9d30799789fb96a2d71855168d8573d2ce6f367e6a0ef7da7bcee72ab31dcc13
    sec:
      encr: f6ce8a1025b3537e3a82ab5461fa7a2db51a2729abe66cdce82b54a573de011d
      auth: 60eabf950dc926735d086f419b2571de6e95c4e1d1efe179590b1acc8ffee39c9d30799789fb96a2d71855168d8573d2ce6f367e6a0ef7da7bcee72ab31dcc13

Before sending the acknowledgment the server creates these blobs and
associated `asn` index entries.

    asn/author
    asn/auth
    asn/type

It's up to the App to securely store and distribute the secrete encryption and
authentication keys.

### rm ###
    rm SHA...

The device may make an `rm` request in the `established` state for the server
to remove the identified objects and references within associated indices.  It
may only remove objects that is owned or authored by the login user. The
server doesn't immediately remove these objects, instead it merely propagates
flags for later, independent removal by garbage collectors on each mirror.

### scan ###
    scan LATITUDE LONGITUDE RANGE

The device may make a `scan` request in the `established` state to have the
server forward MARK objects within RANGE meters of LATITUDE and LONGITUDE
degrees.  A subsequent `scan` with zero RANGE discontinues forwarding of all
MARK objects. These also stop with `pause` request so the device must issue
another `scan` command to continue mark reports after `resume`.

Like all `exec` commands, these arguments must be ASCII; for example:

    scan 37.619002 -122.374843 100

### trace ###
    trace [COMMAND [ARG]]

The device may make a `trace` request in the `established` state to retrieve
or adjust the server's PDU trace. The default, "flush" command results in the
text PDU trace returned in the data component of the positive acknowledgment.
The other trace commands return success or error.

    trace [flush]
    trace filter ID...
    trace unfilter ID...
    trace resize RINGSIZE

The default RINGSIZE is 32.

### vouch ###
    vouch USER SIG

The device may make a `vouch` request in the `established` state for the login
user to vouch for the identity on another by signing their key. `USER` may be
abbreviated but must be an ASCII representation of the subject's public key.
`SIG` must be the full, 128-character ASCII representation of login's ED25519
authenticated signing of the subject's key.

## Objects ##
Whereas requests result in some sort of acknowledged action, an object conveys
unacknowledged information. After the magic string, every object has this
header.

    object = random owner author epoch
    random = [32]uint8	// random data
    owner = [32]uint8	// public encryption key
    author = [32]uint8	// "
    epoch = uint64	// Unix epoch (nanoseconds)

Random data is used to differentiate objects with identical content.

### Blobs ###
The data of `blob` objects, with the exception of ASN control blobs, is opaque
to the server.  It may be fetched and read by anyone so the App is generally
responsible for security, compression, content identification and
interpretation. Once written, blobs may not be modified other than being
flagged for later removal during garbage collection. Following the common
object header, blobs have the associated name preceding the actual blob data.

    blob = version id magic random owner author epoch index name data
    version = uint8{ 0 }
    id = uint8{ BlobObjId }
    magic = [8]uint8{"asnmagic"}
    random = [32]uint8	// random data
    owner = [32]uint8	// public encryption key
    author = [32]uint8	// "
    epoch = uint64	// Unix epoch (nanoseconds)
    namelen = uint8
    name = []uint8
    data = []uint8

The service records the blob SHA sum in the index taken from the blob `name`
component up to the first forward slash '/' character. For example, the blob
named `asn/auth` has it SHA sum added to the `asn` index.

### Index ###
An `index` object lists the SHA sum references of the owner's blobs with
matching named preface.

    index = version id magic random owner author epoch index name data
    version = uint8{ 0 }
    id = uint8{ IndexObjId }
    magic = [8]uint8{"asnmagic"}
    random = [32]uint8	// random data
    owner = [32]uint8	// public encryption key
    author = [32]uint8	// "
    epoch = uint64	// Unix epoch (nanoseconds)
    namelen = uint8
    name = []uint8
    shas = [][32]unint8

Each user has an implicit index for messages in addition to one named `asn`.
Any other indices are App specific and opaque to the service.

These aren't ordinarily used directly by App devices; instead, they are used
by the server to complete these `exec` commands: `add`, `cat`, `ls`, and `rm`.

### Mark ###
A `mark` is simply a `blob` object with data identifying the user location.

    mark = version id magic random owner author epoch lat lon ele
    version = uint8{ 0 }
    id = uint8{ BlobObjId }
    magic = [8]uint8{"asnmagic"}
    random = [32]uint8	// random data
    owner = [32]uint8	// public encryption key
    author = [32]uint8	// "
    epoch = uint64	// Unix epoch (nanoseconds)
    namelen = uint8{8}
    name = []uint8{"asn/mark"}
    lat = float64	// latitude (degrees)
    lon = float64	// longitude (")
    ele = float64	// elevation (meters)

To check-in at an event, the device sends a `mark` with the event's key as
`owner` and it's login user key as `author`. Outside of the event, `owner`
must equal `author`. To remain anonymous (gray pin), the device uses the
session ephemeral keys instead of the user key. To clear an earlier registered
location, the device sends a mark `lat` > 91 degrees.

The server forwards all received or recorded marks within the user's `scan`
range.

### Message ###
A `message` is simply a `blob` with an empty name and App specific data.

    message = version id magic random owner author epoch data
    version = uint8{ 0 }
    id = uint8{ MessageObjId }
    magic = [8]uint8{"asnmagic"}
    random = [32]uint8	// random data
    owner = [32]uint8	// public encryption key
    author = [32]uint8	// "
    epoch = uint64	// BigEndian Unix epoch nanoseconds
    namelen = uint8{0}
    data = []uint8

## ASN Control ##
The service loads and stores these named blobs reference by the `asn` index.

    asn/auth
    asn/author
    asn/editors
    asn/mark
    asn/moderators
    asn/references
    asn/type

At the `provisional` login, `new-event` or `new-bridge` request the service
creates a blob named `asn/author` that contains the requester, binary public
key.  It also records the respective string `actual`, `event`, or `bridge` as
data in a blob named `asn/type`.

The author may assign one or more editors by adding their user keys to a blob
named `asn/editors`.

Only the author and editors are permitted to make blobs in anything other than
the empty named message index.

The author and editors may assign one or more (including themselves) as forum
and bridge moderators by adding their keys to a blob referenced by
`asn/moderators`. Any messages for the forum or bridge are added to the
`moderators'` message index. Any moderator may then issue the `exec approve`
request for the server to add the respective SHA to the forum or forward to
the bridge.

The author and editors may statically mark a forum or bridge location by
sending a `mark` object with `owner` and `author` set to subject along with
its latitude, longitude and elevation. The service will store this object
named `asn/mark` and forward to all user's filtered by their `scan` range.

The service lists the keys of all validated, vouching users in a blob named
`asn/references`. This may not be modified in any other way other than with
the `exec vouch` request.

The service doesn't recognize any other blobs named with the `asn/` index
prefix.

# References #
1. http://tools.ietf.org/html/rfc6455
2. http://ed25519.cr.yp.to/
3. http://nacl.cr.yp.to/box.html
4. [NaCL "Security model"](http://nacl.cr.yp.to/box.html)
5. http://en.wikibooks.org/wiki/Regular_Expressions/POSIX_Basic_Regular_Expressions

***
&copy; 2014 Apptimist, Inc. All rights reserved.
