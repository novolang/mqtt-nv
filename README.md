# mqtt-nv

MQTT is a client/server publish/subscribe messaging transport protocol,
lightweight enough for a device with kilobytes of memory and specified
by OASIS in
[MQTT Version 5.0](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
and
[MQTT Version 3.1.1](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html).
This package is the MQTT client: the half that owns a socket, keeps the
session and counts the keep-alive. It is built on
[mqtt-codec-nv](https://novo-lang.org/packages/mqtt-codec-nv), which
turns packets into bytes and bytes into packets and knows nothing about
a connection.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What an MQTT client is

An MQTT client connects to a server, which the specification also calls
a **broker**, and all traffic goes through it. No client addresses
another. A client publishes a message under a **topic name** and the
broker delivers it to every client whose subscription filter matches
that name.

The first packet on a connection is a CONNECT and the broker answers it
with a CONNACK (sections 3.1 and 3.2). A client may send nothing else
until the CONNACK arrives. After it, the client subscribes, publishes,
and answers what the broker sends.

The **keep-alive** is the longest interval the client may let pass
between packets it sends (section 3.1.2.10). A client with nothing to
say sends a PINGREQ and the broker answers with a PINGRESP. A broker
that has heard nothing for one and a half keep-alive intervals closes
the connection.

The **session** is the state the client and the broker keep about each
other between connections: the subscriptions, the publications sent and
not acknowledged, and the QoS 2 identifiers received and not released
(section 4.1). The session is what makes a quality of service above 0
mean anything. A QoS 1 publication is delivered at least once because
the client keeps sending it until a PUBACK arrives, and that wait
outlives the socket it started on.

This package holds the client as a value and never runs a loop of its
own. The caller reads its clock once a turn, calls `poll`, and acts on
the events it gets back. Nothing here starts a thread, sleeps, opens a
file or draws a random number.

`mqttopts` ships three sets of defaults. MQTT's own defaults are
unlimited almost everywhere, so a client that took them literally
buffers without bound the first time a broker stops acknowledging.

| Setting | `options` | `rt_profile` | `unlimited` |
| --- | --- | --- | --- |
| Protocol version | 5.0 | 3.1.1 | 5.0 |
| Keep-alive | 60 seconds | 300 seconds | 60 seconds |
| Clean start | yes | yes | yes |
| Receive Maximum | 100 | 1 | 65,535 |
| Maximum Packet Size | 256 KiB | 2 KiB | 268,435,455 bytes |
| Topic Alias Maximum | 0 | 0 | 65,535 |
| Decoder limits | `mqttread.default_limits()` | `mqttread.embedded_limits()` | the protocol's own |

What a call costs is decided by what the caller hands it. A function
that opens a socket declares the effects of the standard library
surface it uses, and the functions that drive a client declare whatever
their transport declares.

| Call | Effects |
| --- | --- |
| `mqtttrans.dial_tcp`, `.dial_tcp_timeout` | `[io, net]`, which is `std.net`'s own |
| `mqtttrans.dial_tls` | `[net]`, which is `std.tls`'s own |
| `mqttclient.now_ms` | `[time]`, and it is the only clock reading in the package |
| `mqttclient.connect`, `.poll`, `.publish`, `.publish_packet`, `.subscribe`, `.unsubscribe`, `.ping`, `.disconnect`, `.resume` | whatever the transport costs |
| Everything else: the options, the session, the backoff, the keep-alive arithmetic | none |

A program that runs in a sandbox granting `[net]` and refusing `[io]`
can therefore use the TLS transport and not the TCP one, and can see
that before it writes the call.

## Install

```
novo pkg add mqtt-nv
```

## Example

```novo
use mqttclient
use mqttopts
use mqttsession
use mqtttrans

fn main() [io, net, time]
    // Open the socket. `dial_tls` is the same call on port 8883.
    match mqtttrans.dial_tcp("broker.example.com", mqtttrans.MQTT_PORT)
        Err(e) => println("the broker did not answer")
        Ok(t)  =>
            // What the CONNECT will say: this client's name, and a
            // PINGREQ owed after thirty seconds of silence.
            let o = mqttopts.with_keep_alive(mqttopts.options("sensor-7"), 30)

            // The client is a value. The session beside it is the state
            // that outlives the socket, and storing it between runs is
            // what makes a QoS 1 publication survive a restart.
            var c = mqttclient.client(o, mqttsession.session("sensor-7"))

            // Send the CONNECT and wait for the CONNACK.
            match mqttclient.connect(c, t, mqttclient.now_ms())
                Err(e)   => println("the connection failed: ${e.message()}")
                Ok(step) =>
                    c = step.client

                    // One turn of your own loop: resend what is due, ping
                    // if the keep-alive says so, read, and answer.
                    match mqttclient.poll(c, t, mqttclient.now_ms())
                        Err(e) => println("the connection ended: ${e.message()}")
                        Ok(s)  =>
                            c = s.client
                            for ev in s.events
                                match ev
                                    MqttMessageArrived(m) => println(m.topic)
                                    _                     => println("something else")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: mqtt-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `mqtttrans` | The transport: a trait of four methods over a byte pipe, and two implementations of it, one over TCP and one over TLS with its options record. |
| `mqttopts` | Everything decided before a socket exists: the CONNECT's fields, three sets of defaults, and the reasons a set of options cannot be used. |
| `mqttclient` | The client value, the five states a connection can be in, the eight events a turn can report, and the calls that drive it. |
| `mqttsession` | The state that outlives a socket: unacknowledged publications, received QoS 2 identifiers, subscriptions, packet identifiers and both topic alias tables. |
| `mqttretry` | A capped exponential backoff as arithmetic, with the attempt count and the jitter as arguments. |
| `mqttclienterr` | Every way a client stops, and the two questions a caller has about each: can it reconnect, and is the session still worth keeping. |

## How to choose an entry point

**A transport is whatever moves bytes.** `mqtttrans.dial_tcp` gives one
over a plain socket and `mqtttrans.dial_tls` one over TLS. Implement
`MqttTransport[e]` yourself for anything else: a board's own socket, a
serial link to a cellular module, or a script of recorded bytes in a
test. The four methods are send, receive, close and a question about
whether the pipe is open, and nothing in them knows about MQTT packets.

**`connect` waits for the CONNACK. `poll` does everything else.** Call
`connect` once per connection, then `poll` on every turn of your loop.
`poll` returns as soon as the transport has nothing more to give, so a
caller with other work does it between turns.

**`publish` takes the five fields most messages have.** Use
`publish_packet` with a `MqttPublish` you built yourself when you need
an MQTT 5.0 property that `publish` has no parameter for, such as a
response topic, correlation data or a message expiry.

**Call `resume` after a `connect` whose CONNACK reported the session
present.** It sends what the session owes. It is a separate call so that
a caller who has been away can read `mqttsession.outgoing_of` and see
what is about to go out.

**`wait_ms` answers without taking a turn.** A program that selects over
several clients computes one timeout from all of them before it blocks
on any.

## The rules a user needs

1. **Store the session, or QoS 1 and QoS 2 stop at the process
   boundary.** `MqttSession` is a value with no storage attached. Take
   it with `session_of`, put it where your program keeps state, and pass
   it back to `client` on the next run. A client whose in-flight table
   lived only in memory delivers at least once only until it exits,
   which is a weaker promise than the caller asked for (section 4.1).
2. **The keep-alive counts from the last packet sent, not received**
   (section 3.1.2.10). A broker that is publishing steadily to a silent
   client still expects the PINGREQ. `keep_alive_due` answers when one
   is owed and `keep_alive_expired` answers when the PINGRESP has been
   outstanding longer than `ping_timeout_ms`, which is the only way to
   notice a socket that is open and dead. A keep-alive of 0 turns both
   off.
3. **A turn does five things in one order: resend, ping, check the
   keep-alive, read, answer.** The first three come before the read
   because a transport read can block, and a keep-alive checked only
   after a read that never returns never fires.
4. **`clean_start` false does not guarantee a session.** The broker's
   CONNACK carries the Session Present flag (section 3.2.2.1.1), and a
   broker that has nothing stored answers false. That is
   `MqttSessionNotResumed`, and the answer to it is to subscribe again
   from `MqttSession.subscriptions`, which `clear` keeps for that
   purpose.
5. **Packet identifiers come from the session** (section 2.2.1). An
   exchange survives a reconnection, so a counter that restarted with
   the connection would hand out an identifier still in flight.
   `take_packet_id` skips the outstanding ones and answers
   `MqttPacketIdsExhausted` when all 65,535 are in use.
6. **A retransmission is byte-identical except for the DUP flag**
   (sections 3.3.1.1 and 4.4). The session stores the encoded packet for
   that reason, not the value it came from. By default a packet is
   resent only on a new connection, which is what the specification
   requires. Setting `resend_after_ms` above zero also resends it on the
   connection it was first sent on.
7. **A QoS 2 exchange changes shape half-way.** After the PUBREC the
   PUBLISH must never go out again and the PUBREL must
   (section 4.3.3). `advance_to_pubrel` replaces the stored bytes, and
   `is_duplicate_incoming` is what keeps a redelivered PUBLISH from
   being delivered to the application twice.
8. **The effective limits are the smaller of yours and the broker's.**
   The CONNACK announces the broker's Receive Maximum and Maximum Packet
   Size among its properties (section 3.2.2.3). Sending past the Receive
   Maximum is a protocol error the broker answers by closing the
   connection (section 4.9), so `publish` refuses first with
   `MqttSendQuotaExceeded`.
   Read `send_quota` before you pull from a queue of your own.
9. **Topic aliases belong to one connection** (section 3.3.2.3.4). Call
   `clear_aliases` on every reconnection. A client that carried an alias
   across would publish to whatever alias 3 means on the new connection.
   An incoming alias this connection was never taught is
   `MqttTopicAliasUnknown`, because there is no topic to deliver the
   message under.
10. **A SUBACK that grants a lower QoS than you asked for is a
    success** (section 3.9.3). The reason codes come back one per
    filter, in the order the filters were sent, and a client that
    checked only for failure reports the subscription as working and
    then receives nothing at the level it wanted.
11. **A refused CONNECT is not worth retrying.** The broker read the
    packet and said no, which is a protocol success with a failure
    reason code (section 3.2.2.2). `is_reconnectable` answers false for
    it, and a client that retried anyway turns a mistyped password into
    a flood against the broker that rejected it. `keeps_session` is the
    second question: a transport failure leaves the session valid and a
    refused CONNECT does not.
12. **A clean DISCONNECT is what suppresses the will.** The broker
    publishes the last will exactly when a connection ends without one
    (sections 3.1.2.5 and 3.14). A program with a will has to call
    `disconnect` on the way out.
13. **The clock is an argument.** Every function that cares about time
    takes a monotonic millisecond reading, and `mqttclient.now_ms` is
    the only function that obtains one. A caller driving twenty brokers
    reads its clock once a turn rather than twenty times, and a program
    with its own clock never calls it at all.
14. **The backoff jitter is an argument too.** `mqttretry.delay_ms`
    takes a draw from 0 to 999 in thousandths and answers a number of
    milliseconds. Nothing here waits. Passing 0 gives the deterministic
    curve, and a fleet reconnecting after a broker restart needs a real
    draw so that it does not arrive all at once.

## What is not included

- **A loop, a thread, a channel or a callback.** Every program worth
  putting an MQTT client into already has a loop. This package is
  driven from that one.
- **Sleeping.** `mqttretry.delay_ms` and `MqttPoll.wait_ms` answer
  numbers of milliseconds. Waiting is the caller's.
- **Randomness.** The backoff jitter is a number the caller draws.
- **Session storage.** The session is handed out as a value and read
  back as one. Redis, a file, a flash page or a variable in a test are
  all the caller's decision.
- **A broker.** A subscription index, retained message storage and a
  client table are not here. `mqtttopic.matches` in
  [mqtt-codec-nv](https://novo-lang.org/packages/mqtt-codec-nv) is the
  predicate a broker would be built on.
- **A firmware build.** This package is written against the standard
  library's sockets and does not build for a microcontroller. A device
  runs mqtt-codec-nv, an `MqttTransport[e]` written against its own
  hardware socket, and the numbers `mqttopts.rt_profile` names. Two
  things are still missing before that works: a fixed-capacity session
  store, since `MqttSession` holds three growable lists, and
  mqtt-codec-nv's own device build, which waits on a toolchain defect
  its README records.
- **MQTT-SN.** A different protocol with a different packet format, for
  links with no TCP under them.

## Related packages

- [mqtt-codec-nv](https://novo-lang.org/packages/mqtt-codec-nv) is the
  other half of MQTT: every control packet as a typed value, the
  variable byte integer, the topic rules and a reader that takes bytes
  in any pieces. Take that package alone to read or write packets with
  no connection in the program, which is what a broker, a proxy, a
  capture tool and a device with its own socket each do. Take this one
  to talk to a broker. Every packet this package sends was built there,
  and every packet it reads was decoded there.
- [websocket-nv](https://novo-lang.org/packages/websocket-nv) is the
  same split for WebSocket: a client and a server over
  [websocket-codec-nv](https://novo-lang.org/packages/websocket-codec-nv).
  MQTT over WebSocket is a transport this package does not ship, and
  that pair is what one would be written over.
- `std.net` in the standard library is what `MqttTcp` is written
  against. `MqttTls` is written against a TLS connection handle, and the
  options beside it are `mqtttrans.MqttTlsConfig` — this package's own
  record, because the standard library's TLS surface is not published
  (there is no `docs/stdlib/tls.md` and no module a `use` resolves), so
  no package can name a type from it. `std.time` is where `now_ms` reads
  its clock, and it is the only place this package touches it.

## Tests

```bash
novo test tests/mqttopts_tests.nv      # 10 tests: the options and the CONNECT they mean
novo test tests/mqttsession_tests.nv   # 11 tests: what a session keeps, replaces and reuses
novo test tests/mqttclient_tests.nv    # 12 tests: the keep-alive, the backoff, the errors
novo test tests/mqtttrans_tests.nv     #  9 tests: a client driven over a script of bytes
```

There is no broker in the suite and no clock. The behaviour asserted
comes from the OASIS specifications: the CONNECT the options must
produce (section 3.1), the combinations of options the specification
forbids, the session rules of section 4.1 and the QoS flows of section
4.3. The client's shape is read against `rumqttc`, the Rust client this
package takes its options builder and its caller-driven loop from.

`tests/mqtttrans_tests.nv` implements `MqttTransport[]` over bytes
already in memory and drives a client with it. The compiler checks that
file before it runs: a driving function that was secretly effectful, or
an effect that did not flow from the transport to the call, would stop
it compiling.

The tests compile today and fail at run, each on the
`not implemented: mqtt-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `mqtttrans.MQTT_PORT`, `.MQTT_TLS_PORT` | yes (they are constants) |
| `mqtttrans.dial_tcp`, `.dial_tcp_timeout`, `.dial_tls`, `.tls_for`, `.stream_of` | no |
| `mqtttrans`: both `MqttTransport` implementations | no |
| `mqttopts.options`, `.rt_profile`, `.unlimited` | no |
| `mqttopts.with_version`, `.with_keep_alive`, `.with_session`, `.with_credentials`, `.with_will` | no |
| `mqttopts.with_limits`, `.with_decode_limits`, `.with_timeouts` | no |
| `mqttopts.faults`, `.connect_packet`, `.keep_alive_ms` | no |
| `mqttsession.session`, `.clear`, `.is_present`, `.take_packet_id` | no |
| `mqttsession.record_outgoing`, `.advance_to_pubrel`, `.complete_outgoing`, `.outgoing_of`, `.inflight_count` | no |
| `mqttsession.due_for_resend`, `.resume_packets` | no |
| `mqttsession.record_incoming`, `.release_incoming`, `.is_duplicate_incoming` | no |
| `mqttsession.record_subscription`, `.forget_subscription` | no |
| `mqttsession.alias_for`, `.topic_for_alias`, `.learn_alias`, `.clear_aliases` | no |
| `mqttsession.encode`, `.decode`, the `message` implementation | no |
| `mqttretry.backoff`, `.never`, `.policy` | no |
| `mqttretry.delay_ms`, `.base_delay_ms`, `.verdict`, `.has_attempts_left` | no |
| `mqttclient.client`, `.state_of`, `.session_of`, `.send_quota`, `.now_ms` | no |
| `mqttclient.connect`, `.poll`, `.publish`, `.publish_packet`, `.resume` | no |
| `mqttclient.subscribe`, `.unsubscribe`, `.ping`, `.disconnect` | no |
| `mqttclient.on_connection_lost`, `.wait_ms` | no |
| `mqttclient.keep_alive_due`, `.keep_alive_expired` | no |
| `mqttclient.effective_receive_maximum`, `.effective_max_packet_size` | no |
| `mqttclienterr.is_reconnectable`, `.keeps_session`, `.disconnect_reason`, the `message` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
