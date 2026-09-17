# syslog-nv

Syslog is the way computers have sent event messages to each other
since the 1980s. It is written down twice: the current format in
[RFC 5424](https://www.rfc-editor.org/rfc/rfc5424), and the older BSD
format in [RFC 3164](https://www.rfc-editor.org/rfc/rfc3164). A third
document, [RFC 6587](https://www.rfc-editor.org/rfc/rfc6587), says how
messages are separated when they are sent over a stream. This package
reads and writes both formats, and cuts a stream into messages, in
novo-lang. It opens no socket and reads no clock.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What syslog is

A syslog message is a short record of one event. It begins with a
**priority**, written `<34>`: a `<`, a number, a `>`. The number is not
a scale. RFC 5424 section 6.2.1 defines it as a **facility** multiplied
by eight plus a **severity**. The facility says which part of a system
spoke, and the severity says how serious the event is.

| | |
| --- | --- |
| Facilities | 24, numbered 0 to 23. Numbers 16 to 23 are for local use. |
| Severities | 8, numbered 0 to 7. |
| Priority values | 0 to 191. |

The severities run from 0 for Emergency to 7 for Debug, so a **lower
number is more severe**.

After the priority the two formats differ.

An **RFC 5424** message has a version number, then five fields
separated by single spaces: a timestamp, the sending machine's name,
the sending program's name, a process identifier and a message
identifier. Any field a sender cannot supply is written as the
**NILVALUE**, a single `-`. Then comes the **structured data**, a list
of named elements each holding `name="value"` pairs, and finally the
free-text content.

```
<165>1 2003-10-11T22:14:15.003Z mymachine.example.com evntslog - ID47 [exampleSDID@32473 iut="3"] An application event log entry
```

An **RFC 3164** message has a timestamp written `Mmm dd hh:mm:ss`, the
sending machine's name, and then the content. The content conventionally
starts with a **tag**, the program's name, followed by the process
identifier in square brackets and a colon. RFC 3164 has no message
identifier and no structured data.

```
<34>Oct 11 22:14:15 mymachine su: 'su root' failed for lonvick on /dev/pts/8
```

The RFC 3164 timestamp carries **no year and no time zone**, and it is
in the sender's local time. It does not name an instant. Whoever reads
it has to supply the year and the zone from somewhere else.

Over UDP one datagram is one message and nothing more is needed. Over
TCP and TLS there is no boundary, so RFC 6587 section 3.4 gives two
ways to find one. **Octet counting** puts the length in front: a
decimal number, a space, then that many bytes. **Non-transparent
framing** puts a delimiter after, usually a line feed.

## Install

```
novo pkg add syslog-nv
```

## Example

```novo
use syslogframe
use syslogmsg
use syslog5424
use syslogpri

fn main() [io]
    // The reader that cuts a TCP stream into messages. The sender in
    // this example separates its messages with a line feed.
    let reader = syslogframe.reader(SyslogNonTransparent(SyslogTrailerLf),
                                    syslogframe.default_limits())

    // One message, as it arrived from the network. The host read
    // these bytes; this package never touches a socket.
    let chunk = bytes.to_byte_list(bytes.from_str("<34>1 - myhost su - - - hello\n"))

    // Hand the bytes over and take back the messages they completed.
    match syslogframe.drain(reader, chunk)
        Err(e) => println("the stream is unreadable: ${e.message()}")
        Ok(drained) =>
            for f in drained.frames
                // A frame is the bytes of one message. Parsing it is a
                // separate step, so a relay can forward it untouched.
                match syslog5424.parse(f.octets)
                    Err(e) => println("one bad message: ${e.message()}")
                    Ok(m)  =>
                        let how_bad = syslogpri.severity_name(m.pri.severity)
                        println("from ${m.hostname} at ${how_bad}")
                        match syslogmsg.msg_text(m)
                            Some(text) => println(text)
                            None       => println("the content is not text")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: syslog-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `syslogerror` | Every refusal, whether it costs one message or the whole connection, and where in the stream it was found. |
| `syslogpri` | The priority: 24 facilities, 8 severities, the arithmetic between them and the number on the wire. |
| `syslogsd` | RFC 5424 structured data: elements, parameters, the escaping rules and the rules on names. |
| `syslogmsg` | The message value both formats produce, the timestamp with its optional year and offset, and the builders. |
| `syslog5424` | RFC 5424 parsed and written, with its timestamp syntax and its field limits. |
| `syslog3164` | RFC 3164 parsed and written, strictly and the way a relay has to, with its timestamp syntax. |
| `syslogframe` | RFC 6587 framing: a reader that takes bytes and hands back messages, and a writer that frames one. |

## How to choose an entry point

**`syslogframe.reader` is for a stream.** Over TCP or TLS the bytes
arrive in pieces that have nothing to do with message boundaries. Make
a reader, hand it every chunk, and take the messages back. Over UDP
there is no framing to do, so hand the datagram straight to a parser.

**`syslogframe.feed` takes one message at a time and
`syslogframe.drain` takes all of them.** Use `feed` when a caller acts
on each message before reading the next, and `drain` when it wants the
batch.

**`syslog5424.parse` and `syslog3164.parse` are strict.** They refuse
what the specification refuses. Use them when a sender's own output is
being checked, or when both ends are known.

**`syslog3164.parse_relaxed` cannot fail.** It always returns a
message. Use it in a collector that receives from senders it does not
control, which is nearly every collector.

**`syslogmsg.detect_format` says which format some bytes are in.** Use
it on a port that receives both, then call the matching parser.

**A relay does not have to parse at all.** A frame is the bytes of one
message. Forwarding those bytes is the whole job.

## The rules a user needs

1. **A lower severity number is more severe.** Emergency is 0 and
   Debug is 7 (RFC 5424 section 6.2.1, Table 2). Use
   `syslogpri.is_at_least_as_severe` rather than comparing the numbers.
   A filter written the other way round passes Debug and drops
   Emergency, and looks from the outside like a working filter.
2. **A framing fault ends the connection; a message fault does not.**
   `syslogerror.is_framing_fault` says which happened. After a framing
   fault the receiver no longer knows where the next message starts,
   and under octet counting there is nothing in the stream to
   resynchronise on. After a message fault the boundary is still known,
   so the receiver reports that message and reads the next.
3. **Three characters are escaped inside a structured-data value:**
   `"`, `\` and `]`, each with a backslash (RFC 5424 section 6.3.3).
   The bracket is the one that is missed. Unescaped, it ends the
   element early, and one message arrives as two malformed ones.
4. **A backslash before any other character is kept**, together with
   the character after it (RFC 5424 section 6.3.3). A parser that
   removed it would turn `C:\temp` into `C:temp`.
5. **An RFC 3164 timestamp has no year and no time zone.** It parses to
   a `SyslogTimestamp` whose `year` and `offset_minutes` are absent.
   `syslogmsg.timestamp_resolved` is where a caller supplies them.
   Nothing here guesses, because a guess is wrong for exactly the
   messages around midnight on the thirty-first of December.
6. **In an RFC 3164 timestamp a day below 10 is written with a leading
   space**, not a leading zero (RFC 3164 section 4.1.2). The fifth of
   February is `Feb  5`, with two spaces.
7. **An RFC 5424 timestamp carries at most six fractional digits**
   (RFC 5424 section 6.2.3.1). Nine digits is invalid, and the section
   gives that exact case as its example of an invalid timestamp.
8. **A header field may not contain a space and may not be empty.** The
   five fields are separated by single spaces, so a space inside one
   shifts every later field by one. A sender with nothing to say writes
   the NILVALUE `-` (RFC 5424 section 6.2).
9. **The message content is bytes, not text.** RFC 5424 section 6.4
   allows any byte there. `syslogmsg.msg_text` answers `None` when the
   content is not UTF-8, rather than replacing what it could not
   decode.
10. **The byte order mark is a declaration and not content.** RFC 5424
    section 6.4 requires it in front of UTF-8 content. It is removed
    when a message is parsed and written when one is formatted, so a
    caller never sees it.
11. **A structured-data identifier appears at most once in a message**
    (RFC 5424 section 6.3.2). An identifier with no `@` in it is
    reserved by IANA; a private one is written
    `name@<private enterprise number>`.
12. **The priority is written without leading zeros** (RFC 5424 section
    6.2.1). `<9>` is a priority and `<09>` is refused.
13. **RFC 3164 has no message identifier and no structured data.**
    Writing an RFC 5424 message in RFC 3164 drops both. A program that
    needs them to survive is writing RFC 5424.
14. **An RFC 3164 message is at most 1024 bytes** (RFC 3164 section
    4.1). This package refuses to write or read a longer one. RFC 5424
    section 6.1 instead asks a receiver to support 480 bytes and
    recommends 2048; those two are advice about the network and are not
    enforced here.
15. **Non-transparent framing cannot carry a message containing the
    delimiter.** RFC 6587 section 3.4.2 gives no escape for it.
    `syslogframe.frame` refuses rather than writing a message that
    would arrive as two. Octet counting carries any bytes.
16. **A reader holds one framing for a whole stream.** Deciding per
    message is unsafe, because an RFC 3164 message can begin with a
    digit. `SyslogFramingDetect` decides once from the first byte and
    holds that answer.
17. **The end of a connection is a message under non-transparent
    framing and a fault under octet counting.** RFC 6587 section 3.4.2
    does not require a delimiter on the last message, so
    `syslogframe.finish` returns it. Under octet counting a short frame
    is truncated, because the length said how much was coming.
18. **There is no way to look a facility up by name.** RFC 5424 Table 1
    gives facilities 4 and 10 the same description, and 9 and 15 the
    same description. `syslogpri.facility_description` returns the
    table's words for a person to read. Severity names are distinct, so
    `syslogpri.severity_from_name` exists.
19. **The time is always an argument.** No function in this package
    reads a clock. A timestamp is parsed from a message or supplied by
    the caller.

## What is not included

- **A transport.** Nothing here opens a socket, sends a datagram,
  connects to a collector or retries. Every function takes bytes the
  caller already has and returns bytes the caller sends.
- **A clock.** See rule 19. A sender that wants the current time in a
  message reads it and passes it in.
- **A binding to the C `syslog(3)` interface.** `openlog`, `syslog` and
  `closelog` write to a local socket through the C library, which is a
  transport and a foreign library at once. This package is a reader and
  a writer of the format.
- **Facility names beyond RFC 5424 Table 1's descriptions.** The short
  words used in configuration files are one implementation's
  configuration language and are in none of these documents. They also
  disagree between systems: the constant a C program calls `LOG_CRON`
  is facility 9 on Linux and facility 15 on several other systems.
- **Relay and collector behaviour.** RFC 5424 section 4 and RFC 3164
  section 4.3 describe what a relay does: fill in a missing timestamp
  and hostname, add an `origin` element, decide what to forward where.
  Those are policies a program chooses. This package gives the parts
  they are built from and chooses none of them.
- **A civil date type.** `SyslogTimestamp` holds the fields a message
  carried, with the year and the offset marked absent when the format
  omitted them. A caller that wants date arithmetic passes those fields
  to a calendar library.
- **Message signing.** RFC 5848 defines signed syslog messages as
  structured-data elements. They can be represented with `syslogsd`,
  and nothing here generates or checks a signature.
- **RFC 5425 and RFC 5426.** Those define the TLS and the UDP
  transport. The octet counting RFC 5425 requires is here; the
  transport is not.

## Related packages

- [logging-nv](https://novo-lang.org/packages/logging-nv) is a
  program's own logging: levels, targets, typed fields and the sinks
  that write them. Take it to produce log records. Take this package to
  read or write the syslog wire format.
- [logging-core-nv](https://novo-lang.org/packages/logging-core-nv) is
  the part of that which performs no input or output, for a library or
  for a microcontroller.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) holds civil
  dates and times and parses RFC 3339. Take it to do arithmetic on a
  timestamp this package parsed.
- [tracing-nv](https://novo-lang.org/packages/tracing-nv) is
  distributed tracing, which answers a different question about the
  same systems: how one request moved between them.

## Tests

```bash
novo test tests/syslogpri_tests.nv      # the priority, and the severity ordering
novo test tests/syslogsd_tests.nv       # structured data and its escaping
novo test tests/syslog5424_tests.nv     # RFC 5424, against the RFC's own examples
novo test tests/syslog3164_tests.nv     # RFC 3164, strict and relaxed
novo test tests/syslogframe_tests.nv    # the two framings, and the two kinds of fault
novo test tests/syslogcover_tests.nv    # the message value, and every public function
```

The normative sources are RFC 5424 for the current format, RFC 3164 for
the BSD format and RFC 6587 for the framing. The fixtures are the
example messages those documents publish: RFC 5424 section 6.5, RFC
5424 section 6.2.3.1 for the timestamps, and RFC 3164 section 5.4. The
reference implementations are the Rust crate `syslog_loose`, for a
parser that never fails on a real-world message, and Python's `syslog`
module, for the facility and severity tables.

The suite asserts that `<34>` is facility 4 and severity 2, that a
priority above 191 is refused, that Debug is not at least as severe as
Warning, that a closing bracket in a parameter value is escaped and a
backslash before a letter is not removed, that a nine-digit fractional
second is refused, that `Feb  5` parses and `Feb 05` does not, that a
relaxed RFC 3164 parse of `Use the BFG!` returns a message with
priority 13, that a message containing a line feed cannot be
line-framed, and that the end of a connection is a message under one
framing and a fault under the other.

The tests compile today and fail at run, each on the
`not implemented: syslog-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `SyslogError`, `SyslogPri`, `SyslogFacility`, `SyslogSeverity` and the other types | the types are declared |
| `syslogerror.is_framing_fault`, `.offset_of`, `.code`, `SyslogError.message` | no |
| `syslogpri.pri`, `.prival`, `.from_prival`, `.parse_pri`, `.format_pri` | no |
| `syslogpri.facility_number`, `.severity_number`, `.facility_from_number`, `.severity_from_number` | no |
| `syslogpri.facility_description`, `.severity_name`, `.severity_from_name`, `.is_at_least_as_severe` | no |
| `syslogsd.param`, `.element`, `.value_of`, `.values_of`, `.element_of` | no |
| `syslogsd.escape_param_value`, `.unescape_param_value`, `.is_valid_sd_name` | no |
| `syslogsd.is_registered_sd_id`, `.enterprise_number`, `.check`, `.parse`, `.format` | no |
| `syslogmsg.message` and the eight `with_` builders | no |
| `syslogmsg.msg_text`, `.timestamp`, `.timestamp_resolved`, `.detect_format`, `.nilvalue`, `.check` | no |
| `syslog5424.parse`, `.parse_header`, `.format` | no |
| `syslog5424.parse_timestamp`, `.format_timestamp`, `.field_limits`, `.is_printable_ascii` | no |
| `syslog5424.bom`, `.version`, `.minimum_supported_octets`, `.recommended_supported_octets` | no |
| `syslog3164.parse`, `.parse_relaxed`, `.format`, `.default_pri`, `.split_tag` | no |
| `syslog3164.parse_timestamp`, `.format_timestamp`, `.month_abbreviations` | no |
| `syslog3164.max_message_octets`, `.max_tag_chars` | no |
| `syslogframe.reader`, `.feed`, `.take`, `.drain`, `.finish` | no |
| `syslogframe.default_limits`, `.small_limits`, `.pending_len`, `.offset`, `.framing_of` | no |
| `syslogframe.detect_framing`, `.trailer_octets`, `.frame` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
