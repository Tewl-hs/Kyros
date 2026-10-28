# Kyros 1.1.1 Release Notes

Released October 28, 2026.

These are all user-facing changes made after the Kyros 1.1.0 release.

### Added

- TLS certificate failures that require user action now stop automatic reconnect. A themed, fixed-size dialog shows every validation problem, expiration details, and the SHA-256 fingerprint before allowing an expired certificate for the current connection.
- Accepted expired certificates can be remembered for the current session or permanently for the exact server endpoint and fingerprint. Reconnects can use **Do not ask again for this server**, while changed certificates require review again.
- The Lua Script Editor has a themed **Network Access** manager for viewing and revoking session and persistent network approvals.
- Lua scripts can create asynchronous outbound TCP and TLS clients, local UDP sockets, and outbound WebSocket clients through `net.tcp()`, `net.tls()`, `net.udp()`, and `net.websocket()`.
- Socket APIs support UTF-8 text, validated one-based byte arrays, incremental stream decoding, local and remote endpoint inspection, lifecycle state, connection timeouts, optional WebSocket headers and subprotocols, and automatic cleanup on script unload or reload.
- UDP can bind a fixed port or request a Windows-assigned port with port `0`. Bind conflicts and other failures are delivered through the socket error event; datagrams preserve message boundaries and may contain up to 65,507 bytes.
- Lua network access is permission-controlled by script filename, SHA-256 content fingerprint, transport, host or bind address, and port. Scripts may receive a one-session approval or an approval that remains valid only while their contents are unchanged.
- Socket resource limits prevent unbounded work: 16 sockets per script, 1 MiB stream/WebSocket messages, 16 pending sends per socket, and 256 queued callbacks per script. A socket is closed if callback backpressure is exceeded.
- Lua now provides strict UTF-8 byte conversion, bounded HTTP/1.x response parsing with chunked-body support, bounded JSON encode/decode, and secure XML parse/encode helpers with DTDs and external resolution disabled.
- A weather example script demonstrates postal-code or place-name lookup, raw HTTPS requests, HTTP parsing, JSON handling, and formatted mIRC-colored output without requiring an API key.
- Lua scripts can register selectable chat providers with validated text, choice, boolean, and protected-secret profile fields. Providers may own a connection and implement logical send, action, join, part, open, close, topic, member-list, direct-message, raw-traffic, and status callbacks.
- Providers can disable local echo when a service returns the authoritative message, publish display-name colors, compact text badges, action messages, and message IDs for bounded deduplication, and update protected profile secrets after token rotation.
- Providers may declare a generic OAuth authorization-code flow. Kyros validates cryptographic state, receives the localhost callback, exchanges the code over HTTPS, and stores access and refresh tokens in protected fields.
- Provider protocol traffic can be published to the Raw window with directional markers while keeping authorization headers and secrets out of the transcript.

### Changed

- The Lua Script Editor is modeless and no longer forced above the main window, allowing scripts to remain open while chatting. Reopening the editor activates its existing instance.
- The Script Editor retains full edge-and-corner resizing with a standard bottom-right grip, restores safe on-screen placement, and uses normal themed button colors.
- Kyros keeps a most-recently-used order for the Server, Raw, channel, and private-message views. Closing a view removes it from the order and restores the view that was active immediately before it.
- Newly opened channels become active. Incoming private messages open in the background and blink for attention, while private messages opened manually take focus.
- Connection profiles now select either IRC or a registered provider and generate provider-specific profile fields dynamically. The selected profile is retained across restarts.
- The Connection Profile dialog uses distinct **Add Server** and **Update Server** states, prevents duplicate addresses, and shows **Clear Selection** only while editing an existing entry.
- Connection-profile validation appears left-aligned in the footer opposite the action buttons. Boolean profile fields use labeled checkboxes, profile selection refreshes after saving, and the dialog height and spacing have been tightened.
- Older settings and imported profiles are normalized to the provider-aware schema version while retaining IRC servers, automation settings, selected profiles, and protected secrets.
- TCP and TLS Lua sockets remain strictly outbound clients; only UDP may bind locally.
- Provider raw traffic is queued asynchronously, capped at 100 pending display entries, and truncated at 32 KiB per entry so diagnostics cannot block socket callbacks or flood the UI thread.
- The portable package includes user-facing examples and documentation but omits the socket stress script and provider development/example scripts.
- Update checks now use the public `Tewl-hs/Kyros` GitHub Releases feed. The update dialog opens the verified official release page instead of offering a direct download.
- The Script Editor documentation button now opens the public Lua Scripting Guide in the Kyros release repository.
- The application version is now 1.1.1.

