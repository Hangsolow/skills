# skills
A repo containing skills for Copilot and friends

## Plugins

### [optimizely-cms](plugins/optimizely-cms/)

Optimizely CMS skills for GitHub Copilot, including step-by-step guidance for upgrading from CMS 12 to CMS 13.

| Skill | Description |
|-------|-------------|
| [cms13-migration](plugins/optimizely-cms/skills/cms13-migration/SKILL.md) | Step-by-step guidance for upgrading an Optimizely CMS 12 project to CMS 13, fixing compile errors, and replacing removed APIs. |

#### Install

Add this repository as a plugin marketplace in VS Code:

```json
// .vscode/settings.json
"chat.plugins.marketplaces": ["Hangsolow/skills"]
```

Or install directly from source via the Command Palette: **Chat: Install Plugin From Source**.
