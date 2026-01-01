# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Combat Tracker Extensions is a system-agnostic Foundry VTT module that extends the standard Combat Tracker with features like reverse initiative order, combatant groups, custom combat phases, round sets, and advanced options for obscuring combatant information from players.

**Current Foundry VTT Compatibility:** Version 13+

### Foundry VTT v13 API Changes (Applied)
This module has been fully updated for v13+ only (no backward compatibility):
- All `combatTracker.viewed` → `combatTracker.combat`
- All `game.combats.apps[0].viewed` → `game.combats.apps[0].combat`
- DOM selector changed: `#combat-tracker` → `#combat`
- `getViewedCombat(combatTracker)` helper provides fallback to `game.combats?.viewed`
- All namespaced classes updated:
  - `CombatTracker` → `foundry.applications.sidebar.tabs.CombatTracker`
  - `Token` → `foundry.canvas.placeables.Token`
- **CRITICAL**: `renderCombatTracker` hook now passes raw DOM element, not jQuery object
  - Must wrap in jQuery: `html = $(html)` at start of hook
- Added null safety checks for `canvas.scene` (may be undefined during rendering)
- Removed v10/v11 compatibility code and version checks

### Application Framework
Currently uses V1 Application framework (FormApplication). This works in v13 but shows deprecation warnings. Migration to ApplicationV2 is recommended before v16 but not required yet.

## Development Setup

This module requires no build step - it uses raw ES modules that Foundry VTT loads directly.

**To test changes:**
1. Ensure this directory is in your Foundry VTT `Data/modules/` folder
2. Launch Foundry VTT
3. Enable "Combat Tracker Extensions" in your world's Manage Modules
4. Reload Foundry if changing any `init` hook or initialization code
5. Configure module settings via Configure Settings → Module Settings

**Important:** CSS files in `styles/` are automatically loaded by Foundry - you don't need to compile them.

## Architecture Overview

### Core Entry Point
`scripts/combat-tracker-extensions.js` - Main module initialization
- Registers all Foundry hooks (`init`, `ready`, `renderCombatTracker`, etc.)
- Sets up `libWrapper` patches for core Foundry methods
- Defines module constants like `DISPOSITIONS` with color schemes

### Critical Hook Lifecycle
1. **`init` hook** (`_onInit`):
   - Registers settings via `settingsRegistration()`
   - Applies `libWrapper` patches based on enabled settings
   - MUST reload Foundry if changing logic here
2. **`ready` hook** (`_onReady`):
   - Sets up player-specific overrides (like hiding token effects)
   - Initializes dropdown menu event listeners

### Core Systems

#### Combat Sorting Override
`scripts/fvttt_core_overrides.js` - Monkey-patches Foundry core methods
- `wrappedSortCombatants`: Custom sorting for reverse initiative, phases, and groups
- `wrappedRollAll`/`wrappedRollNPC`: Handle group initiative rolls
- Uses `libWrapper.register()` to safely override `Combat.prototype._sortCombatants`
- Special handling for pf2e system compatibility

#### Initiative Groups
`scripts/initiative-groups.js` - Group management system
- Groups stored in `combat.flags.combattrackerextensions.initiativegroups` array
- Each group has: `id`, `name`, `color`, `sharesinitiative` boolean
- Sharing groups: All members get same initiative/phase (if enabled)
- Non-sharing groups: Logical grouping only for batch commands
- Group leader is first member added (can be changed via dropdown)

#### Phases System
- Phases divide a Foundry round into sub-turns with separate initiative orders
- Stored in combatant flags: `combatant.flags.combattrackerextensions.phase`
- Phase definitions stored in world setting `OPTION_COMBAT_TRACKER_DEFINED_PHASES`
- Always includes an "Unset" phase (configurable name) as default

#### Round Sets
- Defines custom round sequences (e.g., Round 1, Round 2, Segment A, Segment B)
- Stored in world setting `OPTION_COMBAT_TRACKER_DEFINED_ROUNDSET`
- Each round in set can have its own phases

### Settings System
`scripts/settings-registration.js` + `scripts/setting-constants.js`
- All settings defined in `SETTINGATTRIBUTE` constant object
- Structure: `{ID, TYPE, DEFAULT, CATEGORY, SCOPE, CONFIG, REQUIRESRELOAD}`
- `TYPE` can be: `Boolean`, `Number`, `String`, or custom form class names
- Forms like `CombatTrackerExtensionsPhaseEditorForm` registered as menu items
- Access via `getModuleSetting(moduleId, SETTINGATTRIBUTE.SETTING_NAME.ID)`

### UI Components
All custom forms extend `FormApplication` and use Handlebars templates:
- `scripts/group-editor-form.js` + `templates/group-editor.hbs`
- `scripts/phase-editor-form.js` + `templates/phase-editor.hbs`
- `scripts/module-settings-form.js` + `templates/module-settings-form.hbs`
- `scripts/dropdownmenu.js` - Custom dropdown menu system for combat tracker