### Fixed

- Certificate warning dialogs no longer allow connection attempts to continue in the background while the user is deciding, and read-only dialog values no longer use invalid two-way WPF bindings.
- Round-robin server hostnames are handled by the certificate presented by the endpoint actually reached; Kyros does not hardcode or silently substitute a known server address.
- Double-clicking transcript text no longer performs transient selection or copies a word to the clipboard. It is now available as an assignable action gesture with the clicked word, line, character, and token.
- Channel nicklists enforce single selection, preventing Ctrl-click or Shift-click from leaving multiple users selected for nickname menus.
- Transcript selection hit-testing now uses a wrapped-line-aware binary search instead of calculating geometry for every character on every mouse movement, preventing freezes when selecting long Raw-window JSON lines.
- Provider network callbacks no longer synchronously wait for Raw-window rendering while the user changes views.
- Provider joins and parts no longer create duplicate channel windows, and `/part` closes the intended provider conversation whether issued from that room or the server window.
- Provider-created conversations consistently become active except for unsolicited private messages. Closing them restores the correct preceding view.
- Provider member lists include the authenticated identity and observed participants, and displayed sender identity stays consistent with the member list.
- Provider `/me` and `/action` commands use the service's action callback, and incoming CTCP `ACTION` payloads render as actions instead of ordinary text.
- Provider messages returned by the service can replace local echo without duplication, retaining service-supplied colors, badges, and message IDs.
- Raw-window provider traffic now appears alongside IRC raw traffic, and its bounded asynchronous rendering remains responsive during selection and view changes.
- Network permission prompts are themed, use standard button colors, do not duplicate an already approved exact request, and session approvals appear in the Network Access manager.
- Lua network errors identify the script and operation responsible across all loaded scripts.
- The fixed-port UDP example waits for its asynchronous bind to complete before sending to itself.

---

# Kyros 1.1.0 Release Notes

Released October 26, 2026.

These are all user-facing changes made after the Kyros 1.0.3 release.

### Added

- A direct-drawing, virtualized transcript control is now built directly into the Kyros executable for server, channel, and private-message windows; no separate reusable control library is shipped.
- mIRC-style transient selection automatically copies plain text when the mouse button is released and clears the highlight. Pressing Ctrl+C while the mouse is still held copies the selection with its applicable mIRC formatting controls.
- Transcript selection now supports wrapped and multiline ranges, drag auto-scrolling, Shift-click extension, double-click word selection, triple-click line selection, and Ctrl+A.
- Formatted clipboard output supports bold, italic, underline, strikeout, reverse video, numeric foreground/background colors, and hexadecimal foreground/background colors.
- The transcript exposes a read-only UI Automation TextPattern with document, selection, visible-range, point lookup, text search, navigation, selection, scrolling, and visible glyph-boundary support for accessibility clients.
- Named server profiles can contain multiple server addresses. Profiles can be created, renamed, updated, removed, and selected independently from the connection port and TLS setting.
- Existing server profiles are migrated to address lists. Koach.com profiles are populated with `irc.koach.com` and `chat1` through `chat4.koach.com`.
- A cancelable Lua `server_text` event handles server-originated `PRIVMSG` traffic sent to either a channel or nickname. It supplies `e.server`/`e.source`, `e.target`, and `e.message`.
- Server-originated messages receive a default `![server]: message` presentation, with a matching accent-styled presentation in `xvaliant-theme.lua`.
- The Lua `numeric` event now supplies `e.automatic_who` so scripts can distinguish replies generated by Kyros membership maintenance.
- A Lua `error` event supplies `e.error_type`, `e.message`, `e.script`, `e.operation`, and `e.exception_type` for uncaught load, event, command, menu, dialog, and audio errors.
- Lua error-event recursion protection reports a failing `error` handler without firing another `error` event. A failed reload keeps the previous working script active and able to receive the load error.
- A `--self-check` startup mode validates formatted clipboard serialization, single-digit IRC colors, formatted hyperlinks, and server-originated message parsing.
- The repository includes a standalone transcript stress prototype for high-volume rendering, selection, clipboard, scrolling, palette, and UI Automation testing; its production renderer sources are linked directly into Kyros.
- Invalid `settings.json` files are preserved with timestamped `.corrupt` filenames and produce a visible recovery warning.
- Successful settings saves retain the previous readable configuration as `settings.backup.json`; Kyros restores it when the active settings file becomes malformed.

