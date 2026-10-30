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

Channel windows always use a supported channel prefix. When a provider supplies a room name without one, Kyros adds `#` for display and command routing while retaining the provider's original identifier internally. This lets a provider channel such as `#penguintat` coexist with a direct-message window named `penguintat`; `/msg #penguintat text` addresses the channel and `/msg penguintat text` addresses the direct message.

The main window remembers its size, position, maximized state, selected theme, and sidebar state.

### Appearance

Open **Options → Appearance** to select the application theme. Each theme keeps its own font family and size choices for message input, channel transcripts, the Server transcript, private-message transcripts, and the nick list. Choose **Save fonts** to apply those choices to the selected theme. **Reset fonts** removes only that theme's typography override and restores its original font settings without changing its colors or the independent IRC palette.

The IRC color palette is global and independent of the application theme. Open it with the palette button beside Lua Scripts to customize the 32 colors used by IRC formatting. Switching application themes does not change these colors. **Default** is Kyros's built-in palette and cannot be changed, removed, or reused as a saved name. Choose **Save as**, enter a name in the themed prompt, and choose **Save** to store and select the current colors in `%LocalAppData%\ColorPalettes`. Names may contain letters and numbers plus at most three single spaces. Leading and trailing spaces are removed automatically, while consecutive internal spaces are not allowed. Changing the dropdown selection previews that palette in the dialog without changing open transcripts. **Save** updates the selected named palette, **Remove** deletes it, and **Apply** activates whichever palette is selected. Save and Remove are disabled for Default.

You can also change a palette entry from any input box with `/color <0-31> <hexvalue>`. The index must be a whole number from 0 through 31 and the value must contain exactly six hexadecimal digits; a leading `#` is optional. For example, `/color 0 FFFFFF` sets IRC color 0 to white. The global change is saved and existing transcript colors update immediately.

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
| `/lua load scriptname` or `/lua l scriptname` | Load a Lua script. Relative names resolve inside the portable `Scripts` folder; scripts elsewhere require a full path. |
| `/lua unload scriptname` or `/lua u scriptname` | Unload a loaded Lua script without deleting it. |
| `/lua reload scriptname` or `/lua r scriptname` | Reload a loaded Lua script. A provider script cannot be reloaded while its provider is connected or connecting. |
| `/color 0-31 hexvalue` | Set and save one color in the global IRC palette, for example `/color 0 FFFFFF`. |
| `/palette load name` | Load and apply a named palette from `%LocalAppData%\ColorPalettes`, or use `Default` for Kyros's built-in colors. Put names containing spaces in quotes. |

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

Open the Lua Scripts dialog from the toolbar. Kyros loads only explicitly selected scripts. Use **Load** to select a script from the portable `Scripts` folder, a subfolder, or another location; the picker starts in the portable Scripts folder and creates it when needed. Only one registered script may use a given filename, regardless of its folder, which keeps script selection and permissions unambiguous. The list displays the filename; hover over it to see its complete path. **Unload** stops a script, closes its sockets, revokes its network approvals, removes its registry entry, and removes it from the visible list without deleting its file. Select it with **Load** to register and load it again. Saving a loaded script reloads it. If the new version has an error, Kyros reports the error and retains the previous working runtime. Profiles belonging to an unloaded provider are retained and marked unavailable. Loading a script that registers the same provider ID and name makes those profiles usable again.

Script load failures and other scripting diagnostics detected during startup are written to the Server window once it is ready. Each load error identifies the responsible script and operation; full exception details remain available in the Lua error log.

Kyros treats `scripts.json` as untrusted startup input. It limits the file to 1 MiB, limits JSON nesting and the number of registrations, validates schema and paths, ignores invalid entries, and handles inaccessible files and folders without stopping application startup. A malformed or unsupported registry is preserved with a timestamped `scripts.corrupt-*.json` name when possible, then Kyros continues with a clean registry.

The same lifecycle operations are available from any input box. Use `/lua load script.lua` or `/lua l script.lua` for a file in the portable `Scripts` folder. A subfolder may be included, such as `/lua l Utilities\helper.lua`. A script outside that folder requires its full path; surround a path containing spaces with double quotes. Use `/lua unload` (or `/lua u`) and `/lua reload` (or `/lua r`) with the registered script name. Kyros refuses to reload a provider script while its provider is connected or connecting; disconnect it first. Unloading remains available and performs the provider's normal disconnect cleanup.

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