### Dropdown Menu System
`scripts/dropdownmenu.js` - Provides contextual menus for:
- Encounter controls (add tokens, manage groups, bulk commands)
- Individual combatants (join/leave groups, change phase/disposition)
- Phase controls (assign combatants to phases)

## Foundry VTT API Usage Patterns

### Use Namespaced Functions (V13+)
```javascript
// CORRECT - Use namespaced API
await foundry.applications.handlebars.loadTemplates([...]);

// WRONG - Don't use deprecated globals
await loadTemplates([...]);
```

### Accessing Combat Data (V13+)
```javascript
// Get current viewed combat from CombatTracker
const combat = combatTracker.combat ?? game.combats?.viewed;

// Get combatants
const combatants = combat?.combatants;

// Access combat tracker DOM
const tracker = document.querySelector("#combat");
const combatantElements = html.find('#combat .combatant');
```

### Combat & Combatant Data Structure
```javascript
// Combat flags
combat.flags.combattrackerextensions.initiativegroups = [
  {id: "abc123", name: "Orcs", color: "#ff0000", sharesinitiative: true}
];

// Combatant flags
combatant.flags.combattrackerextensions.phase = 0; // Phase index
combatant.flags.combattrackerextensions.initiativegroup = {
  id: "abc123",
  membernumber: 0 // Join order, used for sorting
};
combatant.flags.combattrackerextensions.hidden = true; // Visibility to players
combatant.flags.combattrackerextensions.namemasked = true; // Mask name from players
```

### LibWrapper Usage Pattern
```javascript
import { libWrapper } from './shim.js';

// In init hook:
libWrapper.register(moduleId, 'Combat.prototype._sortCombatants', wrappedSortCombatants);

// Wrapper function signature:
export function wrappedSortCombatants(wrapped, a, b) {
  // wrapped = original function
  // a, b = combatants to compare
  // Return comparison result for Array.sort()
}
```

## Key Concepts

### Obscuring System
Players see filtered combat tracker based on:
- Token visibility (fog of war, lighting)
- Token disposition (Friendly/Neutral/Hostile/Secret)
- Token ownership
- Individual combatant hidden/masked flags
Settings control which combinations hide initiative values vs entire combatants

### Duplicate Combatants
- Same token can have multiple combatants (multiple actions per round)
- All duplicates synchronized for defeated status
- Hover highlighting shows all duplicates of same token

### Scene Tools Integration
The module correctly creates buttons in scene controls using:
```javascript
// Proper pattern for scene control buttons
Hooks.on("getSceneControlButtons", (controls) => {
  // Add custom controls here
});
```

## Common Modification Patterns

### Adding a New Setting
1. Add entry to `SETTINGATTRIBUTE` in `scripts/setting-constants.js`
2. Add localization keys in `languages/en.json`:
   - `settings.settings.YOUR_SETTING.Name`
   - `settings.settings.YOUR_SETTING.Hint`
3. Access in code: `getModuleSetting(moduleId, SETTINGATTRIBUTE.YOUR_SETTING.ID)`

### Adding Combat Tracker UI Elements
1. Hook into `renderCombatTracker`:
```javascript
Hooks.on('renderCombatTracker', async (app, html, data) => {
  // Manipulate html jQuery object
});
```
2. Add styles to `styles/combat-tracker-extensions.css`
3. Use existing dropdown system or add new elements

### Modifying Sort Order
Edit `wrappedSortCombatants` in `scripts/fvttt_core_overrides.js`
- Returns negative if `a` should be before `b`
- Check `OPTION_COMBAT_TRACKER_REVERSE_INITIATIVE` flag
- Consider phase → initiative → group → member order → name → tokenId

## File Organization

```
combat-tracker-extensions/
├── scripts/
│   ├── combat-tracker-extensions.js  # Main entry point
│   ├── fvttt_core_overrides.js      # LibWrapper patches
│   ├── initiative-groups.js          # Group management
│   ├── settings-registration.js      # Setting registration
│   ├── setting-constants.js          # Setting definitions
│   ├── *-form.js                     # FormApplication classes
│   ├── dropdownmenu.js               # Dropdown menu system
│   └── shim.js                       # LibWrapper shim
├── templates/                        # Handlebars templates
├── styles/                           # CSS (auto-loaded)
├── languages/                        # Localization files
└── module.json                       # Module manifest
```

## Testing Scenarios

When making changes, test these scenarios:
1. **Reverse initiative** - Enable and verify low-to-high sorting
2. **Phases** - Add combatants to different phases, verify separate orders
3. **Groups (non-sharing)** - Create group, verify visual grouping only
4. **Groups (sharing)** - Create sharing group, verify initiative sync
5. **Obscuring** - As player, verify hidden/masked combatants respect visibility
6. **Duplicate combatants** - Add duplicate, verify defeated status sync

## Important Notes

- Module uses `libWrapper` shim for compatibility - don't bypass it
- Combat tracker rendering is expensive - minimize re-renders
- Player vs GM logic: Check `game.user.isGM` for conditional behavior
- Token visibility calculations use `token.visible` property per user
- All user-facing text MUST use localization keys from `languages/en.json`
