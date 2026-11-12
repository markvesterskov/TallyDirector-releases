# TallyDirector API (version 1.1)

This API lets other software (a Bitfocus Companion module, a show controller, your own scripts) list the tallies TallyDirector knows, follow their state live and control them. It runs inside the TallyDirector app on the Director machine.

## Changes in 1.1

All additions: a 1.0 client keeps working unchanged.

* `hello` has `api_minor` (1).
* Every tally has `connection` (`"usb"` or `"wifi"`), `host` (the machine it is connected to) and `rssi` (WiFi signal).
* New message `switcher`: the video switcher TallyDirector follows (type, address, connection state, model). Read only.

## Connecting

* WebSocket, plain text JSON frames: `ws://<director-ip>:9101/`
* Discovery: TallyDirector advertises the mDNS service `_tallydirector._tcp`. Its TXT record has `api=9101`, the API port. Read the port from that TXT key, and fall back to 9101 if the key is missing. (In a Bitfocus Companion module: `bonjourQueries` for `_tallydirector._tcp`.)
* Any number of clients may connect at the same time. Each gets the same messages.
* There is no authentication. The API is meant for the show network only.
* Web pages cannot connect. Browsers send an `Origin` header with every WebSocket handshake, and TallyDirector refuses those with HTTP 403, so that a page opened somewhere on the network cannot control the tallies. Connect from a program instead (Node.js, Python, a Companion module), and do not set an origin in its WebSocket options.
* A message to TallyDirector can be 64 KB at most. A bigger one closes the connection.
* Keepalive: the Director sends a WebSocket ping frame every 2 s and closes the connection after 10 s without a pong. Standard clients (the `ws` package in Node.js) answer pings on their own. Clients send no application level ping.
* Reconnect: when the connection drops, reconnect every 2 s until it works. All state arrives again right after connecting, so the client needs no memory across connections.

## Message format

Every message is one JSON object with a `type` field. Clients must ignore fields they don't know and messages with a `type` they don't know.

The version has two parts, both in `hello`. `api` changes only for a breaking change. `api_minor` goes up for additions (new fields, new messages), which older clients simply ignore. A client written for 1.1 works with any `api` 1, and uses the 1.1 additions when `api_minor` is 1 or more.

## Director to client

### `hello`

The first message on every connection.

```json
{"type": "hello", "api": 1, "api_minor": 1, "version": "0.4.1"}
```

* `api`: the major API version, an integer. A client written for version 1 should refuse to work with another major version and say so.
* `api_minor`: the minor version. Missing (treat as 0) on TallyDirector before 1.1.
* `version`: the TallyDirector app version, for display only.

### `tallies`

The complete list of tallies. Sent once right after `hello`, then every time anything in the list changes. It always replaces the previous list: the client never merges or patches. Bursts of changes are combined, at most one message every 50 ms.

```json
{
  "type": "tallies",
  "tallies": [
    {
      "mac": "AA:BB:CC:DD:EE:01",
      "device": 1,
      "name": "Stage left",
      "state": "program",
      "osc_state": "program",
      "switcher_state": "off",
      "online": true,
      "links": [
        {"kind": "usb", "hostname": "STAGE-PC-1", "ip": "10.48.10.21"}
      ],
      "battery": null,
      "product": "TallyLED",
      "fw": "0.3.1",
      "connection": "usb",
      "host": "STAGE-PC-1",
      "rssi": null
    },
    {
      "mac": "AA:BB:CC:DD:EE:02",
      "device": 2,
      "name": null,
      "state": "off",
      "osc_state": "off",
      "switcher_state": "off",
      "online": false,
      "links": [],
      "battery": null,
      "product": "TallyLCD",
      "fw": "0.3.1",
      "connection": null,
      "host": null,
      "rssi": null
    },
    {
      "mac": "AA:BB:CC:DD:EE:03",
      "device": 3,
      "name": "Lectern",
      "state": "preview",
      "osc_state": "off",
      "switcher_state": "preview",
      "online": true,
      "links": [
        {"kind": "wifi", "ip": "10.48.20.53", "rssi": -61}
      ],
      "battery": {"percent": 84, "charging": false, "minutes_left": 190},
      "product": "TallyLCD",
      "fw": "0.3.1",
      "connection": "wifi",
      "host": "DIRECTOR-MAC",
      "rssi": -61
    }
  ]
}
```

Fields of a tally:

