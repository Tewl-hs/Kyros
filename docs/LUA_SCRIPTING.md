# Kyros Lua Scripting Guide

Kyros uses sandboxed Lua scripts. Scripts can react to IRC events, print or echo text, send IRC commands, add `//commands`, store simple values, add context-menu actions, and create dialogs.

## Managing scripts

Open **Lua Scripts** from the toolbar. Scripts live under `%LocalAppData%\Kyros\Scripts`.

- **New** creates an unloaded script.
- **Save** saves the selected file and reloads it if it is loaded.
- **Load** enables a script.
- **Unload** disables it without deleting the file.
- **Reload** reloads all enabled scripts.

Bundled examples are installed unloaded. Start with `getting-started.lua`.

## Output and sending

```lua
print("Shown in the server transcript")
echo("server", "Also shown in the server transcript")
echo("active", "Shown in the active window")
send("JOIN #example")
```

`send()` sends raw IRC protocol text without a trailing line break. Scripts can act as the user, so do not load untrusted scripts.

## Network sockets

Sockets are asynchronous and are automatically closed when their script is unloaded or reloaded. Each script may own up to 16 sockets, individual sends are limited to 1 MiB, each socket may have at most 16 pending sends, and a script may have at most 256 pending network callbacks. A socket is closed with a visible error if its script cannot process incoming events quickly enough. Socket text is encoded and incrementally decoded as UTF-8, so a character split between network reads remains intact.

Before the first exact outbound connection, UDP send, or local bind, Kyros shows a themed permission dialog identifying the script, transport, host/address, and port. **Allow for session** lasts until Kyros exits. **Always allow** is tied to the script filename and SHA-256 content fingerprint, so editing or replacing the script invalidates the approval. Denial is reported through the socket's `error` event. Use **Network Access** in the Lua Script Editor to revoke remembered permissions or clear session approvals.

TCP and TLS sockets are strictly outbound clients. Lua scripts cannot listen for or accept inbound TCP/TLS connections and cannot select their local TCP/TLS bind address or port. Only UDP sockets may bind locally, subject to an exact bind permission. UDP datagrams may contain at most 65,507 bytes.

```lua
local udp = net.udp()
udp:on("datagram", function(data, remote)
    print(remote.address .. ":" .. remote.port .. " " .. data)
end)
udp:bind("127.0.0.1", 19771)
udp:send_to("127.0.0.1", 19770, "hello")
```

TCP and TLS sockets use the same event style:

```lua
local client = net.tls()
client:on("connected", function()
    client:send("GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")
end)
client:on("data", function(data) print(data) end)
client:on("closed", function(reason) print(reason) end)
client:on("error", function(message) print(message) end)
client:connect("example.com", 443)
```

`connect(host, port [, timeout_ms])` uses a 30-second timeout by default. A custom timeout may be from 1,000 through 120,000 milliseconds and covers both TCP connection and TLS authentication. Timeouts are delivered through the `error` event. After connection, `local_endpoint()` and `remote_endpoint()` return `{ address, port }` tables; before connection they return `nil`. `state()` reports `created`, `authorizing`, `connecting`, `connected`, `closed`, or `failed`. A stream socket accepts only one connection attempt; create a new socket after it closes or fails.

Binary protocols use one-based Lua byte arrays. Every value must be a whole number from `0` through `255`, with no missing indexes:

```lua
local client = net.tcp()
client:on("connected", function()
    client:send_bytes({ 0x4B, 0x79, 0x72, 0x6F, 0x73 })
end)
client:on("bytes", function(bytes)
    print("Received " .. #bytes .. " raw bytes")
end)
```

The `bytes` event represents one network read, not necessarily one complete application message. Scripts must buffer and frame messages according to their protocol. UDP preserves datagram boundaries: `send_to_bytes(host, port, bytes)` sends a byte array and `datagram_bytes` receives the byte array followed by the remote `address`/`port` table.

