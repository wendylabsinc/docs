Sets the default Wendy Cloud session when multiple auth sessions exist.

The selector is matched against stored sessions in this order:

1. If the selector is all digits, it is matched against each session's organization ID.
2. Otherwise, it is matched as a case-insensitive substring of the gRPC endpoint or dashboard URL.

An error is returned if the selector matches zero sessions or more than one session.

When called interactively with no argument, a picker is shown to select from available sessions.

## Usage

```sh
wendy auth use [selector]
wendy auth use               # interactive picker
wendy auth use 42            # match by organization ID
wendy auth use staging       # match by endpoint substring
```

## See also

- [`wendy auth default`](./default.md) — show or clear the current default session
- [`wendy auth login`](./login.md) — add a new session
