# CLAUDE.md - AI Assistant Guide for StarfallEx

This document provides comprehensive guidance for AI assistants working on the StarfallEx codebase.

## Project Overview

**StarfallEx** is a Garry's Mod addon that provides a sandboxed Lua scripting environment for programming in-game contraptions. It allows players to write Lua code that controls entities, renders graphics, plays audio, and interacts with the game world, all within a secure, resource-limited execution environment.

- **Type**: Garry's Mod Addon (tool)
- **Language**: Lua
- **Primary Use**: In-game scripting for contraptions
- **Workshop ID**: 3412004213
- **Documentation**: http://thegrb93.github.io/StarfallEx/
- **Discord**: https://discord.gg/yFBU8PU
- **License**: GPL (see License.txt)

## Repository Structure

```
StarfallEx/
├── lua/                          # Main codebase (137 Lua files)
│   ├── autorun/                  # Auto-initialization scripts
│   │   ├── client/sf_init.lua    # Client-side startup
│   │   ├── server/sf_init.lua    # Server-side startup
│   │   └── netstream.lua         # Shared networking utility
│   ├── entities/                 # Starfall entity definitions
│   │   ├── starfall_processor/   # Main programmable chip
│   │   ├── starfall_screen/      # Display screen entity
│   │   ├── starfall_hologram/    # Holographic projections
│   │   ├── starfall_hud/         # HUD overlay entity
│   │   └── starfall_prop/        # Programmable prop entity
│   ├── starfall/                 # Core framework (26,000+ lines)
│   │   ├── sflib.lua             # Main library loader & module system
│   │   ├── instance.lua          # Execution context & quota management
│   │   ├── preprocessor.lua      # Compile-time directive processor
│   │   ├── transfer.lua          # Code transfer system
│   │   ├── libs_sh/              # Shared libraries (20+ libraries)
│   │   ├── libs_cl/              # Client-only libraries (14 libraries)
│   │   ├── libs_sv/              # Server-only libraries (8 libraries)
│   │   ├── permissions/          # Permission system
│   │   ├── editor/               # In-game code editor
│   │   └── examples/             # Example scripts
│   └── weapons/                  # Tool gun integration
│       └── gmod_tool/stools/     # Starfall tool modes
├── models/                       # 3D models for entities
├── materials/                    # Textures and materials
├── resource/                     # Additional resources (fonts)
├── .github/workflows/            # CI/CD automation
│   ├── doc_build.yml             # Documentation validation
│   ├── doc_deploy.yml            # Deploy docs to GitHub Pages
│   └── workshop.yml              # Deploy to Steam Workshop
├── gmcl_*.dll                    # Binary modules (joystick, socket, xinput)
├── addon.json                    # Garry's Mod addon metadata
├── Readme.md                     # User-facing documentation
└── License.txt                   # GPL license

```

## Architecture & Key Concepts

### Client/Server/Shared Realms

StarfallEx follows Garry's Mod's **realm-based architecture**:

1. **Server Realm** (`SERVER`)
   - Physics simulation
   - Entity management
   - Player data
   - Libraries in `libs_sv/`

2. **Client Realm** (`CLIENT`)
   - Rendering
   - UI/HUD
   - Audio playback
   - Input handling
   - Libraries in `libs_cl/`

3. **Shared Realm** (`SERVER` and `CLIENT`)
   - Common functionality
   - Libraries in `libs_sh/`

### File Naming Convention

- **`init.lua`** - Server-side only code
- **`cl_init.lua`** - Client-side only code
- **`shared.lua`** - Shared between client and server

### Preprocessor Directives

User scripts can use preprocessor directives for configuration:

```lua
--@name My Script
--@author John Doe
--@shared          -- Run on both realms (default)
--@server          -- Server-only execution
--@client          -- Client-only execution
--@include path/to/file.lua  -- Include other files
--@model models/props_c17/oildrum001.mdl
--@superuser       -- Enhanced permissions (admin only)
--@owneronly       -- Restrict to chip owner
```

### Module System

Libraries are registered and loaded via `sflib.lua`:

```lua
-- Library registration pattern
SF.RegisterLibrary("libraryname")

return function(instance)
    local library_table = instance.env

    -- Cache commonly used functions for performance
    local checkluatype = SF.CheckLuaType
    local haspermission = SF.Permissions.hasAccess

    -- Wrap/unwrap functions for type safety
    local ewrap = instance.Types.Entity.Wrap
    local eunwrap = instance.Types.Entity.Unwrap

    --- Function description for documentation
    -- @param Entity ent The entity to process
    -- @return boolean success Whether operation succeeded
    library_table.functionName = function(ent)
        checkluatype(ent, TYPE_ENTITY)
        local unwrapped = eunwrap(ent)
        -- Implementation
        return true
    end

    return library_table
end
```