### Changed

- Transcript layout is created only for visible lines instead of retaining a WPF element tree for every message, reducing the cost of large retained histories and conversation switching.
- Each conversation retains its own font, size, scroll position, tail-following state, and visible anchor. These remain stable across window switches, retention trimming, palette changes, font changes, and wrapped-line recalculation.
- IRC palette changes invalidate visible transcript layouts immediately, and transcript scrollbars continue to use the active Kyros theme.
- URL, channel, and nickname tokens retain activation and themed context menus in the direct renderer. Nicknames are actionable only while they are present in the supplied nicklist.
- Connection profiles and their server-address lists are sorted, always keep an address selected, and require at least one registered address. At least one profile is retained.
- The connection port remains a normal editable field, allowing the same profile address to be used with whichever plaintext or TLS port the server requires.
- The Automation tab has been reorganized into a roomier two-column layout for startup commands and protected values, with server identity editing moved into the compact profile dialog.
- The Lua Script Editor is resizable from every edge and corner, remembers its size and position, rejects off-screen saved positions, and retains its borderless themed appearance.
- Script Editor placement is restored during native-window initialization to reduce main-window blinking when the editor opens.
- Combo-box drop-downs size to their contents up to a maximum height instead of leaving unused blank space beneath short lists.
- `xvaliant-theme.lua` now formats numeric replies through the dedicated `numeric` event instead of parsing numerics from `server_line`.
- The Lua scripting guide and event reference document `server_text`, `numeric`, `automatic_who`, and structured error-event handling.
- Settings that cannot be read because of permissions, file locking, or disk errors are left untouched; defaults are used and further settings writes are blocked for that session.
- The application version is now 1.1.0.

### Fixed

- Update notifications no longer terminate Kyros while opening the update dialog; installed version, latest version, and release notes bindings are explicitly read-only.
- JSON release-notes responses are now unwrapped from their `changelog` or `releaseNotes` field instead of displaying the raw JSON envelope and escaped line breaks.
- The update dialog now uses a standard themed **Download** button with enforced horizontal text padding instead of an orange **Download update** button.
- The update dialog is now a fixed-size window titled **Update Available**, and Markdown heading markers are rendered as clean bold headings in its release-notes document.
- Manually disconnecting or pressing the connection button during an automatic reconnect now cancels the reconnect timer, resets the attempt count, and leaves Kyros disconnected.
- The connection button and icon now show **Cancel reconnect** while a reconnect is pending and **Cancel connection** while a connection attempt is active.
- Plain selection-copy no longer includes mIRC controls, while Ctrl+C during the active mouse selection retains them.
- Formatted clipboard copies no longer convert ordinary theme brushes into hexadecimal IRC color controls or emit a redundant leading reset.
- Single-digit IRC colors such as `4` are parsed alongside two-digit colors such as `04`.
- URLs interrupted by mIRC formatting controls are detected from their complete visible text, remain one logical destination, and do not include formatting bytes in the opened address.
- Token detection no longer changes a transcript token's foreground, background, underline, bold, italic, strikeout, or reverse-video appearance.
- Clipboard contention is handled without crashing the transcript UI.
- Selection and scroll anchors are clamped safely when old retained lines are removed.
- Server-prefixed messages are routed to the intended channel or private-message conversation rather than being treated as ordinary user messages or displayed in the wrong window.
- Server-profile selectors no longer show port or TLS descriptions in the server-address list.
- The profile dialog uses themed controls, keeps its address list compact, and refuses to save a profile without a name and at least one address.

---

# Kyros 1.0.3 Release Notes

Released October 23, 2026.

### Added