Pass port `0` to `udp:bind(address, 0)` to let the operating system select an available local port. Binding is asynchronous when permission is required; use the `bound` event before reading `udp:local_endpoint()` or sending from the socket. `local_endpoint()` then returns its `address` and assigned `port`. For fixed ports, binding is the authoritative availability check; conflicts are delivered to the socket's `error` handler because a separate preflight check would have a race with other processes.

A datagram sent to the same socket's own loopback address and port inherits its approved bind permission because it cannot leave the local machine. Every other UDP destination requires a separate exact send permission.

```lua
udp:on("bound", function()
    local endpoint = udp:local_endpoint()
    print("UDP is listening on " .. endpoint.address .. ":" .. endpoint.port)
end)
udp:bind("127.0.0.1", 0)
```

Available constructors are `net.tcp()`, `net.tls()`, `net.udp()`, and `net.websocket()`. Stream sockets provide `on`, `connect`, `send`, `send_bytes`, `local_endpoint`, `remote_endpoint`, `state`, `close`, and `is_open`, with `connected`, `data`, `bytes`, `closed`, and `error` events. UDP sockets provide `on`, `bind`, `local_endpoint`, `send_to`, `send_to_bytes`, `close`, and `is_open`, with `bound`, `datagram`, `datagram_bytes`, and `error` events. Text and byte handlers may both be registered when a script needs both representations.

WebSockets are outbound clients and use the same exact per-script endpoint permissions, callback queue, 1 MiB message limit, 16-pending-send limit, and unload cleanup as other sockets:

```lua
local socket = net.websocket()
socket:on("connected", function(subprotocol)
    socket:send(json.encode({ type = "hello" }))
end)
socket:on("message", function(text)
    local message = json.decode(text)
    print(message.type)
end)
socket:on("binary", function(bytes)
    print("Received " .. #bytes .. " bytes")
end)
socket:on("closed", function(code, reason)
    print("Closed: " .. tostring(code) .. " " .. reason)
end)
socket:on("error", function(message) print(message) end)
socket:connect("wss://example.com/events", {
    timeout_ms = 30000,
    headers = { ["X-Client"] = "Kyros" },
    subprotocols = { "example.v1" }
})
```

WebSocket handles expose `on`, `connect`, `send`, `send_bytes`, `state`, `close`, and `is_open`. Events are `connected`, `message`, `binary`, `closed`, and `error`. URLs must use `ws://` or `wss://`; embedded credentials and fragments are rejected.

For protocols layered over raw sockets, `encoding.utf8_encode(text)` produces a one-based byte array and `encoding.utf8_decode(bytes)` strictly decodes one. `http.parse_response(bytes)` parses a complete bounded HTTP/1.x response collected from a socket `bytes` event and returns `{ status, headers, body }`; it supports content-length and byte-correct chunked bodies but does not open a connection itself.

Use `net.defer(function)` to schedule work for the next application-dispatcher turn. This is useful when one socket callback needs to finish before creating or operating another socket.

## JSON and XML

`json.decode(text)` converts objects and arrays into Lua tables; `json.encode(value)` performs the reverse conversion. Use `json.null` when an explicit JSON null must be preserved. Documents and generated output are limited to 1 MiB with a maximum nesting depth of 64.

`xml.parse(text)` returns a node table with `name`, `namespace`, `text`, `attributes`, and ordered `children`. `xml.encode(node)` serializes that representation. DTD processing, external entities, filesystem resolution, and network resolution are disabled. XML uses the same document-size and nesting limits as JSON.

## Context values

The following globals are refreshed before callbacks:

```lua
client.nickname
client.active_target

server.ip
server.port
server.name
server.network
```

`server.network` may be `nil` until the network advertises its name.

## IRC state

`irc.channels()` returns a fresh, read-only snapshot of the channels you are
currently on. Lua arrays are indexed from `1`:

```lua
for _, channel in ipairs(irc.channels()) do
    print(channel.name .. " has " .. channel.member_count .. " members")

    for _, member in ipairs(channel.members) do
        print(member.name, member.address or "address unknown")
    end
end
```