### Security & Permissions

StarfallEx has a sophisticated permission system (`permissions/core.lua`):

1. **Privilege Registration**: Functions register required permissions
2. **Provider-Based**: Multiple permission providers (usergroups, client settings, etc.)
3. **Runtime Checks**: `SF.Permissions.hasAccess()` before privileged operations
4. **Override Support**: Superuser mode can override restrictions
5. **Persistent Settings**: Saved to `sf_perms_sv.txt` / `sf_perms_cl.txt`

Example permission registration:
```lua
local registerprivilege = SF.Permissions.registerPrivilege

registerprivilege("entities.setPos", "Set position", "Allows the user to teleport entities", {
    usergroups = { default = 1 }
})
```

### Resource Management

StarfallEx enforces strict quotas to prevent abuse:

1. **CPU Quota** (`instance.lua`)
   - Configurable time limit per execution
   - Soft-lock protection
   - Quota checking at hook boundaries

2. **RAM Limit** (`instance.lua`)
   - Default: 1.5MB per instance
   - Checked via `collectgarbage("count")`

3. **Burst Limiting**
   - Rate-limited operations (print, HTTP requests, etc.)
   - Prevents spam and abuse

### Type System

All game objects are wrapped/unwrapped for safety:

```lua
-- Wrapping: Lua object → SF object
local wrapped_entity = instance.WrapObject(gmod_entity)

-- Unwrapping: SF object → Lua object
local gmod_entity = instance.UnwrapObject(wrapped_entity)

-- Type-specific wrapping
local ewrap = instance.Types.Entity.Wrap
local eunwrap = instance.Types.Entity.Unwrap
```

## Development Workflow

### Adding a New Library

1. **Choose the appropriate directory**:
   - `lua/starfall/libs_sh/` - Shared functionality
   - `lua/starfall/libs_cl/` - Client-only (rendering, audio, input)
   - `lua/starfall/libs_sv/` - Server-only (physics, NPCs)

2. **Create the library file** (e.g., `mylib.lua`):

```lua
-- Global to all starfalls
local checkluatype = SF.CheckLuaType
local haspermission = SF.Permissions.hasAccess
local registerprivilege = SF.Permissions.registerPrivilege

-- Register permissions if needed
registerprivilege("mylib.doSomething", "Do Something", "Allows doing something", {
    usergroups = { default = 1 }  -- 0=block, 1=default, 2=allow
})

SF.RegisterLibrary("mylib")

return function(instance)
    local library_table = instance.env

    -- Cache wrap/unwrap functions
    local checkpermission = instance.player ~= SF.Superuser and haspermission

    --- Does something useful
    -- @param number value The input value
    -- @return number result The result
    library_table.doSomething = function(value)
        checkluatype(value, TYPE_NUMBER)
        if checkpermission then
            checkpermission(instance, nil, "mylib.doSomething")
        end

        -- Implementation
        return value * 2
    end

    return library_table
end
```

3. **Test in-game**: The library will auto-load on next restart or use `sf_reloadlibrary mylib`

4. **Document**: Use LDoc-style comments for API documentation generation

### Adding a New Hook

Hooks are registered in `lua/starfall/libs_sh/hook.lua`:

```lua
--- Called when something happens
-- @name SomethingHappened
-- @class hook
-- @server  -- or @client or @shared
-- @param Entity ent The entity involved
add("SomethingHappened")
```

With permission checking:
```lua
add("SensitiveHook", "hookname", returnOnlyOnYourself, "sensitive.hook")
```

### Modifying Entities

Each entity has three files:
- `init.lua` - Server-side logic
- `cl_init.lua` - Client-side logic
- `shared.lua` - Shared definitions

Example: `lua/entities/starfall_processor/init.lua`

Follow the existing pattern:
1. Include shared definitions
2. Set up networking
3. Initialize instance on spawn
4. Handle cleanup on remove

### Adding Examples

Examples go in `lua/starfall/examples/`:

```lua
--@name Example Name
--@author Your Name
--@shared  -- or @client or @server

-- Example code demonstrating a feature
hook.add("think", "example", function()
    -- Do something
end)
```

## Code Style & Conventions

### General Conventions

1. **Indentation**: Tabs (not spaces)
2. **Local caching**: Cache frequently used functions at file scope
   ```lua
   local checkluatype = SF.CheckLuaType
   local haspermission = SF.Permissions.hasAccess
   ```

