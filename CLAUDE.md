# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cockatrice is an open-source, multiplatform application for playing tabletop card games over a network, built with C++20 and Qt. The project consists of three main components: **Cockatrice** (game client), **Servatrice** (server), and **Oracle** (card database tool).

## Build System

### Basic Build Commands

```bash
# Standard build
mkdir build && cd build
cmake ..
make

# Build with tests enabled
cmake .. -DTEST=1
make
make test

# Build server (disabled by default)
cmake .. -DWITH_SERVER=1

# Debug build with warnings as errors
cmake .. -DCMAKE_BUILD_TYPE=Debug

# Don't treat warnings as errors (Debug builds only)
cmake .. -DWARNING_AS_ERROR=0

# Update translation files
cmake .. -DUPDATE_TRANSLATIONS=1

# Install to release directory or create package
make install
make package
```

### CMake Build Options

- `-DWITH_SERVER=1` - Build Servatrice server (default: OFF)
- `-DWITH_CLIENT=0` - Skip building Cockatrice client (default: ON)
- `-DWITH_ORACLE=0` - Skip building Oracle (default: ON)
- `-DTEST=1` - Enable regression tests with GoogleTest (default: OFF)
- `-DCMAKE_BUILD_TYPE=Debug` - Debug build with extra logging and verbose warnings
- `-DWARNING_AS_ERROR=0` - Don't treat warnings as errors in debug mode (default: ON in debug)
- `-DUPDATE_TRANSLATIONS=1` - Update .ts translation files (default: OFF)
- `-DFORCE_USE_QT5=1` - Skip Qt6 detection and use Qt5

### Running Tests

```bash
# Run all tests
cd build
make test

# Run specific test
./tests/expression_test
./tests/carddatabase/carddatabase_test
```

Available tests: `dummy_test`, `expression_test`, `test_age_formatting`, `password_hash_test`, `deck_hash_performance_test`, `carddatabase_test`, `filter_string_test`, `loading_from_clipboard_test`, `parse_cipt_test`

## Code Architecture

### Component Structure