- A unified Theme Editor for application colors, typography, and each theme's associated 32-color IRC palette.
- Editable built-in themes with a single reset action that restores both application styling and the linked IRC palette.
- Dedicated theme colors for channel, server, private-message, nicklist, and message-input surfaces.
- A network-scoped internal IRC user directory populated from channel membership, presence events, NAMES, and WHO data.
- Read-only Lua IRC-state snapshots through `irc.channels()` and `irc.users()`, including channel membership, known addresses, negotiated rank modes, and address-completeness state.
- A first-class cancelable Lua `numeric` event for every numeric IRC reply, with the three-character reply available as `e.numeric`.
- An optional Raw protocol window showing all incoming and outgoing IRC lines with directional markers. `/raw [on|off]` controls the window and its enabled state is remembered between sessions.
- Independent Raw-window transcript logging through `/log [on|off]`, written to daily `raw.YYYY-MM-DD.log` files.

### Changed

- Application preferences now use a stable `%LocalAppData%\Kyros\settings.json` file instead of .NET's versioned `Kyros_Url_*` user-settings directories; the newest existing Kyros configuration is imported automatically.
- Theme overrides and custom palettes are stored outside version-specific application folders so they remain available after upgrades.
- The Application Colors editor uses a spacious category-and-details layout without scrollbars or reduced typography.
- IRC parsing and channel detection now use structured protocol messages and server-advertised `PREFIX`, `CHANTYPES`, and `STATUSMSG` capabilities.
- Socket connections support both IPv4 and IPv6 endpoints.
- Lua state is updated before presence-event handlers read it, and scripts receive detached tables that cannot modify client state.

### Fixed

- Update checks now understand the live release API's versioned envelope and `changelogUrl` fields instead of silently treating every valid response as “no update.”
- IPv4 connections made through the dual-stack socket are displayed as normal IPv4 addresses instead of IPv4-mapped IPv6 values beginning with `::ffff:`.
- Transcript words no longer inherit WPF's default blue hyperlink color; ordinary text and interactive tokens now respect the active theme's foreground color.
- Nickname menu handling no longer strips a hardcoded set of status prefixes that may not match the connected server.
- Unknown IRC commands are passed through for script and server-line handling instead of being treated as parse failures.
- Theme Editor color controls remain visible without relying on scrollbars.

---

# Kyros 1.0.2 Release Notes

Released October 22, 2026.

### Added

- Per-window daily transcript logging with `/log`, `/log on`, and `/log off`. Server and channel choices persist between sessions, while private-message choices last for the current session.
- Log files are written per window and per day in the `Log` folder beside Kyros, with an independent timestamp on every recorded line.
- Connection and disconnection events are recorded in every enabled log for an open server, channel, or private-message window.
- Selective Bold, Italic, Color, and Reverse stripping for displayed messages, with an optional configurable code-count threshold, plus an independent option to strip all formatting codes from logs.
- Lua macros can define multiple commands as an ordered table of strings.
- The Lua script editor includes a Documentation button linking to the online scripting reference.
- Channel, private-message, and server input boxes insert IRC formatting controls with Ctrl+B, Ctrl+K, Ctrl+I/Ctrl+G, Ctrl+R, Ctrl+U, and Ctrl+O.
- The application theme editor supports separate channel, server, private-message, and nicklist fonts and sizes up to 22 points.

### Changed

- Theme selection and application-theme editing now live on the right side of the taller main toolbar.
- The former Appearance tab is now **Messages / Logs**, with a compact two-column layout for timestamp, formatting, and logging preferences.
- Channel log filenames no longer include the `Ch.` prefix.
- The channel nicklist is narrower and its complete right border remains visible.
- Newly created application themes become active immediately.
- Saving an edited custom application theme also renames it, including case-only name changes.
- Deleting the active custom application theme applies and selects the built-in Dark theme.
- The conversation sidebar defaults to expanded on a new installation while continuing to remember the user's saved choice.
- Loaded scripts are listed first in the Lua script editor.
- Lua `format.reverse` emits the standard `\x12` reverse control character.
- The xValiant topic metadata label now reads **Set by**.

### Fixed

- Reverse formatting recognizes `\x12` correctly while retaining `\x16` as a compatibility alias.
- Plain-text topic and title cleanup removes the complete supported set of IRC formatting controls.
- Nickname changes no longer produce a duplicate raw `NICK` line in the server transcript after the formatted event is handled.
- Deleting an active custom theme no longer leaves the main-window theme selector blank.
- Legacy startup-command, authentication, auto-join, and perform-profile values are cleared after successful migration into automation profiles.