Each channel provides `name`, `topic`, `modes`, `member_count`,
`addresses_complete`, and `members`. Each member provides `name`, `address`,
`prefix`, `modes`, and `away`. `address` is `nil` until the server supplies it.
The `modes` array contains the server-negotiated member mode letters rather
than assuming specific IRC prefixes.

`irc.users()` returns each known user once and adds a `channels` array containing
the channels you share with that user:

```lua
for _, user in ipairs(irc.users()) do
    print(user.name .. " is visible in " .. #user.channels .. " channels")
end
```

Both functions return detached snapshots. Changing a returned table does not
change Kyros's channel list or internal member state.

## Events

Register an event handler with `on()`:

```lua
on("connect", function()
    print("Connected to " .. server.name)
end)

on("text", function(e)
    print(e.nick .. " said " .. e.message .. " in " .. e.channel)
end)
```

The `text` event runs for both received and locally sent messages. `e.self` is
`true` for your own outgoing message and `false` for an incoming message.
`e.private_message` identifies direct-message windows. Outgoing messages have
already been sent when the display callback runs, so `e.prevent_default()` only
replaces their default local rendering:

```lua
on("text", function(e)
    e.prevent_default()

    local nick_color = e.self and 10 or 11
    echo(e.channel,
        format.color(nick_color) .. e.nick ..
        format.reset .. ": " .. e.message)
end)
```

The `action` event provides the same `e.self` and `e.private_message` fields for
`/me` actions.

The cancelable `server_text` event runs when a server or service sends
`PRIVMSG` directly rather than through a user prefix. It provides `e.server`
and `e.source` (the server name), `e.target` (the destination channel or
nickname), and `e.message`. An unavailable channel falls back to the server
window. Its default display is `![server]: message`.

| Event | Description |
| --- | --- |
| `error` | Reports an uncaught Lua error at a Kyros script boundary. This notification cannot be canceled. |
| `connect` | Runs after the IRC connection is established. It has no meaningful event argument. |
| `disconnect` | Runs when the IRC connection closes. It has no meaningful event argument. |
| `raw` | Observes every incoming raw IRC line before specialized routing. Its callback receives the line directly as a string. |
| `numeric` | Runs for every numeric reply, including replies consumed by features such as LIST and MOTD. Canceling it suppresses normal display handling only. |
| `server_line` | Runs for protocol lines considered for the server/status transcript. Lines consumed by an earlier dedicated feature may not reach it. |
| `ping` | Runs when an incoming `PING` line is considered for display. Canceling it does not affect protocol handling. |
| `pong` | Runs when an incoming `PONG` line is considered for display. Canceling it does not affect protocol handling. |
| `text` | Runs for incoming and locally sent channel or direct messages. |
| `server_text` | Runs when a server or service sends a `PRIVMSG` to a channel or nickname. |
| `action` | Runs for incoming and locally sent `/me` actions, with the same direction fields as `text`. |
| `join` | Runs when any user, including the local user, joins a channel. |
| `part` | Runs when a user leaves a channel with `PART`. |
| `quit` | Runs when a user disconnects from IRC, once for each open channel containing that user. |
| `kick` | Runs when a user is removed from a channel. |
| `nick` | Runs when a nickname changes, once for each applicable channel; local-user changes also have a server form. |
| `topic` | Runs when Kyros first displays the current topic after opening or joining a channel. |
| `topic_change` | Runs when a live `TOPIC` command changes a channel topic. |
| `topic_set` | Runs for numeric 333, which identifies who set the current topic and when. |
| `mode` | Runs when channel modes change. |
| `user_mode` | Runs when the local user's server-level modes change. |
| `notice` | Runs for a user-originated `NOTICE`; CTCP replies use `ctcp_reply` instead. |
| `server_notice` | Runs for a server- or service-originated `NOTICE`. |
| `ctcp_request` | Runs for received CTCP requests and requests sent by Kyros. Canceling it affects transcript display only. |
| `ctcp_reply` | Runs for received CTCP replies and automatic replies sent by Kyros. |

