# Fader remote

A one-page phone remote for the motorized faders. It has no server of its own:
each button publishes one device command, as plain text, to a topic on
[ntfy.sh](https://ntfy.sh). The app that owns the Arduino's serial port polls
that topic with an HTTP GET and writes each command to the device.

```
phone page  --POST-->  ntfy.sh/<topic>  <--GET--  serial app  --serial-->  Arduino
```

Nothing ever calls into the laptop; both sides only make outgoing requests.

## The channel (topic)

The topic name is the only secret: anyone who knows it can move the faders. It
is not in this repository. Open the page, tap "Make a random one", and use the
same name in the serial app. The page remembers it on that phone. You can also
open the page as `…/#<topic>` to set it from a link.

## For the serial app: checking for commands

The full specification — algorithm, validation, timing and error handling — is
in [SERIAL-APP-SPEC.md](SERIAL-APP-SPEC.md). In short:

On start-up, note the current Unix time so commands sent earlier are skipped:

```
GET https://ntfy.sh/<topic>/json?poll=1&since=<unix time at start-up>
```

The reply is zero or more lines, each a JSON object, oldest first:

```json
{"id":"MDxO78WCdJkl","time":1795645384,"event":"message","topic":"…","message":"M3,20"}
```

For each line: write `message` plus a newline to the serial port (115200 baud),
and remember `id`. Then keep asking for what came after it:

```
GET https://ntfy.sh/<topic>/json?poll=1&since=<last id>
```

- Poll every 5 seconds or slower; the public server limits how often one client
  may ask. An empty reply means there is nothing new.
- Forward a message only if it is a valid device command (see below), so nothing
  else can be pushed to the serial port.
- ntfy keeps messages for about 12 hours, which is why `since` matters.

## Commands

The page only ever publishes these, exactly as the sketch expects them:

| Command | Effect |
|---|---|
| `G<n>,<v>` | move fader n (1 score, 2 comm, 3 solv, 4 fits, 5 take) to v (0–1000) |
| `A<v1>,<v2>,<v3>,<v4>,<v5>` | move all five faders |
| `R` | all faders to the middle |
| `T<sec>` | idle timeout in seconds, 0 = off |
| `I1` / `I0` | score-fader detents on / off |
| `M0` `M1` `M2` `M3` | normal / random one / random all / wave |
| `M2,<sec>` | random all, every `<sec>` seconds |
| `M3,<sec>` | wave for `<sec>` seconds, then back to normal |
| `?` | print status on the serial line |

To send one without the page:

```sh
curl -d "M3,20" https://ntfy.sh/<topic>
```
