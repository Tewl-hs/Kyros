# Kyros IRC Client

Kyros is a Windows IRC client with configurable server profiles, connection automation, themes, and sandboxed Lua scripting.

## Development status

Kyros is under active, continuous development and receives frequent updates. Features, behavior, and configuration formats may continue to change as the application evolves.

## Windows SmartScreen

Kyros is not currently code-signed. Windows Defender SmartScreen, when enabled, may warn about or block the application. Only bypass this warning for a Kyros package downloaded from the [official GitHub releases](https://github.com/Tewl-hs/Kyros/releases/latest), and verify the published SHA-256 checksum when possible.

If Windows marks the extracted application as blocked:

1. Right-click `Kyros.exe` and select **Properties**.
2. Open the **General** tab.
3. Under **Security**, select **Unblock**.
4. Select **Apply**, then **OK**.

If the **Unblock** option is not present, Windows has not marked that file as blocked.

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

## System requirements

- Windows 10 or Windows 11 (64-bit).
- The [Microsoft .NET 10 Desktop Runtime for Windows x64](https://dotnet.microsoft.com/en-us/download/dotnet/10.0). The .NET SDK is not required.

User settings and the registered-script list are stored for the current Windows account. Portable scripts may be kept in the `Scripts` folder beside `Kyros.exe`, including subfolders, while **Load** can select a script from another location. The Lua Script Editor creates the portable Scripts folder when needed and uses it as the file picker's default location.

## License

Kyros is proprietary software licensed for personal, non-commercial use. Redistribution is not permitted without prior written permission from Tewl-hs. See [LICENSE.txt](LICENSE.txt) for the complete terms and [THIRD_PARTY_NOTICES.txt](THIRD_PARTY_NOTICES.txt) for bundled component licenses.