Except for `error`, `connect`, `disconnect`, and `raw`, these events are cancelable with `e.prevent_default()`. Cancellation suppresses Kyros's normal transcript presentation; it does not undo network activity or internal state updates.

The `numeric` event runs for every numeric IRC reply, including replies handled by dedicated client features such as LIST and MOTD. `e.numeric` is the three-character numeric as text, `e.line` and `e.data` contain the complete raw protocol line, and `e.automatic_who` identifies WHO replies generated by Kyros background membership maintenance:

```lua
on("numeric", function(e)
    if e.numeric == "332" then
        print("Topic reply: " .. e.line)
    end
end)
```

Calling `e.prevent_default()` suppresses the client's normal display handling for that numeric without preventing its internal state updates.

Many display events are cancellable:

```lua
on("text", function(e)
    if e.message == "hide this" then
        e.prevent_default()
    end
end)
```

Event fields depend on the event. Typical fields include `nick`, `address`, `channel`, `target`, `message`, `topic`, `kicked`, `numeric`, and `raw`.

Events that include a raw `line` or `raw` value also provide mIRC-style token
selection through `e:get("{3-}")`. Both `{3}` and `{3-}` selectors are supported.

## Custom `//commands`

Use `alias.add()` for Lua logic:

```lua
alias.add("hello", function(e)
    local name = e.tokens[1] or "everyone"
    echo(e.window, "Hello, " .. name)
end)
```

Run it with:

```text
//hello friend
```

The callback receives:

- `e.command` — alias name
- `e.args` — the complete argument text
- `e.tokens` — space-separated arguments, indexed from 1
- `e.window` — active channel or `server`
- `e.context` — current window context

Use `macro.add()` for simple command expansion:

```lua
macro.add("j", "/join {1}")
macro.add("wave", "/me waves to {1}")

macro.add("announce", {
    "/say {args}",
    "/echo Announced in {window}"
})
```

Macro placeholders are `{1}` through `{9}`, ranges such as `{3-}`, `{args}`, and `{window}`.
Range placeholders expand from that argument through the final argument.
Pass a table of command strings to run multiple commands in order. A single string
remains supported for one command and for compatibility with existing scripts.

## Variables from the input box

Client variables are shared by all scripts and survive application restarts. Set them from any server, channel, or private-message input box:

```text
/set greeting Hello everyone
/set value1 15.75
/unset greeting
```

Use the name without `${}` in `/set`; `${name}` is how the value is inserted into a `//command`. Values entered with `/set` are stored as text, which preserves exactly what was typed.

Lua scripts can use the same shared variables:

```lua
variables.set("greeting", "Hello everyone")
local greeting = variables.get("greeting", "Hello")
local exists = variables.has("greeting")
variables.remove("greeting")
```

## Persistent script storage

Storage is isolated by script and survives reloads and application restarts.

```lua
storage.set("greeting", "Hello")
storage.set("count", 1)
storage.set("enabled", true)

local greeting = storage.get("greeting", "Hi")
local exists = storage.has("greeting")
storage.remove("greeting")
```

Stored values may be strings, numbers, or booleans. `storage.get()` returns Lua `nil` when a value does not exist and no default is supplied.

Persistent storage limits are applied per script:

- 1,024 stored variables
- 128 characters per variable name
- 4,150 UTF-8 bytes per string value (4,150 ASCII characters; fewer for multi-byte Unicode text)
- 1 MiB total storage

Stored numbers must be finite. Lua numbers use double-precision floating point; whole numbers are exact only through `2^53`. Use the exact-number helpers below for decimal values or larger integers.

In a `//command`, identify a variable with `${name}`:

```text
//hello ${greeting}
```

A missing variable resolves to `nil`. Inside a command function, it is passed as an actual Lua `nil`.

Shared client variables take precedence over a script's private stored value when both use the same name.

## Hash tables

