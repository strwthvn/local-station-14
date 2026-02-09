# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **Space Station 14** (SS14), a multiplayer space station simulation game remake of SS13. This repository is a Russian localization fork ("local-station-14") that:
- Provides full Russian translation
- Maintains sync with the upstream SS14 repository
- Adds custom content via "Corvax" interfaces

The game runs on the **RobustToolbox** engine (C# game engine in `RobustToolbox/` submodule).

## Build Commands

```bash
# First-time setup: initialize submodules and download engine
python RUN_THIS.py

# Build solution
dotnet build

# Build with optimization (used by CI)
dotnet build --configuration DebugOpt

# Run server
dotnet run --project Content.Server

# Run client
dotnet run --project Content.Client

# Run unit tests
dotnet test Content.Tests/Content.Tests.csproj

# Run integration tests
dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj

# Run a single test
dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj --filter "FullyQualifiedName~TestClassName"

# Validate YAML prototypes (linter)
dotnet run --project Content.YAMLLinter/Content.YAMLLinter.csproj
```

## Architecture

### Project Structure

| Project | Purpose |
|---------|---------|
| `Content.Server` | Server-side game logic |
| `Content.Client` | Client-side rendering, UI, input |
| `Content.Shared` | Code shared between client and server (networked components, shared systems) |
| `Content.IntegrationTests` | Integration tests with client/server pairs |
| `Content.Tests` | Unit tests |
| `Content.YAMLLinter` | Validates YAML prototype definitions |
| `Corvax/` | Custom interface extensions for this fork |
| `RobustToolbox/` | Game engine submodule |

### Entity Component System (ECS)

SS14 uses an ECS architecture:

- **Components** (`*Component.cs`): Data containers with `[RegisterComponent]` attribute
  - Defined in `Content.Shared` for networked state, or side-specific projects
  - Use `[DataField]` for YAML-serializable fields
  - Use `[AutoNetworkedField]` for auto-synced networked state

- **Systems** (`*System.cs`): Logic processors that operate on entities with specific components
  - Inherit from `EntitySystem`
  - Subscribe to events in `Initialize()` via `SubscribeLocalEvent<TComponent, TEvent>`
  - Server-only systems in `Content.Server`, client-only in `Content.Client`
  - Shared systems in `Content.Shared` with side-specific overrides

- **Prototypes** (YAML in `Resources/Prototypes/`): Data-driven entity templates
  - Entity prototypes define which components an entity has
  - Use `parent:` for prototype inheritance
  - Validated by `Content.YAMLLinter`

Example pattern:
```csharp
// Content.Shared/Access/Components/AccessComponent.cs
[RegisterComponent, NetworkedComponent]
[AutoGenerateComponentState]
public sealed partial class AccessComponent : Component
{
    [DataField, AutoNetworkedField]
    public HashSet<ProtoId<AccessLevelPrototype>> Tags = new();
}

// Content.Shared/Access/Systems/SharedAccessSystem.cs
public abstract class SharedAccessSystem : EntitySystem
{
    public override void Initialize()
    {
        SubscribeLocalEvent<AccessComponent, MapInitEvent>(OnAccessInit);
    }
}

// Content.Server/Access/Systems/AccessSystem.cs
public sealed class AccessSystem : SharedAccessSystem { }
```

### Resources Directory

| Path | Content |
|------|---------|
| `Resources/Prototypes/` | YAML entity/prototype definitions |
| `Resources/Textures/` | Sprites and RSI (Robust Sprite Image) files |
| `Resources/Audio/` | Sound files |
| `Resources/Locale/` | Localization files (en-US, ru-RU, nl-NL) |
| `Resources/Maps/` | Map files |

### Integration Tests

Tests use a pooled client/server pair system:
```csharp
await using var pair = await PoolManager.GetServerClient();
var server = pair.Server;
// Use server.ResolveDependency<T>() to get systems/managers
// Use server.WaitPost(() => { ... }) to run code on server thread
```

## Code Style

- C# 14, .NET 10
- File-scoped namespaces preferred
- Use `var` for type inference
- Nullable enabled by default
- EditorConfig in repository root defines formatting rules
