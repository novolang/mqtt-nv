# Changelog

All notable changes to mqtt-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md).

### Changed

- **`mqtttrans.dial_tls` and `.tls_for` name `MqttTlsConfig`**, a struct
  this package declares, where 0.0.1 named a bare `TlsConfig`.  No such
  type exists in any build: the standard library's TLS surface is not
  published, and the name type-checked in 0.0.1 only because the
  undefined-type check (`E2033`) had not landed.  The fields are the
  ones the two functions always meant — `hostname` for SNI and the
  certificate name match, `verify_peer`, and `ca_bundle_path`, which is
  the field a broker behind a private certificate authority needs.  A
  0.0.1 consumer could not have called either function, so nothing that
  compiled before stops.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `mqtttrans` — `MqttTransport[e]`, the trait every driving function is
  generic over, with `MqttTcp` (`[io, net]`) and `MqttTls` (`[net]`)
  over the standard library and `mqtt_is_open` as the one method with
  no effect row.
- `mqttclient` — the client value, the five states a link can be in,
  the eight events a turn can produce, and the nine functions a host
  calls from its own loop.  `now_ms` is the only function in the
  package that reads a clock.
- `mqttsession` — the state that outlives a socket as a value the host
  stores: the in-flight table, the received QoS 2 identifiers, the
  subscriptions, the packet-identifier allocator and both alias tables,
  with `encode` and `decode` for whatever the caller stores it in.
- `mqttopts` — the CONNECT as a builder, the three profiles (default,
  device, unlimited), and `faults`, which answers every reason a set of
  options cannot be used rather than the first.
- `mqttretry` — a capped exponential backoff as arithmetic, with the
  jitter draw and the attempt count as arguments.
- `mqttclienterr` — the four audiences a failure can have, and
  `is_reconnectable` / `keeps_session`, the two questions every caller
  has to answer about one.

### Known

- **`MqttTransport[e]` is the load-bearing interface.**  Every driving
  function is generic over it and charged what it costs, so a caller
  decides what an MQTT client may do by deciding what to hand it — and
  a test hands it a buffer and pays nothing.
- **The clock is an argument.**  Every function that cares about time
  takes `now_ms`, so the keep-alive, the resend schedule and the
  backoff curve are tables rather than integration runs, and the same
  functions work where `std.time` does not exist.
- **The jitter is an argument too**, for the reason
  websocket-codec-nv's mask keys are: `[rand]` would make the one
  decision nobody can see untestable.
- **The order inside a turn is the protocol**: resend, ping, expire,
  read, answer.  The first three come before the read, because a
  keep-alive checked only after a read that never returns never fires.
- **The session is a value and never a file.**  Every row in
  `mqttsession` is `[]`; where it is stored is the caller's decision.
- **A QoS 2 exchange changes shape half-way**, so `advance_to_pubrel`
  replaces the stored bytes rather than flagging the record.
- **Packet identifiers come from the session**, because an exchange
  survives a reconnection and a per-connection counter would hand out
  one still in flight.
- **Two effect rows, not one.**  `dial_tcp` is `[io, net]` and
  `dial_tls` is `[net]`, which is what the two standard library
  surfaces declare; the narrower row is not a rounding.
- **No device claim.**  The package is `host`.  What a device runs is
  mqtt-codec-nv, this package's trait written against a HAL socket, and
  `mqttopts.rt_profile`'s numbers; the README says what is still
  missing before that is real.
- **One `core` dependency**, mqtt-codec-nv, itself an interface today.

### Design notes

The client is ported from `rumqttc`, with `paho.mqtt` as the second
reading.  Three shapes come across unchanged: `MqttOptions` as a
builder, the split between a client value and a loop the caller turns,
and requests and events as the two directions of one conversation.
Three change.  `rumqttc`'s `EventLoop` owns a `tokio` task and a
channel; here the caller owns the loop, because a `host` package that
brought an async runtime with it would be choosing one for every
program that depends on it.  Its transport is an enum of the
connections it knows about; here it is a trait, so a caller's own
transport is an ordinary case rather than a patch.  And its `Duration`s
become millisecond integers taken as arguments, which is what makes the
timing assertable against a table and lets the same functions run where
`std.time` does not exist.

A device consumer takes the trait and `mqttopts.rt_profile`'s numbers
from this package, mqtt-codec-nv for the packets, and writes its own
transport over the board's socket.  It does not take this package,
which is `host` and does not build into firmware.