Lua calls hash tables **tables**. No extra library or knowledge of another language is required. A table can use text keys, numeric keys, or both:

```lua
local users = {}

users["alice"] = { joins = 1, greeted = true }
users["bob"] = { joins = 3, greeted = false }

users["alice"].joins = users["alice"].joins + 1
print(users["alice"].joins)
```

Test for an entry and remove it with `nil`:

```lua
if users["alice"] ~= nil then
    print("Alice is known")
end

users["bob"] = nil
```

Loop over every key and value with `pairs()`:

```lua
for nick, details in pairs(users) do
    print(nick .. " has joined " .. details.joins .. " times")
end
```

Tables declared outside a callback remain available while the script is loaded. Reloading the script clears them. Save important individual values with `storage` or `variables` if they must survive a reload.

## Regular expressions

Lua includes simple text-pattern functions such as `string.match()`. Kyros also supplies `regex` for more advanced matching:

```lua
regex.is_match("Nick42", "^[A-Za-z]+[0-9]+$")       -- true

local match = regex.match("price: 19.95", "([0-9]+[.][0-9]+)")
if match ~= nil then
    print(match.value)       -- 19.95
    print(match.groups[1])   -- 19.95
end

local words = regex.matches("one two three", "[A-Za-z]+")
local clean = regex.replace("too    many spaces", "[ ]+", " ")
local parts = regex.split("red,green,blue", ",")
```

Pass `true` as the last argument for case-insensitive matching:

```lua
regex.is_match("HELLO", "hello", true)
regex.replace("Hello HELLO", "hello", "hi", true)
```

Useful pattern pieces include `^` for the start, `$` for the end, `.` for any character, `[0-9]` for a digit, `[A-Za-z]` for a letter, `+` for one or more, `*` for zero or more, `?` for optional text, and parentheses for a captured group. Prefix a special character with `\` when it should be matched literally.

`regex.match()` returns one match or `nil`. A match contains `value`, a one-based `index`, `length`, and `groups`. `regex.matches()` returns every match. Patterns are limited to 1,024 characters, input is limited to 65,536 UTF-8 bytes, and slow matches are stopped automatically.

## Exact numbers

Lua's normal `number` type is suitable for counters, ports, timers, and most calculations. Kyros also provides exact helpers for money-like decimals, signed 64-bit integers, and integers of any size. Pass operands as strings and store the returned strings so Lua never converts them through floating point.

```lua
local price = decimal.add("19.99", "5.25")       -- "25.24"
local rounded = decimal.round("12.345", 2)       -- "12.34"
local next_id = int64.add("9007199254740993", "1")
local huge = bigint.multiply("999999999999999999", "1000")
```

Each library provides `add`, `subtract`, `multiply`, `divide`, and `compare`. `decimal` also provides `round(value, places)`, with zero through 28 decimal places. Arithmetic results are strings; `compare` returns `-1`, `0`, or `1`. Invalid input, division by zero, and `int64` or `decimal` overflow produce Lua errors.

## Command functions

Register functions that may be evaluated inside `//command` arguments:

```lua
command_function.add("add", function(a, b)
    return a + b
end)
```

Identify a command function with `@`:

```text
//echo ${value1} + ${value2} = @add(${value1}, ${value2})
```

Variables resolve first, followed by command functions. Only functions registered with `command_function.add()` can run. Function arguments may be numbers, booleans, `nil`, quoted strings, or unquoted strings. Functions may return a string, number, boolean, or `nil`. Unknown functions, Lua errors, invalid argument syntax, unsupported return values, and excessive nesting stop the command and report an error.

`echo`, `msg`, `say`, and `action` are reserved client commands and cannot be registered with `alias.add()` or `macro.add()`. Their `//` forms expand variables and command functions before invoking the corresponding built-in command. If a referenced variable is missing or a command function returns `nil`, the command stops without displaying or sending anything.

## Formatting

```lua
local message = format.bold .. "Bold" .. format.bold
message = message .. " " .. format.color(4) .. "colored" .. format.reset
echo("active", message)
```