| Field | Type | Meaning |
|---|---|---|
| `mac` | string | Stable identity of the physical tally, upper case with colons. Never changes. |
| `device` | integer or null | Device number, the number every command uses. Unique among tallies: giving a number to another tally in TallyDirector leaves the one that had it without a number. Null when the tally has none. A tally can be moved to another number, so don't use it as an identity across sessions. |
| `name` | string or null | Name set in the Director, for display. |
| `state` | state | What the tally shows right now. Use this for button colours. |
| `osc_state` | state | The part set through OSC, this API or the Director's buttons. |
| `switcher_state` | state | The part set by a connected video switcher. |
| `online` | boolean | True while the tally is connected to the Director (over a Helper on USB, or over WiFi). |
| `links` | array | How it is connected right now. Empty when offline. Can hold both a `usb` and a `wifi` link. |
| `battery` | object or null | Only for tallies with a battery, while online. `minutes_left` is null while charging or not known yet. |
| `product` | string or null | `"TallyLED"`, `"TallyLCD"`, or null when unknown. |
| `fw` | string | Firmware version the tally last reported. |
| `connection` | string or null | Since 1.1. `"usb"` (through a TallyHelper) or `"wifi"` (straight to the Director). Null when offline. |
| `host` | string or null | Since 1.1. Name of the machine the tally is connected to: the machine running TallyHelper for `"usb"`, the Director machine for `"wifi"`. Null when offline. |
| `rssi` | integer or null | Since 1.1. WiFi signal in dBm while the tally has a WiFi link, else null. A tally on USB turns its WiFi off, so this is null on USB apart from the brief overlap described below. |

`connection`, `host` and `rssi` are a summary of `links`. A tally on a TallyHelper turns its own WiFi off, but both links can show for a second or two while it switches over. `connection` is then `"usb"`.

The tally's own screen shows the signal as bars: 4 from -55 dBm, 3 from -65, 2 from -75, 1 below that. Use the same steps for a bar display, so the two agree.

The list holds every tally the Director knows, online or not, sorted by device number (tallies without a number last). "Currently connected" means `online: true`.

A **state** is one of `"off"`, `"aux"`, `"preview"`, `"program"`. The tally shows the highest of `osc_state` and `switcher_state`, in the order program, preview, aux, off. Colours on the tally: program red, preview green, aux blue, off dark.

### `switcher`

Since 1.1. The video switcher TallyDirector follows, and its connection. Sent once after the first `tallies`, then on every change. Read only: the switcher is set up in TallyDirector's Settings.

```json
{
  "type": "switcher",
  "switcher": {
    "kind": "atem",
    "name": "Blackmagic ATEM / Constellation",
    "host": "10.48.10.30",
    "port": 9910,
    "status": "connected",
    "model": "ATEM Television Studio HD8",
    "firmware": "9.6",
    "error": null
  }
}
```

Without a switcher:

```json
{
  "type": "switcher",
  "switcher": {
    "kind": "none", "name": null, "host": null, "port": null, "status": "disabled",
    "model": null, "firmware": null, "error": null
  }
}
```

| Field | Type | Meaning |
|---|---|---|
| `kind` | string | `"none"`, `"live_premier"`, `"atem"`, `"vmix"`, `"panasonic_uhs500"` or `"panasonic_hs410"`. New types may be added. Show an unknown one by its `name`. |
| `name` | string or null | The type as TallyDirector's Settings name it, e.g. `"Analog Way LivePremier / Aquilon"`. Null for `"none"`. |
| `host` | string or null | IP address or host name of the switcher. Null when not set. |
| `port` | integer or null | Port in use: the one set in Settings, or the type's default. Null for `"none"`. |
| `status` | string | See below. |
| `model` | string or null | Model the switcher reports about itself, once connected. Not every type reports one. |
| `firmware` | string or null | Firmware version the switcher reports, once connected. Not every type reports one. |
| `error` | string or null | Why the last connection attempt failed, a sentence for display. It stays while TallyDirector retries (`status` going between `"error"` and `"connecting"`) and is cleared once the switcher connects, or when no switcher is set. |

`status`:

| Value | Meaning |
|---|---|
| `"disabled"` | No switcher set. |
| `"connecting"` | Trying to connect. TallyDirector retries every 2 s on its own. After a failure, `error` still holds the reason. |
| `"waiting_for_approval"` | Analog Way only: connected, waiting for someone to accept TallyDirector in the Web RCS. |
| `"connected"` | Working. Tally from the switcher is live. |
| `"error"` | The last attempt failed, see `error`. Still retrying. |

While the status is anything but `"connected"`, every tally's `switcher_state` is `"off"`.

### `result`

The answer to a command that carried an `id`. Commands without an `id` get no `result`, but errors are still logged in the Director.

