# Kyros IRC client architecture

Kyros is evolving from a single-server IRCX client into a general-purpose IRC
client. The existing WPF interface remains usable while the protocol and
application layers are replaced incrementally.

## Design boundaries

### Protocol

`IRC.Net.Protocol` owns wire-level IRC concepts only: message tags, prefixes,
commands, parameters, formatting, case mapping, ISUPPORT tokens, and channel
mode metadata. It must not reference WPF, a particular network, or scripting.

### Transport and session

A transport owns TCP, TLS, UTF-8 line framing, cancellation, and serialized
writes. An IRC session owns registration and state: CAP negotiation, optional
SASL, nick and user registration, reconnect policy, channels, users, queries,
and server features learned from `005`.

Connections will be configured by a `NetworkProfile`; no server address,
credential, nickname, or port belongs in source code. Multiple profiles and
multiple simultaneous sessions must be supported.

### Extensions

Network-specific behavior is implemented as a protocol extension. The existing
IRCX commands (`AUTHUSER`, `AUTHTYPE`, `CLIENTMODE`, `LOGINH`, `PROP`,
`WHISPER`, and `DATA`) will move into an IRCX extension. Ordinary IRC sessions
do not load it.

### Presentation

View models consume session events and immutable message models. They do not
parse raw IRC lines or write directly to a socket. WPF dispatcher changes are
kept at the presentation boundary.

The workspace uses three levels rather than MDI windows:

- a narrow network rail for simultaneous server connections
- a conversation navigator for the selected network, with separate collapsible
  channel and direct-message sections, filtering, unread counts, and favorites
- one active conversation surface, with an optional member list

Channels and private messages share a `Conversation` abstraction so selection,
unread state, history, logging, drafts, notifications, and scripting behave the
same way. Closing or hiding a conversation does not part a channel unless the
user explicitly chooses to leave it.

### Appearance and themes

Application themes are resource dictionaries made from semantic tokens such as
window, sidebar, conversation, input, primary text, secondary text, accent, and
selection colors. Views consume semantic resources rather than literal colors.
Options provides theme selection and a separate per-theme typography override
for message input, channel, server, private-message, and nick-list fonts and
sizes. Resetting typography removes that override without changing theme colors.

Theme changes are applied at runtime and persisted independently from network
profiles and Lua scripts. The global 32-color IRC formatting palette is edited
through its own dialog and remains independent of application themes. Named
palette files are stored in `%LocalAppData%\ColorPalettes` and can be loaded
from the dialog or the `/palette load` client command.

### Lua scripting

Lua runs through an `IScriptEngine` abstraction. The initial implementation
will use Lua 5.4 through NLua, but scripts receive a deliberately small Kyros
API rather than unrestricted CLR access.

The first public API should include:

- `irc.on(event, handler)` and removable subscriptions
- `irc.command(name, handler)` for slash commands
- `irc.send_raw(line)` with IRC framing and length validation
- `irc.message(target, text)`, `irc.notice(target, text)`, and `irc.action(...)`
- read-only network, channel, and user snapshots
- timers scheduled by the host
- per-script storage and logging
- `irc.format(event, handler)` for safe message-display transformations

Incoming events should include `connected`, `disconnected`, `raw`, `message`,
`notice`, `action`, `join`, `part`, `quit`, `nick`, `mode`, `topic`, `numeric`,
and `ircx`. Script errors are isolated to the script that caused them and are
reported in a script console.

Scripts are user-trusted automation, but the default environment will not
expose arbitrary .NET types, process creation, or unrestricted filesystem and
network access. Those capabilities can be added later as explicit permissions.

Lua format handlers operate on a presentation model, never WPF controls. A
handler can replace text or return styled spans with foreground/background
roles or colors, bold, italic, underline, strikethrough, and an optional font.
It can also attach a safe action or suppress a line. The host validates colors,
fonts, lengths, and links, then combines the result with the active application
theme. Original message data remains unchanged for logs and later scripts.

The render pipeline is:

`IRC message -> normalized event -> Lua format handlers -> validated spans -> active theme -> WPF document`

Format handlers run in deterministic priority order with time and output limits.
One failing script is disabled or skipped without preventing the message from
appearing.

## Migration sequence

1. Parse all incoming lines into the shared IRC/IRCv3 message model.
2. Replace the callback socket with an async, cancellable transport and correct
   TLS certificate validation.
3. Add configurable network profiles and standard IRC registration, including
   CAP and SASL.
4. Move connection/channel/query state out of WPF view models.
5. Extract the existing IRCX behavior into an optional extension.
6. Add the sandboxed Lua host and stable scripting API.
7. Add multi-network UI, query windows, persistence, logging, notifications,
   and reconnect behavior.

Every stage should keep the application buildable and should gain protocol unit
tests before the old regular-expression handlers are removed.