Available values include `format.bold`, `format.italic`, `format.underline`, `format.strike`, `format.reverse`, `format.reset`, and `format.color(foreground [, background])`. `format.reverse` emits the `\x12` reverse control; Kyros also accepts `\x16` when received for compatibility.

## Context menus

The simple form adds a normal menu item:

```lua
context_menu.add("nicklist", "Say hello", function(e)
    send("PRIVMSG " .. e.nick .. " :Hello!")
end)
```

Contexts are `nicklist`, `channel`, `private_message`, and `server`. Callback fields are normalized; fields that do not apply are empty strings.

Add a divider with `context_menu.separator()`. Give items a `parent` to place them in a submenu; `/` creates deeper nested submenus:

```lua
context_menu.separator("nicklist")

context_menu.add("nicklist", "Give voice", function(e)
    send("MODE " .. e.channel .. " +v " .. e.nick)
end, { parent = "Moderation" })

context_menu.separator("nicklist", "Moderation")

context_menu.add("nicklist", "Remember user", function(e)
    storage.set("last_user", e.nick)
end, { parent = "Tools/History" })
```

The label and the `parent`, `visible`, `enabled`, and `checked` settings may be either fixed values or functions. Functions are evaluated every time the menu opens and receive the same context table as the click action. A dynamic `parent` can move an item between submenus based on the current context:

```lua
context_menu.add("nicklist", function(e)
    return "Message " .. e.nick
end, function(e)
    send("PRIVMSG " .. e.nick .. " :Hello!")
end, {
    visible = function(e)
        return e.nick ~= client.nickname
    end,
    enabled = function(e)
        return variables.get("allow_greetings", "yes") == "yes"
    end
})
```

Use `checked` for a checked or unchecked menu item. Change the underlying variable in the action; the checkmark is recalculated the next time the menu opens:

```lua
context_menu.add("channel", "Notifications", function(e)
    local enabled = variables.get("channel_notifications", "yes") == "yes"
    variables.set("channel_notifications", enabled and "no" or "yes")
end, {
    checked = function(e)
        return variables.get("channel_notifications", "yes") == "yes"
    end
})
```

This example only demonstrates persistent checked state. It does not alter message handling.

Nicklist menus also provide information about the selected user's channel rank and your own:

- `e.rank_prefix` and `e.my_rank_prefix` — strongest prefixes, such as `@` or `+`.
- `e.modes` and `e.my_modes` — active advertised user modes, such as `ov`.
- `e.rank` and `e.my_rank` — numeric hierarchy positions; a smaller number is higher.
- `e.is_higher`, `e.is_lower`, and `e.is_same_rank` — selected user's hierarchy relative to your own.

This example shows an action only for users below you, disables it unless you are an operator, changes its label from the selected user's modes, and places it in a submenu:

```lua
context_menu.add("nicklist", function(e)
    if string.find(e.modes, "v", 1, true) then
        return "Remove voice from " .. e.nick
    end
    return "Give voice to " .. e.nick
end, function(e)
    local removing = string.find(e.modes, "v", 1, true) ~= nil
    send("MODE " .. e.channel .. " " .. (removing and "-v" or "+v") .. " " .. e.nick)
end, {
    parent = "Moderation",
    visible = function(e)
        return e.is_lower
    end,
    enabled = function(e)
        return string.find(e.my_modes, "o", 1, true) ~= nil
    end,
    checked = function(e)
        return string.find(e.modes, "v", 1, true) ~= nil
    end
})
```

## Dialogs

Open a modeless dialog by providing its settings, controls, and one event function:

```lua
local window

window = dialog.open({
    title = "Greeting",
    width = 460,
    height = 320,
    resizable = true,
    minimizable = false,
    maximizable = false,
    controls = {
        { type = "label", text = "Enter a nickname:" },
        { type = "textbox", id = "nickname", text = "" },
        { type = "status", id = "status", text = "Ready" },
        {
            type = "row",
            controls = {
                { type = "button", id = "send", text = "Send" },
                { type = "button", id = "cancel", text = "Cancel" }
            }
        }
    }
}, function(e)
    if e.event == "click" and e.control == "send" then
        local nick = dialog.get_text(e.dialog, "nickname")
        if nick == "" then
            dialog.set_text(e.dialog, "status", "Enter a nickname first")
            return
        end
        send("PRIVMSG " .. nick .. " :Hello!")
        dialog.close(e.dialog)
    elseif e.event == "click" and e.control == "cancel" then
        dialog.close(e.dialog)
    elseif e.event == "closed" then
        window = nil
    end
end)
```

Available controls:

- `label` — wrapped primary text. Supports `text` and optional `id`.
- `status` — wrapped secondary text. Supports `text` and optional `id`.
- `textbox` — editable single-line text. Supports `id` and `text`.
- `button` — clickable button. Supports `id` and `text`; raises `click`.
- `list` — selectable list. Supports `id`, `items`, and `height`; raises `selection_changed` and raises `activate` on double-click.
- `row` — places nested `controls` horizontally with buttons aligned to the right.

Every control accepts `margin_bottom`. Controls that need to be read or changed should have a unique `id`.

Dialog functions:

```lua
dialog.get_text(window, "nickname")
dialog.set_text(window, "status", "Working...")
dialog.set_items(window, "results", { "one", "two", "three" })

local selected = dialog.selection(window, "results")
print(selected.index) -- one-based; 0 means no selection
print(selected.text)

dialog.close(window)
```

The event table contains `e.dialog`, `e.control`, and `e.event`. Closing the window raises `closed` with an empty control name. Dialogs do not stop the rest of the client while open.

## Token helpers

```lua
token.count("one two three")       -- 3
token.get("one two three", 2)      -- "two"
token.get("one two three", "2-") -- "two three"
token.expand("from {2-}", "one two three") -- "from two three"
token.find("one two three", "two") -- 2
token.split("one two three")       -- table indexed from 1
```

`token.expand()` supports `{n}` and `{n-}` placeholders in Lua strings. Bare
`{3-}` is not valid Lua syntax, so it must appear inside a quoted template.

## Chat providers

`provider.register()` adds a selectable connection provider and returns the handle that provider uses to report state and incoming messages to Kyros. Only the provider selected in the active connection profile may publish through its handle.

Providers use immediate local echo by default. Set `local_echo = false` in the provider definition when the service sends the authoritative copy back with metadata or may reject a send; the provider must then publish the sender's own message through `handle:message`.

```lua
local my_provider
my_provider = provider.register({
    id = "example-chat",
    name = "Example Chat",
    fields = {
        { key = "account", label = "Account", type = "text", required = true },
        { key = "token", label = "Token", type = "secret", required = true }
    },
    connect = function(profile)
        -- Start sockets or API requests, then report the actual result.
        my_provider:status("Signing in as " .. profile.account .. "…")
        my_provider:connected(profile.account)
    end,
    disconnect = function()
        -- Close provider-owned sockets and timers.
    end,
    send = function(target, message)
        -- Send a logical chat message through the service.
    end,
    join = function(target) end, -- optional
    part = function(target) end  -- optional
})
```

Profile field types are `text`, `secret`, `boolean`, and `choice`. Choice fields require a `choices` array. Secrets are protected by Kyros at rest and are supplied to `connect(profile)` only while connecting.

Provider handle functions:

