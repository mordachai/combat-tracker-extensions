# Combat Tracker Extensions - Project Context

## Project Overview
**Combat Tracker Extensions** is a system-agnostic Foundry VTT module that enhances the default Combat Tracker. It introduces features such as reverse initiative order, combatant groups (with shared or independent initiative), custom combat phases, round sets, and advanced options for obscuring combatant information (names, initiative, effects) from players.

**Key Technologies:**
*   **JavaScript (ES Modules):** Core logic, utilizing Foundry VTT's API and Hooks.
*   **Handlebars (.hbs):** Templating for custom UI elements (forms, dialogs).
*   **CSS:** Styling for the combat tracker and custom dialogs.
*   **Foundry VTT API:** Extensive use of `Hooks` (`init`, `ready`, `renderCombatTracker`, `updateToken`, etc.) and `libWrapper` for patching core methods.

## Building and Running
This project does not require a build step. It is distributed as raw ES modules.

**To Run/Test:**
1.  Ensure the `combat-tracker-extensions` directory is located in your Foundry VTT `Data/modules/` folder.
2.  Launch Foundry VTT.
3.  Enable the "Combat Tracker Extensions" module in your World's "Manage Modules" settings.
4.  Configure settings via "Configure Settings" -> "Module Settings".

## Development Conventions
*   **Entry Point:** `scripts/combat-tracker-extensions.js` handles initialization (`init`, `ready` hooks) and main event listeners.
*   **Core Overrides:** `scripts/fvttt_core_overrides.js` contains functions that monkey-patch Foundry's `Combat` and `CombatTracker` classes using `libWrapper`.
*   **Settings:** Defined in `scripts/settings-registration.js` and managed via `scripts/module-settings-form.js`.
*   **UI Components:** Custom forms (e.g., Phase Editor, Group Editor) extend `FormApplication` and use templates from the `templates/` directory.
*   **Localization:** All user-facing text should be localized using keys in `languages/en.json`.
*   **Styling:** CSS is split into functional files: `combat-tracker-extensions.css` (main), `dropdownmenu.css` (menus), `module-settings.css` (settings).

## Key File Structure
*   `module.json`: Manifest file defining the module, version, entry points, and dependencies.
*   `scripts/`
    *   `combat-tracker-extensions.js`: Main module logic and hook registration.
    *   `fvttt_core_overrides.js`: Patches for `Combat.prototype._sortCombatants`, `Combat.prototype.rollAll`, etc.
    *   `initiative-groups.js`: Logic for managing combatant groups.
    *   `*-form.js`: Logic for custom settings/editor windows.
*   `templates/`: Handlebars templates for UI components.
*   `styles/`: CSS stylesheets.
*   `languages/`: Localization files (`en.json`).
