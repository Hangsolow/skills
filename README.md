# skills
A repo containing skills for Copilot and friends

## Plugins

### [optimizely-cms](plugins/optimizely-cms/)

Optimizely CMS skills for GitHub Copilot, including step-by-step guidance for upgrading from CMS 12 to CMS 13 and authoring content types.

| Skill | Description |
|-------|-------------|
| [cms13-migration](plugins/optimizely-cms/skills/cms13-migration/SKILL.md) | Step-by-step guidance for upgrading an Optimizely CMS 12 project to CMS 13, fixing compile errors, and replacing removed APIs. |
| [cms-content-types](plugins/optimizely-cms/skills/cms-content-types/SKILL.md) | Creating and updating page types and block types in Optimizely CMS 12 and 13 — properties, attributes, grouping, culture-specificity, block-as-property, and safe refactoring. |

#### Install

**VS Code — Extensions GUI (recommended)**

1. Open the Extensions view (<kbd>Ctrl+Shift+X</kbd> / <kbd>⇧⌘X</kbd>) and search `@agentPlugins`
2. Find **optimizely-cms** and click **Install**

**VS Code — settings.json**

Add the marketplace and VS Code will discover the plugin automatically:

```json
// .vscode/settings.json  (or user settings.json)
"chat.plugins.marketplaces": ["Hangsolow/skills"]
```

**Copilot CLI**

```sh
# Register the marketplace once
copilot plugin marketplace add Hangsolow/skills

# Then install the plugin
copilot plugin install optimizely-cms@hangsolow-skills
```

**Direct install (no marketplace registration)**

```sh
# VS Code Command Palette → Chat: Install Plugin From Source
# or CLI:
copilot plugin install Hangsolow/skills:plugins/optimizely-cms
```
