# Changelog

All notable changes to syslog-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `syslogframe` — the load-bearing interface, and the decision it
  rests on is that a FRAMING fault and a MESSAGE fault are two
  different things. Under the octet counting of RFC 6587 § 3.4.1 a
  length that is not a length leaves a receiver holding a stream it
  cannot cut, and there is nothing in that stream to resynchronise on,
  so the transport is finished; a message that does not parse costs one
  message and no more, because the framing already settled where the
  next one starts. `syslogerror.is_framing_fault` is the division, and
  a receiver that did not have it would either drop a connection over
  one bad message or keep writing slices of somebody else's message
  into a log. The reader is fed and drained — the host pumps chunks in,
  frames come out, one per call, with `frame: None` meaning "not yet"
  rather than "never" — and a frame is OCTETS, not a parsed message, so
  a relay forwards exactly what it received. The framing is settled
  once per stream and never per message, because an RFC 3164 message
  can begin with a digit and a stream of them read as octet-counted
  would be cut at lengths taken out of somebody's log text.
- `syslogmsg` — one message value for both formats, because a
  collector that held two would make every filter, router and renderer
  downstream be written twice. RFC 3164's TAG arrives in `app_name`
  and its `[1234]` in `procid`, which is what they are. The timestamp's
  year and offset are `?Int`: an RFC 3164 timestamp (§ 4.1.2) has
  neither, it names no instant, and filling in the receiver's own year
  would be wrong for exactly the messages from the minutes around
  midnight on the thirty-first of December.
  `timestamp_resolved` is where a caller that knows supplies them,
  under its own name. The MSG is `[u8]` and not `Str`, because RFC 5424
  § 6.4 allows any octet there and a relay that replaced what it could
  not decode would be rewriting what it was only forwarding.
- `syslogpri` — the priority as the pair it is rather than the number
  it is encoded as, so that `prival / 8` and `prival % 8` are written
  once instead of at every call site that thought it knew which was
  which. `is_at_least_as_severe` exists because Emergency is 0 and
  Debug is 7, so the comparison everybody wants is `<=` and reads
  backwards; written the other way it gives a filter that passes Debug,
  drops Emergency and looks from the outside like a working filter.
  There is no facility lookup by name: RFC 5424 Table 1 gives
  facilities 4 and 10 the same description and 9 and 15 the same
  description, and the short configuration-file words are not in the
  RFC and disagree between systems.
- `syslogsd` — structured data with RFC 5424 § 6.3.3's escaping as a
  function rather than as something each caller writes. The three
  characters are `"`, `\` and `]`, and the bracket is the one that is
  missed: unescaped it ends the element early and one message arrives
  as two malformed ones, which only happens once a user puts a bracket
  in a log line. `unescape_param_value` carries the other half of the
  same section — a backslash before any other character is kept,
  together with the character after it — without which every Windows
  path in every log is silently corrupted.
- `syslog5424` and `syslog3164` — both formats parsed and formatted,
  from octets and to octets. RFC 3164 has two parsers, and that is
  § 4.3.3 taken literally: `parse` refuses what the grammar refuses,
  and `parse_relaxed` has no error channel at all, because a relay
  meeting a packet with no valid priority must insert one and treat the
  whole packet as content rather than drop it. A collector that refused
  those would be discarding exactly the senders nobody is watching.
- `syslogerror` — twenty-one reasons, each naming the RFC section it
  comes from, with `is_framing_fault` dividing the ones that end a
  connection from the ones that cost one message, `offset_of` counted
  from the first octet the reader was ever given rather than from the
  chunk that happened to contain the fault, and `code` for a metric
  label that carries nothing a sender chose.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suites reaches `not implemented:
  syslog-nv.<module>.<fn>`.
- **No transport, and none is planned here.** RFC 5425 (TLS) and RFC
  5426 (UDP) define how these messages travel. The octet counting RFC
  5425 requires is in `syslogframe`; the socket is a `host` package's
  job.
- **No relay or collector policy.** RFC 5424 § 4 and RFC 3164 § 4.3
  describe filling in a missing timestamp and hostname, adding an
  `origin` element and deciding what to forward where. Those are
  decisions a program makes, and this package gives the parts they are
  built from without making any of them.
- **No dependency on calendar-nv**, which was the one candidate. A
  `CivilDate` requires a year and an RFC 3164 timestamp has none, so a
  timestamp type whose year and offset can be absent has to exist here
  regardless; taking calendar-nv for the RFC 5424 half alone would put
  two timestamp types in one message value.