```json
{"type": "result", "id": "7", "ok": true}
```

```json
{"type": "result", "id": "8", "ok": false, "error": "no tally has device number 9"}
```

* `id`: the command's `id`, unchanged. It is a string, chosen by the client.
* `error`: present when `ok` is false. A human readable sentence, meant for the log.

A message that is not valid JSON, or has an unknown `type`, gets `{"type": "result", "id": null, "ok": false, "error": "..."}`.

The `result` comes first. The `tallies` showing the change follows within about 50 ms. Use `tallies` for display, never `result`.

## Client to Director

All commands address tallies by device number, the same way OSC does. They change `osc_state` only. A switcher's state is never overridden: a tally the switcher puts in program stays red even after `set_state` to `"off"`.

### `set_state`

Sets one device, leaves all others alone. Same as OSC `/tally/<n>/state`.

```json
{"type": "set_state", "id": "1", "device": 3, "state": "program"}
```

Accepted for any device number 1 or higher, even when no tally has it yet. A tally given that number later shows the state.

### `set_exclusive`

Puts exactly one device in the given state. The device that was in that state before loses it. Same as OSC `/tally/program <n>`, `/tally/preview <n>` and `/tally/aux <n>`.

```json
{"type": "set_exclusive", "id": "2", "state": "preview", "device": 2}
```

* `state` is `"program"`, `"preview"` or `"aux"`.
* `device: 0` removes that state from every device.
* Other states a device has are kept. A device can be in program and preview at once, and then shows program.

### `clear`

Turns every device's `osc_state` off. Same as OSC `/tally/clear`. Switcher states stay.

```json
{"type": "clear", "id": "3"}
```

### `identify`

The tally blinks white for 3 s, so someone can find it. Fails when no online tally has that device number.

```json
{"type": "identify", "id": "4", "device": 3}
```

## Using the API from Bitfocus Companion

A suggested mapping for a Companion module. Other clients can skip this section.

* **Config:** host (Bonjour picker plus manual IP field) and port (default 9101).
* **Actions:** set state (device dropdown plus state), exclusive program, exclusive preview, exclusive aux, clear all, identify.
* **Feedbacks:**
  * tally state is X for device N (boolean, default style: red for program, green for preview, blue for aux)
  * tally online for device N (boolean)
  * since 1.1: tally connected by USB or WiFi for device N (boolean), and switcher connected (boolean)
  * a combined "tally colour" feedback that sets the button colour from `state`.
* **Variables:**
  * for each tally with a device number: `device_<n>_name`, `device_<n>_state`, `device_<n>_online`, and since 1.1 `device_<n>_connection` (USB, WiFi or empty), `device_<n>_host`, `device_<n>_rssi`
  * `tallies_online` (count) and `api_version` (e.g. `1.1`)
  * since 1.1, the switcher: `switcher_type` (`name`), `switcher_host`, `switcher_port`, `switcher_status`, `switcher_model`, `switcher_firmware`, `switcher_error`.
* **Dropdowns and presets:** build them from the `tallies` list. Label each choice `<device> <name>` (just `<device>` without a name). Rebuild only when the set of device numbers or names changes, not on every state change.
* **Connection status:** "ok" after `hello` with `api: 1`. "connection failure" while disconnected. "bad config" for another API version.

## Example session

```text
client connects to ws://10.48.10.4:9101/
<- {"type":"hello","api":1,"api_minor":1,"version":"0.4.1"}
<- {"type":"tallies","tallies":[ ...device 1 off, device 2 off... ]}
<- {"type":"switcher","switcher":{"kind":"atem", ... "status":"connected", ...}}
-> {"type":"set_exclusive","id":"a1","state":"program","device":2}
<- {"type":"result","id":"a1","ok":true}
<- {"type":"tallies","tallies":[ ...device 2 "state":"program"... ]}
   (a switcher puts device 1 in preview)
<- {"type":"tallies","tallies":[ ...device 1 "state":"preview","switcher_state":"preview"... ]}
```

## Trying it out

With TallyDirector running, this prints everything the API sends and puts device 1 in Program. It needs Node.js 22 or newer and nothing else. Save it as `try-api.mjs` and run `node try-api.mjs`.

```js
const socket = new WebSocket('ws://127.0.0.1:9101/');
socket.addEventListener('open', () => {
  socket.send(JSON.stringify({ type: 'set_exclusive', id: '1', state: 'program', device: 1 }));
});
socket.addEventListener('message', (event) => console.log(event.data));
```

`{"type": "clear"}` turns everything off again.
