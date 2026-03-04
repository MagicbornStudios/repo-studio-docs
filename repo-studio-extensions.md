# Repo Studio Extensions

Repo Studio is extension-first and project-scoped.

Active project extensions are loaded from:

- `.repo-studio/extensions/<extension-id>/manifest.json`
- optional layout file referenced by `layoutSpecPath`

## Extension manifest v1

```json
{
  "manifestVersion": 1,
  "id": "story",
  "label": "Story",
  "workspaceId": "story",
  "workspaceKind": "story",
  "description": "Story workspace extension",
  "layoutSpecPath": "layout.generated.json",
  "assistant": {
    "forge": {
      "aboutWorkspace": {
        "title": "Story Workspace",
        "summary": "Workspace purpose and scope",
        "context": ["content/story"]
      },
      "tools": [
        {
          "name": "forge_open_about_workspace",
          "action": "open_about_workspace",
          "label": "Open About Workspace",
          "description": "Open workspace context modal"
        }
      ]
    }
  }
}
```

`layout.generated.json`:

```json
{
  "workspaceId": "story",
  "panelSpecs": [
    { "id": "story", "label": "Story", "rail": "left" },
    { "id": "viewport", "label": "Viewport", "rail": "main" },
    { "id": "assistant", "label": "Assistant", "rail": "right" }
  ],
  "mainPanelIds": ["viewport"],
  "mainAnchorPanelId": "viewport"
}
```

## APIs

`GET /api/repo/extensions`

- Returns installed project extensions from active root.

`GET /api/repo/extensions/registry`

- Returns installable extension entries from `vendor/repo-studio-extensions/extensions`.
- Returns studio examples from `vendor/repo-studio-extensions/examples/studios`.

`POST /api/repo/extensions/install`

- Installs installable entries only.
- Studio example ids are rejected as non-installable.

`POST /api/repo/extensions/remove`

- Removes installed extension folder from active project.

## Installables vs examples

- Installables are runtime-compatible extensions with manifest contract.
- Studio examples are browse-only references with `example.json` metadata.
- Examples appear in the Extensions workspace as link-out cards and have no install action.
- Current installables include `story` and `env-workspace` from registry submodule.

## Runtime model

- Host-rendered extension kinds only (no arbitrary extension JS execution).
- `workspaceKind: "story"` uses host Story renderer.
- `workspaceKind: "env"` uses host Env renderer.
- `workspaceKind: "generic"` uses host generic renderer.
- Extension/studio authors should build UI surfaces with `@forge/dev-kit` (or `@forge/shared`), not app-internal imports such as `@/`.
- Current installables `story` and `env-workspace` are adapter kinds in this slice; any bundled `src/` content is reference snapshot data, not portable runtime JS loading.
- Layout resolution priority:
1. Extension-discovered `layout`
2. Generic fallback

## AI tool contract

- Forge merges extension tools only for active workspace.
- Base Forge tool remains (`forge_open_about_me`).
- Workspace proof tool is `forge_open_about_workspace`.

## Notes

- Story is optional per project and discovered like any other extension.
- Repo Studio no longer shows a Story-specific install prompt in the root shell.
- Use `templates/repo-studio-extension/` to scaffold new installables.
- Installed extensions are copied into `<activeProject>/.repo-studio/extensions/<id>/`.
- You can iterate in place by editing installed extension files directly; if updates are not picked up immediately, reload extensions or restart Repo Studio.