- `handle:connected(display_name)` marks the connection ready for user input. The optional display name labels the user's own messages.
- `handle:disconnected(reason)` marks it closed; `reason` is optional.
- `handle:status(text)` writes provider status in the server transcript.
- `handle:raw(text, outgoing)` publishes sanitized provider protocol traffic to the Raw window. Providers must not include access tokens, authorization headers, secrets, or other credentials. `outgoing` defaults to `false`.
- `handle:message(conversation, sender, text, direct)` publishes an incoming message. Set `direct` to `true` for a private-message window. Existing scripts may continue using this form.
- `handle:message(conversation, sender, text, options)` may instead receive an options table with `direct`, `color` (`#RRGGBB`), `badges` (an array of short text labels), `action`, and `message_id`. Kyros renders colors and badge labels as transcript text, formats actions like `/me`, and suppresses duplicate non-empty message IDs.
- `handle:open(conversation, direct)` opens a conversation. New channel conversations take focus; direct-message conversations open in the background.
- `handle:close(conversation, direct)` closes a provider-owned conversation without sending another part request.
- `handle:topic(conversation, text)` updates a channel conversation's topic and title.
- `handle:users(conversation, users)` replaces the member list. Entries may be names or `{ name = "...", address = "..." }` tables.
- `handle:update_secret(field, value)` replaces a declared secret in the active connection profile using protected storage. This is intended for rotated OAuth access and refresh tokens.

Normal conversation text and `/msg` call `send`. `/join` and `/part` call their optional callbacks. A provider should call `open` after a successful join and `close` after a successful part. IRC-only commands are rejected instead of being sent to a provider. Unloading or reloading the active provider script disconnects its active session.

Providers may define `action(target, text)` to support `/me` and `/action`. If omitted, Kyros reports that the selected provider does not support actions.

Providers may add a generic authorization-code flow with an `oauth` definition. Referenced client-secret and token fields must be declared as `secret`; the client ID must be a `text` field. Kyros adds an authorization button to the generated profile form and protects the resulting tokens when the profile is saved.

```lua
oauth = {
    authorization_url = "https://service.example/oauth/authorize",
    token_url = "https://service.example/oauth/token",
    redirect_uri = "http://localhost:8080/",
    scopes = { "chat:read", "chat:write" },
    client_id_field = "client_id",
    client_secret_field = "client_secret",
    access_token_field = "access_token",
    refresh_token_field = "refresh_token"
}
```

Authorization and token URLs must use HTTPS. Redirects must use an explicit loopback HTTP port. Kyros verifies a random `state`, limits and validates the callback request, times out after three minutes, and never writes token values to the transcript.

## Error handling

Kyros catches errors at script boundaries. Load, event, alias, menu, dialog, storage, regex, and command-function failures are reported in the server transcript with a Lua prefix. One failing event callback does not close the client, and a failing `//command` is stopped rather than partially executed. Most scripts do not need additional error handling.

Scripts can observe uncaught Lua errors with the `error` event:

```lua
on("error", function(e)
    print(e.error_type .. " in " .. e.script .. ": " .. e.message)
end)
```

The event provides:

- `e.error_type` — `load`, `event`, `command`, `menu`, `dialog`, or `audio`.
- `e.message` — the readable error message.
- `e.script` — the Lua filename responsible for the error, or `Kyros` when no script owns it.
- `e.operation` — the event, command, menu context, or callback being executed.
- `e.exception_type` — the underlying MoonSharp or .NET exception class.

A script that fails during its initial load cannot receive its own error because its handlers were never registered. Other loaded scripts can receive it. When a reload fails, the previous working copy remains active and can receive the load error. Errors raised by an `error` handler are written to the server transcript without recursively raising more `error` events.

Check optional values before using them:

```lua
command_function.add("safe_add", function(a, b)
    if a == nil or b == nil then
        return "missing value"
    end
    return a + b
end)
```

Use `assert()` when a required value is missing:

```lua
local channel = client.active_target
assert(channel ~= "", "This command needs an active channel")
```

Use `pcall()` only when the script can recover and continue. It returns `true` plus the result on success, or `false` plus the error message on failure:

```lua
local ok, result = pcall(function()
    return decimal.divide("10", "0")
end)

if not ok then
    print("Could not divide: " .. result)
end
```

Common mistakes are concatenating `nil`, using a control id that does not exist, passing a number where an exact-number helper requires text, and assuming optional server or event fields are always present. Check optional values with `if value ~= nil then`, or supply a default with `storage.get("name", "default")`.

