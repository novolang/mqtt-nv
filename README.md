# mqtt-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The half of MQTT that owns a socket.  mqtt-codec-nv turns packets into
bytes and bytes into packets; this package connects, keeps the session,
counts the keep-alive, retransmits what was not acknowledged, resolves
the topic aliases, reconnects on a policy, and hands the caller the
messages that arrived.

It does not own a loop, a thread or a channel.  A host holds an
`MqttClient`, reads its clock once per turn, calls `poll`, and acts on
the events it gets back.

## Adding it, and checking it

```bash
novo pkg add mqtt-nv        # into your novo.toml
novo pkg build              # type- and effect-check the package
novo test --isolate tests/mqttclient_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: mqtt-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use mqttclient
use mqttopts
use mqttsession
use mqtttrans

// Connect, subscribe, and run the caller's loop until something ends
// it.  One clock reading per turn, and every decision the client makes
// is made from that reading.
fn run(host: Str) -> Result<Unit, MqttClientError> [io, net, time]
    let t = mqtttrans.dial_tcp(host, mqtttrans.MQTT_PORT)!
    let o = mqttopts.with_keep_alive(mqttopts.options("sensor-7"), 30)
    var c = mqttclient.client(o, mqttsession.session("sensor-7"))

    c = mqttclient.connect(c, t, mqttclient.now_ms())!.client
    c = mqttclient.subscribe(c, t, subscriptions(), mqttclient.now_ms())!.client

    loop
        let step = mqttclient.poll(c, t, mqttclient.now_ms())!
        c = step.client
        for ev in step.events
            match ev
                MqttMessageArrived(m) => deliver(m)
                _                     => nothing()
    Ok(unit())
```

## The layer, and why

`host`, and the rows are not all the same.

| module | row | why |
| --- | --- | --- |
| `mqtttrans.dial_tcp` | `[io, net]` | `std.net`'s own row for a connect and a read |
| `mqtttrans.dial_tls` | `[net]` | `std.tls`'s own row, which is narrower — and the narrower row is not a rounding |
| `mqttclient.now_ms` | `[time]` | the one function in the package that reads a clock |
| `mqttclient.connect`, `.poll`, `.publish`, `.subscribe`, `.unsubscribe`, `.ping`, `.disconnect`, `.resume` | `[e]` | effect-POLYMORPHIC: whatever the caller's transport costs |
| everything else — options, session, backoff, keep-alive arithmetic | `[]` | values, and the loop that uses them is the caller's |

A caller in a sandbox that grants `[net]` and refuses `[io]` can run
the TLS half of this package and not the TCP half, and can see that
before writing the call.

## The load-bearing interface

`MqttTransport[e]`, and the fact that the clock is an argument.

```novo
pub trait MqttTransport[e]
    fn mqtt_send(self, data: Bytes) -> Result<Int, IoError> [e]
    fn mqtt_recv(self, max: Int) -> Result<Bytes, IoError> [e]
    fn mqtt_close(self) -> ?IoError [e]
    fn mqtt_is_open(self) -> Bool
```

Every driving function in `mqttclient` is generic over it and charged
what it costs (SPEC § 5.6), so the caller decides what an MQTT client
is allowed to do by deciding what to hand it: `[io, net]` for
`MqttTcp`, `[net]` for `MqttTls`, whatever a board declares for a HAL
socket, and **nothing at all** for a buffer in a test.

That last row is the one that pays for the trait.  Most of this package
is timing: a keep-alive that fires at the right moment, a
retransmission that goes out after the right interval, a backoff curve
that a fleet of ten thousand devices does not all follow at once.  A
client whose transport was a fixed `TcpStream` could only be tested
against a broker, which makes every one of those a flaky integration
run.  A client whose transport is a trait and whose clock is an
argument is tested against a byte script and a table of milliseconds.

`mqtt_is_open` is the one method with no effect row, and it is
deliberate: a reconnection loop asks it on every turn, and a question
that costs `[io]` to ask is a question callers start caching — wrongly.

## The clock is an argument

Every function that cares about time takes `now_ms`, a monotonic
millisecond reading the host obtained.  `mqttclient.now_ms()` is the one
function in the package that reads a clock.  Three things follow:

- the keep-alive, the resend schedule and the backoff curve are
  assertable against a table, with no clock in the room;
- a caller driving twenty brokers reads its clock once per turn rather
  than twenty times;
- and the same functions work where there is no `std.time` at all.