3. **Type checking**: Always validate input types
   ```lua
   checkluatype(arg1, TYPE_ENTITY)
   checkluatype(arg2, TYPE_NUMBER)
   ```

4. **Permission checking**: Check permissions before privileged operations
   ```lua
   if checkpermission then
       checkpermission(instance, target, "privilege.name")
   end
   ```

5. **Comments**: Use LDoc format for documentation
   ```lua
   --- Brief description
   -- @param Type name Description
   -- @return Type Description
   ```

### Naming Conventions

- **Libraries**: lowercase with underscores (`bass.lua`, `render.lua`)
- **Functions**: camelCase (`getPosition`, `setColor`)
- **Constants**: UPPERCASE (`SF.Version`, `TYPE_ENTITY`)
- **Private functions**: Local at file scope
- **Entity folders**: `starfall_<type>` pattern

### Performance Best Practices

1. **Cache global references**: Reduces table lookups
   ```lua
   local math_sin = math.sin
   local math_cos = math.cos
   ```

2. **Avoid repeated unwrapping**: Store unwrapped objects if used multiple times
   ```lua
   local ent = eunwrap(wrapped_ent)
   -- Use ent multiple times
   ```

3. **Use local variables**: Faster than global access

4. **Respect quotas**: Be mindful of CPU time in loops

## Git & PR Guidelines

### Commit Message Format

Follow the existing pattern from recent commits:

