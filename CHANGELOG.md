# Changelog

All notable changes to syslog-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-10-13

**Every body, and a reader that answers the octets it was given.** The
interface of 0.0.1 is unchanged: every public signature, effect row and
doc comment is the one that release declared. What changed is that all
of them have bodies, and that the refusals a receiver is judged by are
now provoked on purpose instead of being reached only when a test
happened to.

### Added

- `tests/syslog3164_cover_tests.nv` — the refusals and the writing path
  of the BSD half: the strict reader's header faults at their own
  octets, all nine refusals of `parse_timestamp`, the relaxed reader's
  five kept-message cases (§ 4.3.3), the exact round trip of § 5.4's
  first example, the writer's TAG cap and character refusals, and
  `format_timestamp`'s refusal of a stamp that carries a year or an
  offset, which § 4.1.2 has nowhere to put.
- `tests/syslog5424_cover_tests.nv` — the same for the 5424 half: the
  empty-field and bad-PRI refusals, a timestamp fault reported at the
  octet of the WHOLE message rather than of the field, each refusal of
  `parse_timestamp`, the smallest legal message round-tripped, the
  byte-order-mark and half-a-mark cases of § 6.4, the writer's refusal
  of a field its checker refuses, and the SD field carried back through
  the wire element for element.
- `tests/syslogframe_cover_tests.nv` — the states a socket actually
  produces and the API suite had no case for: a detecting reader
  decided from octets it already held rather than from the chunk that
  just arrived, a length prefix cut off before its space, an unfinished
  length that outgrows the limit, a fault after a first frame in one
  drain, a close with nothing pending and a close over a partial frame
  under both framings, a close over a frame already in the buffer, and
  `frame`'s refusal of a framing nobody has decided.
- `tests/syslogerror_cover_tests.nv` — all twenty-one reasons, one
  value per variant, asked of every one of the four functions that
  classify them.  The four API suites meet only the variants their own
  fixtures produce, and none of them calls `message()` at all, so the
  taxonomy is asserted whole here rather than in the parts the example
  messages happened to touch.
- `tests/syslogmsg_cover_tests.nv` — the UTF-8 reader's ranges one at a
  time (RFC 3629's first-octet classes, the continuation bytes, and the
  overlong, surrogate and beyond-U+10FFFF forms), both halves of the
  TAG rule, the RFC 3164 total-length arithmetic on both sides of the
  cap, and `detect_format` on octets that are neither dialect and
  therefore get an answer instead of a guess.
- `tests/syslogpri_cover_tests.nv` — all twenty-four facilities and all
  eight severities with the number and the name the table gives each,
  not just the four and three that RFC 5424 § 6.5's example messages
  reach, plus the severity name read without regard to case and two
  priorities that are not written right refused where they stand.
- `tests/syslogsd_cover_tests.nv` — every way a structured-data field
  can be malformed and every lookup that comes up empty: a name that is
  wrong wherever it sits in the field, an enterprise number that is not
  digits, the reader's refusals provoked one octet at a time, and a
  parameter value read back with § 6.3.3's own escape rule.
- `tests/syslogframe_edge_tests.nv` — the reader's boundaries: the
  seven-octet frame, independently of the API suite's own fixture (see
  `### Known`), the offset counter running across chunks, a fault
  named at the octet of the stream rather than of the chunk, the limit
  against both a declared length and a buffer that never ends, the
  other two trailers, a frame written and read back, a detecting reader
  that settles once and keeps its answer, and a close after a complete
  frame.

### Changed

- `syslog3164.head_text` — **fixed**: the writer put a space between the
  priority and the TIMESTAMP. § 4.1.2's own examples write
  `<34>Oct 11 22:14:15` with none, and with the space this package's
  own reader refused what its own writer wrote (`bad_timestamp` at the
  octet the space landed on). Found by the round-trip case of the new
  suite, which is why it is in it.
- `syslog5424.header_after_pri` — **fixed**: a fault inside the
  TIMESTAMP of a whole message was reported at the octet it holds in
  the field, not in the message. The field reader counts from the field
  and the message's reader is what a log has, so the two offsets now
  agree with every other refusal in `syslogerror.offset_of`.
- Dead code removed: `syslog3164.byte_ints`, `syslog5424.msg_separator`
  (neither had a caller) and `syslog5424.digit_count`'s sign branch
  (its one caller passes a value above a million, so there is no sign
  to take off). The comment that said the branch was needed is gone
  with it rather than left to mislead.
- Three `redundant-bool` sites in `tests/syslog5424_cover_tests.nv`
  rewritten to `not m.msg_utf8` / `m.msg_utf8`; they were written that
  way in this session's own new suite, so no released assertion changed.
- Two `cov: skip` markers added where an arm cannot be taken, each
  naming its reason: `syslogframe.at_stream_offset`'s non-positional
  fault and `syslog5424.write_message`'s `syslogsd.format` fault, both
  of which only keep a match exhaustive. The coverage report is what
  says they are honest: every other instrumented line under `src/` is
  run by a suite. Two of the six excused arms were then mutated on
  purpose — `at_stream_offset`'s arm made to answer a different fault,
  and `syslogpri.from_prival`'s facility arm the same — and no suite
  that was green went red, which is the measurement a marker that says
  "nothing can reach this" needs.

### Known

- **`tests/syslogframe_tests.nv` was red on a one-octet fixture, and the
  fixture is now repaired.** The case `7 <13>abc is one seven-octet
  message` fed `octets_of("7 <13>ab")` — eight octets — while its name
  and both of its assertions are about a seven-octet message behind the
  two-octet prefix `7 `, which is nine octets in all. No receiver can
  answer a seven-octet frame that has not arrived, so the case was
  unsatisfiable as it stood.

  The landing lane correctly refused to edit `tests/` and reported it.
  It was then repaired under `ORACLE-EDITS.md` **class 1** — an
  assertion no implementation can satisfy — because the case's own name
  spells the exact string, and both assertions agree with the name, so
  the artifact settles which side is wrong: the fixture literal had
  dropped one octet. The sibling case below it follows the same
  `"<n> " + n octets` convention, which is the third witness. The edit
  is one character: `octets_of("7 <13>abc")`. The suite is `8 passed,
  0 failed`.
- A `SyslogFramingDetect` value handed to `syslogframe.frame` is
  `SyslogBadFrameLength(0)`. Detection is a question about a stream
  that is arriving, so this is a framing fault and not a kind of
  message fault; there is no `SyslogError` variant for "the framing you
  named cannot write", and inventing one would change the twenty-one
  reasons 0.0.1 published.
- Two workarounds are carried for compiler defects, each with a comment
  at the site naming the filing:
  `codegen-llvm/an-identity-cast-in-a-struct-literal-field-or-tuple-element-reads-as-empty`
  (an inline `as` cast as an aggregate field reads as empty, and an
  inline empty one in a tuple return makes a later `list.concat` die —
  filed separately as
  `codegen-llvm/an-inline-empty-list-cast-in-a-tuple-return-makes-a-later-list-concat-die`),
  and `memory-perceus/a-heap-local-moved-out-on-the-fall-through-is-not-dropped-on-an-early-return`
  (`syslogsd.parse` carries octet indices through its refusals and
  builds its `Str`s after the last one, because a heap local the
  success path moves out is not dropped on an early `return Err`).
  Both are revertible when those filings close.

This release requires novo 0.23.0 or later.

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