`mqttretry` takes its jitter the same way and for the same reason
websocket-codec-nv takes its mask keys as an argument: `[rand]` is a
real effect, and declaring it would make the one decision nobody can
see — what the delay actually was — untestable.

## What a turn does, in order

The order is the protocol, and it is written down because getting it
wrong is invisible:

1. resend anything `mqttsession.due_for_resend` names;
2. send the PINGREQ if `keep_alive_due` says so;
3. fail the connection if `keep_alive_expired` says so;
4. read what the transport has, feed mqtt-codec-nv, drain it;
5. answer what must be answered — PUBACK, PUBREC, PUBREL, PUBCOMP;
6. return the events and the next deadline.

Steps 1 to 3 come **before** the read, because a transport read can
block and a keep-alive checked only after a read that never returns is
a keep-alive that never fires.

## The session is a value the host stores

`MqttSession` is not a file, a database or a directory.  It is a value,
and where it goes is the caller's decision: a server puts it in Redis,
a desktop client in a file, a device in a flash page, a test in a
variable.  Every function in `mqttsession` declares `[]`.

That is what makes QoS mean anything.  A QoS 1 publication is delivered
at least once because the client keeps sending it until a PUBACK
arrives, and "until" spans reconnections and process restarts.  A client
whose in-flight table lived only in memory delivers at-least-once until
the program stops, which is a weaker promise than the one the caller
asked for.

Three decisions inside it that are each individually easy to get wrong:

- **the packet is stored encoded.**  `MqttInflight.packet` holds the
  bytes mqtt-codec-nv produced, not the `MqttPublish` they came from.
  A retransmission must be byte-identical except for the DUP flag, and
  re-encoding is a second chance to differ.
- **a QoS 2 exchange changes shape half-way.**  After a PUBREC the
  PUBLISH must never go out again and the PUBREL must, so
  `advance_to_pubrel` replaces the stored bytes rather than flagging
  the record.
- **packet identifiers come from the session, not the client.**  An
  exchange survives a reconnection, so a counter that restarted with
  the connection would hand out an identifier still in flight.

## What sensorhub would take

`orbit/sensorhub` is an nRF52840-DK duty cycle that reads the on-die
temperature, appends it to a block device and puts it in a BLE
advertising packet.  Publishing the same reading to a broker instead of
— or beside — the radio is the device consumer this package's row on
the grid names, and it is worth being exact about what it would and
would not take.

**It would not take this package.**  `mqtt-nv` is `host`; sensorhub is
`@tier(embedded)` with `layer = "app"`, and the layer table is clear
that `host` does not build into firmware.  What a device runs is:

- **mqtt-codec-nv**, which is `core` and was designed for exactly this:
  `mqttread.embedded_limits()`, `feed_byte` for an interrupt handler,
  and no allocation the caller did not ask for;
- **an `MqttTransport[e]` over the board's own socket** — this
  package's trait, written against the HAL rather than `std.net`.  The
  trait is four methods and has no dependency on anything in this
  package, which is what lets a device take the shape without the
  implementation;
- **`mqttopts.rt_profile`'s numbers**, transcribed: 3.1.1 rather than
  5.0 because the property sections are the part that allocates, a
  Receive Maximum of 1 so the session holds at most one retained
  packet, a 2 KiB maximum packet size, and a keep-alive measured in
  minutes rather than seconds because a radio wake-up costs power.

**What is missing before that is real**, and it is a short list:

- a `@tier(rt)`-shaped session store.  `MqttSession` holds three
  growable lists, and a device wants a fixed-capacity one; the
  functions' signatures do not change, the struct's fields do.
- mqtt-codec-nv's own device claim, which is deferred rather than
  withdrawn: `Result<T, E>` cannot be spelled at `@tier(embedded)`
  today, which its README records against the toolchain.  Until that
  closes, no device builds any of this.
- a socket on the board.  sensorhub's nRF52840 has a radio and no IP
  stack, so the transport is a cellular or Wi-Fi module over a serial
  link — which is precisely the case `mqtt_recv` returning bytes rather
  than packets was shaped for.

So the answer to "what would sensorhub take" is: **the trait and the
profile from this package, the codec from mqtt-codec-nv, and its own
transport** — which is the split working as intended, and is why the
plan's row for this package says "both tiers" while the package itself
is `host`.

## What this does not do, on purpose

- **It does not own a loop.**  No thread, no channel, no callback, no
  `run()`.  Every program worth putting an MQTT client into already has
  a loop.
