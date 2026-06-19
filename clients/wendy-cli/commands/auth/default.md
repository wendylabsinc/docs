Shows the current default Wendy Cloud session, or clears it.

When multiple auth sessions exist (e.g. personal cloud and a staging instance), `wendy auth use` sets the default. `wendy auth default` shows which session is currently the default, and `--clear` removes it so the CLI falls back to prompting.

## Usage

```sh
wendy auth default          # show the current default session
wendy auth default --clear  # remove the default (fall back to prompting)
```

## Flags

| Flag | Description |
|------|-------------|
| `--clear` | Remove the default session setting. Subsequent commands that require a cloud session will prompt for one interactively. |

## See also

- [`wendy auth use`](./use.md) — set the default session
- [`wendy auth login`](./login.md) — add a new session
