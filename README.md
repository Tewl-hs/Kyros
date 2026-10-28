# Kyros IRC Client

Kyros is a Windows IRC client with configurable server profiles, connection automation, themes, and sandboxed Lua scripting.

## Tester quick start

1. Extract the complete release folder. Do not run the executable from inside the ZIP.
2. Start `Kyros.exe`.
3. Open Settings and review the Connection and Identity tabs.
4. Enter the IRC server, port, TLS setting, nickname, username, and real name.
5. Close Settings and use the plug button to connect.

The status area reports connection failures. A pending connection can be cancelled with the same toolbar button. Kyros times out an unreachable connection after 15 seconds.

See [User Guide](docs/USER_GUIDE.md) for normal client use and [Lua Scripting Guide](docs/LUA_SCRIPTING.md) for scripts, aliases, variables, and command functions.

## Tester feedback

When reporting a problem, include:

- What you were trying to do
- The exact error displayed by Kyros
- Whether TLS was enabled
- Whether the issue happens after restarting
- Steps that reproduce the problem

Do not include passwords, authentication commands, or private chat content in a report.

## Requirements

Kyros runs on Windows. Framework-dependent builds require the .NET 10 Desktop Runtime. A self-contained release includes the runtime and does not require a separate .NET installation.

User settings are stored for the current Windows account. Script files are stored in the `Scripts` folder beside `Kyros.exe`; use **Open Folder** in the Lua Script Editor to open the active location.

## License

Kyros is proprietary software licensed for personal, non-commercial use. Redistribution is not permitted without prior written permission from Tewl-hs. See [LICENSE.txt](LICENSE.txt) for the complete terms and [THIRD_PARTY_NOTICES.txt](THIRD_PARTY_NOTICES.txt) for bundled component licenses.