- **It does not sleep.**  `mqttretry.delay_ms` answers a number and
  `MqttPoll.wait_ms` answers a number; waiting is the caller's.
- **It does not draw randomness.**  The backoff jitter is an argument.
- **It does not store the session.**  It hands the caller a value and
  reads one back.
- **It is not a broker.**  A broker needs a subscription index, retained
  message storage and a client table, none of which are here — though
  `mqtttopic.matches` in the codec is the predicate one would be built
  on, and it is `core` precisely so that it can be.
- **It does not speak MQTT-SN.**  A different protocol with a different
  packet format, for links with no TCP at all; it would be its own row.

## The reference implementation

`rumqttc`, with `paho.mqtt` as the second reading.  Three shapes come
from `rumqttc`: `MqttOptions` as a builder, the split between a client
value and a loop the caller turns, and `Request`/`Event` as the two
directions of the same conversation.

Three things change in the port.  `rumqttc`'s `EventLoop` owns a
`tokio` task and a channel; here the caller owns the loop and there is
no runtime, because a `host` package that brought an async runtime with
it would be choosing one for every program that depends on it.  Its
transport is an enum of the connections it knows about; here it is a
trait, so a caller's own transport is a first-class case rather than a
patch.  And its `Duration`s become millisecond integers taken as
arguments, which is what makes the timing testable and what lets the
same functions run where `std.time` does not exist.

## Status

| item | implemented |
| --- | --- |
| `mqttopts` — `MqttCredentials`, `MqttOptions`, `MqttOptionsFault` | types only |
| `mqttopts.options`, `.rt_profile`, `.unlimited` | no |
| `mqttopts.with_version`, `.with_keep_alive`, `.with_session`, `.with_credentials`, `.with_will` | no |
| `mqttopts.with_limits`, `.with_decode_limits`, `.with_timeouts` | no |
| `mqttopts.faults`, `.connect_packet`, `.keep_alive_ms` | no |
| `mqtttrans` — `MqttTransport[e]`, `MqttTcp`, `MqttTls` | types only |
| `mqtttrans.MQTT_PORT`, `.MQTT_TLS_PORT` | yes — they are constants |
| `mqtttrans.dial_tcp`, `.dial_tcp_timeout`, `.dial_tls`, `.tls_for`, `.stream_of` | no |
| `mqtttrans` — both `MqttTransport` impls | no |
| `mqttsession` — `MqttInflight`, `MqttTopicAlias`, `MqttSession`, `MqttPacketIdTaken`, `MqttSessionFault` | types only |
| `mqttsession.session`, `.clear`, `.is_present`, `.take_packet_id` | no |
| `mqttsession.record_outgoing`, `.advance_to_pubrel`, `.complete_outgoing`, `.outgoing_of`, `.inflight_count` | no |
| `mqttsession.due_for_resend`, `.resume_packets` | no |
| `mqttsession.record_incoming`, `.release_incoming`, `.is_duplicate_incoming` | no |
| `mqttsession.record_subscription`, `.forget_subscription` | no |
| `mqttsession.alias_for`, `.topic_for_alias`, `.learn_alias`, `.clear_aliases` | no |
| `mqttsession.encode`, `.decode`, the `message` impl | no |
| `mqttretry` — `MqttBackoff`, `MqttRetryVerdict` | types only |
| `mqttretry.backoff`, `.never`, `.policy` | no |
| `mqttretry.delay_ms`, `.base_delay_ms`, `.verdict`, `.has_attempts_left` | no |
| `mqttclient` — `MqttLinkState`, `MqttClient`, `MqttClientEvent`, `MqttPoll` | types only |
| `mqttclient.client`, `.state_of`, `.session_of`, `.send_quota`, `.now_ms` | no |
| `mqttclient.connect`, `.poll`, `.publish`, `.publish_packet`, `.resume` | no |
| `mqttclient.subscribe`, `.unsubscribe`, `.ping`, `.disconnect` | no |
| `mqttclient.on_connection_lost`, `.wait_ms` | no |
| `mqttclient.keep_alive_due`, `.keep_alive_expired` | no |
| `mqttclient.effective_receive_maximum`, `.effective_max_packet_size` | no |
| `mqttclienterr` — `MqttClientError`, the `message` impl | type only |
| `mqttclienterr.is_reconnectable`, `.keeps_session`, `.disconnect_reason` | no |
