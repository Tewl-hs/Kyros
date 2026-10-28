# Kyros IRC Client

Kyros is a Windows IRC client with configurable server profiles, connection automation, themes, and sandboxed Lua scripting.

This public repository contains official Kyros releases and user documentation. The application source code is maintained separately and is not published here.

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

## Download

Download the current version from [Latest Release](https://github.com/Tewl-hs/Kyros/releases/latest). Extract the complete portable ZIP before starting `Kyros.exe`.

Kyros requires Windows and the .NET 10 Desktop Runtime.

## Documentation

- [User Guide](docs/USER_GUIDE.md)
- [Lua Scripting Guide](docs/LUA_SCRIPTING.md)
- [Architecture](docs/ARCHITECTURE.md)
- [Release Notes](RELEASE_NOTES.md)

## License

Kyros is proprietary software licensed for personal, non-commercial use. Redistribution is not permitted without prior written permission from Tewl-hs. See [LICENSE.txt](LICENSE.txt) for the complete terms and [THIRD_PARTY_NOTICES.txt](THIRD_PARTY_NOTICES.txt) for bundled component licenses.

Copyright © 2026 Tewl-hs. All rights reserved.
