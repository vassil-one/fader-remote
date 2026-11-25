# Spec: reading the command channel from the serial-port app

For the app that owns the Arduino's USB serial port. It describes how that app
finds out, over HTTP, that the phone remote has sent a command, and what to do
with it.

The behaviour described here was checked against the public ntfy.sh server on
2026-11-25. The one exception is the rate limit in section 7, which was not
measured.

## 1. Overview

```
phone page  --POST-->  ntfy.sh/<channel>  <--GET--  serial app  --serial-->  Arduino
```

- The phone page publishes each command as a short text message to a channel
  (an ntfy "topic").
- The serial app asks ntfy.sh for new messages with an HTTP GET and writes each
  one to the serial port.
- Nothing calls into the serial app. It only makes outgoing HTTPS requests, so
  it works behind NAT, a phone hotspot or conference wifi.

## 2. Configuration

| Setting | Value |
|---|---|
| Server | `https://ntfy.sh` |
| Channel | The name shown/created on the phone page. 8–64 characters from `A–Z a–z 0–9 - _`. |
| Poll interval | 5 seconds (see section 7) |

The channel name is the only secret: anyone who knows it can send commands.
Read it from a config file or environment variable. Do not hardcode it, log it,
or commit it.

## 3. The request

```
GET https://ntfy.sh/<channel>/json?poll=1&since=<cursor>
```

- `poll=1` makes the server answer immediately and close the connection.
- `since` selects which messages to return:
  - a Unix time in seconds: messages published at or after that time;
  - a message id: messages published after that message.
- No headers, authentication or body are needed.

## 4. The response

Status `200`, content type `application/x-ndjson`: zero or more lines, each one
a complete JSON object, oldest message first.

```json
{"id":"kEvnMxc6F5ID","time":1795645685,"expires":1795688885,"event":"message","topic":"<channel>","message":"M3,20"}
{"id":"hjaib3OO1NTL","time":1795645686,"expires":1795688886,"event":"message","topic":"<channel>","message":"G3,800"}
```

| Field | Use |
|---|---|
| `id` | Unique id of the message. The cursor for the next request. |
| `time` | Unix time (seconds) the message was published. |
| `event` | Act only on `"message"`. Ignore any other value. |
| `message` | The command text, e.g. `M3,20`. |
| others | Ignore. |

**Nothing new** is a `200` with an empty body (0 bytes). It is not an error and
not JSON; do not try to parse it.

## 5. Algorithm

State kept in memory: `startTime`, `cursor`, `seen` (set of ids).

1. **Start-up.** Set `startTime` to the current Unix time and `cursor` to
   `startTime`. This skips commands sent before the app started. The server
   keeps messages for about 12 hours, so without this an old command would be
   replayed.
2. **Poll.** `GET …/json?poll=1&since=<cursor>` with a 10-second timeout.
3. **For each line** of the reply, in order:
   1. Parse it as JSON. Skip the line if it does not parse.
   2. Skip it unless `event` is `"message"`.
   3. Skip it if `time` is less than `startTime`, or `id` is already in `seen`.
   4. Add `id` to `seen` and set `cursor` to `id`.
   5. Trim `message`. Skip it unless it is a valid command (section 6).
   6. Write `message` followed by `\n` to the serial port.
4. **Wait** the poll interval and go to step 2.

Why step 3.3 matters: if the server no longer knows the id in `since` (it
expired, or is wrong), it returns **every** stored message instead of an error.
Checking `time` and `seen` makes that harmless. Checking `seen` also covers a
time-based `since`, which includes messages from that same second.

`cursor` moves forward even when a command is rejected in step 3.5, so a bad
message is skipped once and never retried.

## 6. Valid commands

Forward a message only if it matches one of these exactly (whole string,
upper-case letters). Reject everything else and log it. This is what prevents
anyone who learns the channel name from pushing arbitrary bytes to the device.

| Pattern (regular expression) | Meaning |
|---|---|
| `^G[1-5],(\d{1,3}\|1000)$` | Move fader 1–5 to 0–1000 |
| `^A(\d{1,3}\|1000)(,(\d{1,3}\|1000)){4}$` | Move all five faders |
| `^R$` | All faders to the middle |
| `^T\d{1,5}$` | Idle timeout in seconds, 0 = off |
| `^I[01]$` | Score-fader detents on / off |
| `^M[0-3]$` | Mode: normal / random one / random all / wave |
| `^M[23],\d{1,4}$` | Random-all interval, or wave duration, in seconds |
| `^\?$` | Print status on the serial line |

Fader numbers: 1 score, 2 comm, 3 solv, 4 fits, 5 take.

Serial settings: 115200 baud, each command terminated by a newline. The device
replies on the same port with lines starting `debug ` (for example `debug M,3,20`);
those are separate from the fader-position JSON lines the app already reads.

## 7. Timing and limits

- **Poll every 5 seconds.** The public server limits how many requests one
  client may make. Five seconds is a conservative figure that was not measured;
  if the app gets `429`, poll less often.
- **Delivery delay.** A published command becomes visible to polling about one
  second after it is sent. With a 5-second interval, expect a command to reach
  the device 1–6 seconds after the button is pressed.
- **Retention.** Messages stay on the server for about 12 hours.

## 8. Errors

| Situation | What to do |
|---|---|
| No network, DNS failure, timeout | Keep `cursor`. Try again at the next interval. Commands sent meanwhile are delivered when the connection returns. |
| `429 Too Many Requests` | Keep `cursor`. Wait 30 seconds, then resume. |
| Any other non-200 status | Keep `cursor`. Try again at the next interval. |
| A line that is not valid JSON | Skip that line, carry on with the rest. |
| Serial port not open | Drop the command and log it. Do not queue it: a fader move that arrives minutes late is worse than one that never arrives. |

The serial app must never stop reading fader positions because the command
channel is unreachable. Run the polling independently of the serial read loop.

## 9. Trying it by hand

```sh
CH=<channel>

# What the phone does
curl -d "M3,20" https://ntfy.sh/$CH

# What the serial app does (everything from the last minute)
curl "https://ntfy.sh/$CH/json?poll=1&since=$(( $(date +%s) - 60 ))"
```

## 10. Optional: no polling delay

Dropping `poll=1` turns the same GET into a stream: the connection stays open
and each new message arrives as a line the moment it is published.

```
GET https://ntfy.sh/<channel>/json?since=<cursor>
```

The stream also carries lines with `event` set to `open` and `keepalive`;
step 3.2 already ignores them. Reconnect with the latest `cursor` whenever the
connection drops. Everything else in this spec applies unchanged. Start with
polling; move to this only if the 1–6 second delay matters.