- **Action verb first**: Add, Fix, Improve, Optimize, Update, Remove
- **Be specific**: "Fix Entity:setBoneMatrix" not "Fix bug"
- **Reference PRs**: Include PR number if applicable (#2199)
- **Keep it concise**: Single line preferred

Examples:
```
Add game.getIPAddress (#2187)
Fix Entity:setBoneMatrix (#2188)
Improve count check
Optimize SF.WaitForEntity (#2199)
Small plystream optimization (#2198)
```

### Pull Request Process

1. **Create feature branch**: Work on a dedicated branch
2. **Test in-game**: Ensure changes work in Garry's Mod
3. **Check documentation**: Add/update LDoc comments
4. **Create PR**: Target `master` branch
5. **Automated checks**: Documentation build must pass
6. **Review**: Wait for maintainer review
7. **Merge**: Maintainers will merge and auto-deploy

### Branch Strategy

- **`master`**: Main branch, auto-deploys to Steam Workshop
- **`docgen`**: Documentation generation branch
- **Feature branches**: Use descriptive names

### CI/CD Workflows

1. **`doc_build.yml`**: Validates documentation builds on PRs
2. **`doc_deploy.yml`**: Deploys docs to GitHub Pages
3. **`workshop.yml`**: Auto-publishes to Steam Workshop on master push

## Testing & Quality Assurance

### Current Testing Approach

StarfallEx does **not** have a formal unit test framework. Testing is done through:

1. **Example scripts**: Located in `lua/starfall/examples/`
2. **In-game testing**: Manual testing in Garry's Mod
3. **Community feedback**: Via Discord and GitHub issues
4. **PR reviews**: Code review by maintainers

### Testing Your Changes

When making changes, you should:

1. **Test in Garry's Mod**:
   - Install as development addon
   - Create a Starfall chip
   - Test affected functionality
   - Check console for errors

2. **Test both realms**:
   - If shared/server: Test on dedicated server
   - If client: Test rendering, UI, input
   - Check console on both client and server

3. **Test edge cases**:
   - Invalid inputs (nil, wrong types)
   - Permission denied scenarios
   - Quota limits
   - Entity removal during execution

4. **Check examples**: Verify existing examples still work

### Common Issues to Check

- **Type safety**: Always unwrap before using game objects
- **Nil checks**: Validate entity validity
- **Realm guards**: SERVER/CLIENT conditional code
- **Permission checks**: Don't bypass security
- **Memory leaks**: Clean up hooks/timers on removal
- **Quota exhaustion**: Avoid expensive operations in tight loops

## Important Notes for AI Assistants

### What You Should Do

1. **Read before modifying**: Always use Read tool before Edit/Write
2. **Follow existing patterns**: Match the style of surrounding code
3. **Cache references**: Use local variables for performance
4. **Check permissions**: Add permission checks for privileged operations
5. **Document thoroughly**: Use LDoc comments for all public functions
6. **Test realm-appropriate**: Understand CLIENT/SERVER/SHARED contexts
7. **Validate inputs**: Use `SF.CheckLuaType()` for all parameters
8. **Handle edge cases**: Check for nil, invalid entities, etc.
9. **Respect quotas**: Be mindful of performance impact
10. **Update examples**: If adding features, provide examples

### What You Should NOT Do

1. **Don't bypass security**: Never skip permission checks
2. **Don't create malware**: This is a user scripting environment
3. **Don't break compatibility**: Avoid breaking changes to existing APIs
4. **Don't ignore realms**: Don't use client functions on server or vice versa
5. **Don't guess at patterns**: Use Task tool to explore if uncertain
6. **Don't create unnecessary files**: Prefer editing existing files
7. **Don't skip type checking**: Always validate inputs
8. **Don't commit directly to master**: Use feature branches
9. **Don't add dependencies**: Keep the addon self-contained
10. **Don't modify binary modules**: DLL files are external dependencies

### Common Tasks

#### Finding where a feature is implemented
```
Use Task tool with subagent_type=Explore to search the codebase
Example: "Find where entity positions are set"
```

#### Adding a new entity method
```
1. Find the entity library: lua/starfall/libs_sh/entities.lua
2. Add the method following existing patterns
3. Add permission if needed
4. Document with LDoc comments
5. Test in-game
```

#### Fixing a bug
```
1. Reproduce the issue in-game
2. Check console for errors
3. Use Task/Grep to find relevant code
4. Apply fix following code conventions
5. Test the fix in-game
6. Commit with descriptive message
```

#### Adding a new library feature
```
1. Identify correct library file (libs_sh/cl/sv)
2. Read the entire file first
3. Add function following module pattern
4. Register permissions if needed
5. Add LDoc documentation
6. Test in-game with example script
7. Consider adding to examples/
```

### Understanding the Codebase

Key files to understand:

1. **`sflib.lua`** (2,611 lines): Module system, type registration, library loading
2. **`instance.lua`** (725 lines): Execution context, quotas, instance lifecycle
3. **`permissions/core.lua`**: Permission system implementation
4. **`preprocessor.lua`**: How user scripts are compiled
5. **`libs_sh/entities.lua`**: Entity manipulation API
6. **`libs_cl/render.lua`**: Client rendering API
7. **`entities/starfall_processor/init.lua`**: Main chip implementation

### Useful Commands

- **Reload library**: `sf_reloadlibrary <name>` - Hot-reload a library
- **Version**: Git commit hash embedded in `SF.Version`
- **Examples**: Check `lua/starfall/examples/` for usage patterns

## Resources

### Documentation
- **API Docs**: http://thegrb93.github.io/StarfallEx/
- **Examples**: `/lua/starfall/examples/`
- **Discord**: https://discord.gg/yFBU8PU

### External Dependencies
- **Joystick (Windows)**: https://github.com/MattJeanes/Joystick-Module
- **XInput (Windows)**: https://github.com/mitterdoo/garrysmod-xinput
- **Luasocket**: https://github.com/danielga/gmod_luasocket
- **Joystick (Linux)**: https://gitlab.h08.us/puff/joystick-module-linux

### Key Garry's Mod Concepts
- **Realms**: CLIENT vs SERVER execution contexts
- **AddCSLuaFile**: Sends files to clients
- **include**: Loads Lua files
- **ENT table**: Entity definition structure
- **SWEP table**: Weapon definition structure
- **STool table**: Tool definition structure

## Quick Reference

### Directory Purpose
| Directory | Purpose | Realm |
|-----------|---------|-------|
| `lua/autorun/` | Auto-initialization | Mixed |
| `lua/entities/` | Entity definitions | Mixed |
| `lua/starfall/libs_sh/` | Shared libraries | Both |
| `lua/starfall/libs_cl/` | Client libraries | Client |
| `lua/starfall/libs_sv/` | Server libraries | Server |
| `lua/starfall/permissions/` | Security system | Mixed |
| `lua/starfall/editor/` | Code editor | Client |
| `lua/starfall/examples/` | Example scripts | Mixed |
| `lua/weapons/` | Tool gun integration | Mixed |

### Common Type Codes
```lua
TYPE_ENTITY    -- Game entities
TYPE_NUMBER    -- Numbers
TYPE_STRING    -- Strings
TYPE_BOOL      -- Booleans
TYPE_TABLE     -- Tables
TYPE_FUNCTION  -- Functions
TYPE_VECTOR    -- 3D vectors
TYPE_ANGLE     -- Rotation angles
```

### Permission Provider Settings
```lua
0 = "block"     -- Block operation
1 = "default"   -- Use provider's default logic
2 = "allow"     -- Allow operation
```

---

**Last Updated**: 2025-11-14
**Repository**: https://github.com/thegrb93/StarfallEx
**Version**: Based on commit 3ad1f44