**Three executables:**
- **cockatrice/** - Qt GUI game client
- **servatrice/** - Server supporting TCP and WebSocket connections
- **oracle/** - Wizard-based card database management tool

**Shared libraries (libcockatrice_*):**
- **libcockatrice_protocol** - Protobuf message definitions (100+ .proto files), command/event wrappers
- **libcockatrice_network** - Client/server network layer, supports local (single-player) and remote (multiplayer) modes
- **libcockatrice_card** - Card database system with XML parsers (v3/v4), card/set/printing data structures
- **libcockatrice_deck_list** - Deck management with tree structure, memento pattern for undo, sideboard plans
- **libcockatrice_rng** - SFMT (SIMD-oriented Fast Mersenne Twister) random number generator
- **libcockatrice_utility** - Shared utilities (expression parsing, password hashing, Levenshtein distance)
- **libcockatrice_settings** - Settings management for database, downloads, servers, layouts
- **libcockatrice_models** - Qt model classes for database and deck list views
- **libcockatrice_filters** - Tree-based card filter expressions
- **libcockatrice_interfaces** - Abstract interfaces for dependency injection, allowing libraries to remain UI-agnostic

### Network Protocol Architecture

The system uses a **Command-Event pattern** with Protocol Buffers:

**Commands** (client → server): `SessionCommand`, `GameCommand`, `RoomCommand`, `ModeratorCommand`, `AdminCommand`

**Events** (server → clients): `EventContainer`, `GameEventContainer`, `RoomEvent`

**Key patterns:**
- `CommandContainer` wraps commands with `cmd_id` for request/response matching
- `PendingCommand` (Qt QObject) emits signals when responses arrive, enabling async command tracking
- All network events are dispatched via Qt signals from `AbstractClient`
- `Server_ProtocolHandler` maps incoming commands to handler methods on server side

**Client types:**
- `RemoteClient` - TCP/WebSocket with authentication, password hashing, ping/timeout handling
- `LocalClient` - In-process client for single-player mode (uses same game logic as multiplayer)

**Server architecture:**
- Server-authoritative model prevents client-side cheating
- Connection pooling distributes load across threads
- Supports both TCP (`QTcpSocket`) and WebSocket (`QWebSocket`) with same protobuf messages

### Game State Architecture

**Zone-based card management:**
Cards exist in zones (library, hand, graveyard, battlefield, exile, etc.). Server maintains canonical state in `Server_CardZone`, clients render via zone-specific logic (`pile_zone_logic`, `table_zone_logic`, `hand_zone_logic`).

**Replay system:**
Games are recorded as event sequences (`GameReplay`) that can be played back or uploaded. Uses same event stream as live games.

**Graphics:**
- Game board uses Qt Graphics Framework (`QGraphicsScene`/`QGraphicsView`)
- Custom `QGraphicsItem` subclasses for cards, arrows, counters

### Card Database System

**Pluggable parser architecture:**
- `ICardDatabaseParser` interface with implementations (`CockatriceXml3Parser`, `CockatriceXml4Parser`)
- `CardDatabaseLoader` coordinates parsing
- `CardDatabaseQuerier` provides search functionality
- `CardDatabaseManager` is singleton accessor

**Data model:**
- `CardInfo` - Card metadata (name, types, colors, text)
- `PrintingInfo` - Specific set printings
- `ExactCard` - Unique card+set combination
- `CardSet` - Set/expansion information

## Code Style and Formatting

### Formatting Tool

**Always run `./format.sh` before committing.** The CI will reject PRs that don't comply with formatting.

```bash
# Format all modified files
./format.sh

# Format specific file
clang-format -i path/to/file.cpp
```

The project uses `.clang-format` for C++/headers, `.cmake-format.json` for CMake files.

### Key Conventions

**Language:** C++20 standard throughout

**Qt conventions:**
- Use Qt data structures (`QString` over `std::string`, `QList` over `std::vector`)
- Use `nullptr` instead of `NULL` or `0`
- Use `static_cast<>` instead of C-style casts

**Naming:**
- `UpperCamelCase` for classes, structs, enums
- `lowerCamelCase` for functions and variables
- No Hungarian notation, no decorating member variables with underscores
- Constructor arguments with same name as members: prefix with underscore (`MyClass(int _myData) : myData(_myData)`)

**Pointers/References:**
- Prefer references and stack allocation over pointers and heap allocation
- When pointers needed, use smart pointers (`QScopedPointer`, `QSharedPointer`)
- `*` and `&` go with the variable name: `Foo *foo;` not `Foo* foo;`

**Headers:**
- Use `.h` for headers, `.cpp` for source
- Header guards: `FILE_NAME_H`
- Include order: project includes first (alphabetical), then library includes (alphabetical)
- Group project and library includes with blank line between

**Indentation:**
- 4 spaces, never tabs
- Braces on own line for functions/classes, same line for control statements
- Prefer braces around single-line statements

**Lines:**
- 120 characters or less
- No trailing whitespace

### Protocol Buffer Changes

Changes to `.proto` files in `common/pb/` must be handled with extreme caution - they affect client-server compatibility. Test intensively before merging. See [wiki on Client-server protocol](https://github.com/Cockatrice/Cockatrice/wiki/Client-server-protocol).

### Database Migrations

When modifying `servatrice/servatrice.sql`:
1. Increment `cockatrice_schema_version` in `servatrice.sql`
2. Increment `DATABASE_SCHEMA_VERSION` in `servatrice_database_interface.h`
3. Create migration file in `servatrice/migrations/` named after new version
4. Run `servatrice/check_schema_version.sh` to verify

## Translation Workflow

**For developers:**
- Wrap user-facing strings in `tr("text")` - they'll be picked up automatically
- For plurals: `tr("text", "hint", count)`
- Don't manually update `.ts` files - CI handles this via automated PRs

**Translation files:**
- `cockatrice/translations/` - Client translations
- `oracle/translations/` - Oracle translations
- Managed via Transifex integration

## Project Dependencies

**Required:**
- Qt (5 or 6) - GUI framework
- Protocol Buffers - Message serialization
- CMake - Build system

**Optional (Oracle):**
- xz - Compressed card file support
- zlib - Compressed card file support

## Testing Infrastructure

- **Framework:** GoogleTest (auto-downloaded if not found)
- **Location:** `/tests` directory
- **Test types:** Unit tests (utilities), integration tests (card database, clipboard), performance tests (deck hashing)
- **Timeout:** Performance tests have 5s timeout

## Build System Details

**Auto-MOC:** `CMAKE_AUTOMOC` enabled for all Qt libraries - Qt meta-object code generated automatically

**Platform-specific:**
- macOS: Builds .app bundles, uses DragNDrop installer (.dmg)
- Windows: Uses NSIS installer, includes vcredist
- Linux: Builds .deb or .rpm packages

**ccache:** Enabled by default via `USE_CCACHE=ON` to speed up rebuilds

## Working with the Codebase

**Qt signal/slot pattern:**
All network events, game state changes, and UI interactions use Qt signals/slots extensively. When adding features, follow this pattern for loose coupling.

**Logging:**
Use `QLoggingCategory` for module-based logging (e.g., `RemoteClientLog`, `LocalClientLog`).

**Thread safety:**
Client/server use different threads. Use `QMutex`/`QRecursiveMutex` (e.g., `Server_Game::gameMutex`). Connection pools distribute work across threads.

**Dependency injection:**
Use interfaces from `libcockatrice_interfaces` to keep libraries decoupled from UI code.

## Related Repositories

- [Magic-Token](https://github.com/Cockatrice/Magic-Token) - MtG token data
- [Magic-Spoiler](https://github.com/Cockatrice/Magic-Spoiler) - Spoiler generation from MTGJSON
- [cockatrice.github.io](https://github.com/Cockatrice/cockatrice.github.io) - Official website
