# Kyros User Guide

## Connecting

Open Settings and configure:

- **Server profile:** an existing saved automation profile, when applicable.
- **Server:** IRC hostname or address.
- **Port:** the port supplied by the network. TLS commonly uses 6697 and plaintext commonly uses 6667, but networks may differ.
- **Use TLS encryption:** enable this whenever the server supports it.
- **Nickname, username, and real name:** the identity sent during IRC registration.

Use the plug button to connect or disconnect. While a connection is pending, the action changes to **Cancel connection**. Errors such as an unknown hostname, refused connection, unreachable server, TLS failure, timeout, or lost connection appear in the server transcript.

When **Automatically reconnect** is enabled in Connection settings, an unexpected connection loss is retried after five seconds. Kyros makes up to five reconnect attempts, then stops and reports that it has given up. A manual disconnect or cancelled connection is never retried. After a successful reconnect, channel tabs that were still open are joined again automatically.

## Conversations

The sidebar contains the server window, joined channels, and direct messages. It can be collapsed with the small sidebar handle. Right-click supported transcript items and nicknames for context actions.

The main window remembers its size, position, maximized state, selected theme, and sidebar state.

## Commands

Commands begin with one slash. Lua aliases and macros begin with two slashes.

| Command | Purpose |
| --- | --- |
| `/connect` | Connect using the current settings. |
| `/disconnect` | Disconnect or cancel a pending connection. |
| `/join #channel` | Join one or more channels. |
| `/list` | Open and refresh the channel-list browser while connected. |
| `/part [message]` | Leave the current channel. |
| `/rejoin` | Leave and rejoin the current channel. |
| `/msg nickname message` | Open/send a private message. |
| `/me action` | Send an action in a channel. |
| `/whois nickname` | Request information about a user. |
| `/time nickname` | Request a user's client time. |
| `/version nickname` | Request a user's client version. |
| `/kick nickname [reason]` | Kick a user from the current channel when permitted. |
| `/raw [on\|off]` | Show, hide, or toggle the Raw protocol window. |
| `/clear` | Clear the current transcript. |
| `/quit [message]` | Quit IRC. |
| `/set variable_name value` | Save a shared scripting variable as text. |
| `/unset variable_name` | Remove a shared scripting variable. |

Unknown single-slash commands are sent to the IRC server. Never paste commands from an untrusted person, especially commands containing passwords or authentication tokens.

The Raw window appears directly below Server and above the channel list. Kyros remembers whether it is enabled between sessions. While enabled, it displays every IRC protocol line before normal parsing, using `→` for incoming traffic and `←` for outgoing traffic. Its `/log [on|off]` command writes daily logs named `raw.YYYY-MM-DD.log` in the `Log` folder.

### Input formatting shortcuts

Channel, private-message, and server input boxes accept IRC formatting controls directly: **Ctrl+B** inserts bold, **Ctrl+K** color, **Ctrl+I** or **Ctrl+G** italic, **Ctrl+R** reverse, **Ctrl+U** underline, and **Ctrl+O** reset. Enter color digits immediately after Ctrl+K when needed.

### Channel list

While connected, select **Channel list** on the toolbar or enter `/list` in any conversation. Kyros opens a themed browser showing each channel's name, modes, member count, and plain-text topic, initially sorted by largest membership and then by channel name. Select **Refresh** to request a fresh list. Long topics are shortened in the table; hover over one to see the full text. Double-click a channel you have not already joined to send a join request.

## Server automation

Automation profiles can match a server address, port, TLS setting, and optional network name. A matching profile may run commands and join channels after registration.

The protected placeholders are:

- `{nickserv_password}`
- `{oper_password}`
- `{nick}`

Passwords are protected for the current Windows user. They should still be treated as sensitive and should never be placed directly into scripts, screenshots, or bug reports.

## Lua scripts

Open the Lua Scripts dialog from the toolbar. New and bundled scripts remain unloaded until **Load** is selected. Saving a loaded script reloads it. If the new version has an error, Kyros reports the error and retains the previous working runtime.

Only load scripts you trust. Scripts are sandboxed, but they can send IRC commands and messages on your behalf.

See [Lua Scripting Guide](LUA_SCRIPTING.md) for the complete introduction.

## Troubleshooting

### Unable to connect

Confirm the hostname, port, TLS setting, internet connection, firewall, and system clock. A TLS error often means the TLS setting or port is wrong, the certificate is invalid, or the computer clock is incorrect.

### Connected but not registered

Review the server transcript for nickname conflicts, authentication requirements, bans, or registration errors.

### Script command is unknown

Open Lua Scripts and verify that the script defining the `//command` is loaded. Correct syntax or runtime failures are shown in the server transcript.

## Transcript logging

Use `/log on` or `/log off` in a server, channel, or private-message window. `/log` with no argument toggles logging for the current window. Server and channel choices are remembered; private-message choices reset when Kyros closes.

Logs are written beside Kyros in the `Log` folder, with one file per window per day. Every logged line receives an independent timestamp even when display timestamps are disabled.

