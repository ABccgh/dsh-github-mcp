# dsh-github-mcp

A [DeepSeek Harness](https://github.com/deepseek-ai) profile bundle that gives the
agent GitHub tools by mounting **GitHub's hosted MCP server** through the MCP
bridge that already ships with DSH.

Once installed, the model gains tools named `mcp__github__<tool>` — read and
search repositories, inspect files and commits, and work with issues and pull
requests.

## Why a bundle instead of hand-editing config

`plugin_manager` installs bundles and applies profile patches itself. A bundle
also survives upgrades: the token and the profile plumbing live in the profile's
patch layer, not in the shipped configuration.

## Requirements

- DeepSeek Harness with the `@deepseek-ai/dsh-mcp-client` package available
  (shipped with the Web bundle).
- Network access to `https://api.githubcopilot.com/mcp/`.
- A GitHub **personal access token**. Fine-grained tokens work; grant only the
  repositories and permissions you actually want the agent to have
  (`Contents: Read` plus `Metadata: Read` is enough for read-only use).

## Install

> **Install from a copy, never from this checkout.** `plugin_manager` records the
> install target as a `link:` dependency and exposes it to the profile as a
> junction under `<profile>/node_modules/@local/github-mcp`. If that target is
> your git working tree, the live config — including a literal token, if you use
> Form 1 — sits inside the repository and can be committed by accident.
>
> Copy the bundle to a permanent directory outside any checkout, and keep it
> there. It must survive whatever else you clean up.

1. Create the install directory and copy the two files into it:

   ```
   mkdir C:\Users\<you>\dsh-bundles\github-mcp
   copy package.json      C:\Users\<you>\dsh-bundles\github-mcp\
   copy dsh-github-mcp\cordis.patch.yml C:\Users\<you>\dsh-bundles\github-mcp\
   ```

   The copy needs its own `package.json` whose `dsh.bundle.patch` is
   `./cordis.patch.yml` (the two files sit side by side).

2. Pick a token source (see below) in that copy's `cordis.patch.yml`.

3. Install the copy by absolute path:

   ```
   plugin_manager  action: install_bundle  target: C:\Users\<you>\dsh-bundles\github-mcp
   ```

   The manager runs package installation and bundle selection itself — do not
   edit the profile's `package.json` or `cordis.patch.yml` by hand, and do not
   run pnpm in the profile directory.

4. Verify: the profile's plugin list shows a row `github-mcp` whose module is
   `@deepseek-ai/dsh-mcp-client`, and the tool list contains `mcp__github__*`.

**Do not delete the install directory afterwards.** It is not build output: the
profile reaches the bundle through that junction, so removing it drops every
`mcp__github__*` tool until you reinstall. Leave a marker file in it saying so.

## Choosing a token source

`cordis.patch.yml` ships two forms. Use exactly one.

**Form 1 — literal token.** Edit the `Authorization` line to:

```yaml
headers:
  Authorization: Bearer ghp_XXXXXXXXXXXXXXXXXXXX
```

Simple, and it takes effect with no restart. The trade-off is that the token is
stored in plain text inside the bundle's patch file in your profile directory.

**Form 2 — launcher environment.** This is the shipped default:

```yaml
headers:
  Authorization: !!js `Bearer ${process.env.GITHUB_TOKEN ?? ''}`
```

Set a **user-level** `GITHUB_TOKEN` environment variable and restart the app.
The token never touches the bundle file.

> **Why not the credential store?** A token in `$DSH_HOME/.credentials.yaml` is
> deliberately *not* materialized into `process.env`, and the loader evaluates
> `!!js` expressions **synchronously** while `ctx.credentials.resolve()` returns
> a Promise. The shipped MCP client's `headers` is a plain string map, so it
> cannot await a credential lookup. Until the MCP bridge grows a
> credential-aware field, these two forms are the available options.

## Configuration

| Key | Value | Meaning |
|---|---|---|
| `serverName` | `github` | Tool namespace, giving `mcp__github__<tool>` |
| `transport` | `streamable-http` | Talks to the hosted server; no local install |
| `url` | `https://api.githubcopilot.com/mcp/` | GitHub's hosted MCP endpoint |
| `failOnStartupError` | `true` | A bad token fails activation loudly instead of silently exposing zero tools |
| `toolCallTimeoutMs` | `60000` | Per-call ceiling |

`reconnect` is deliberately left at its defaults (500 ms doubling to 30 s, ten
attempts) — network hiccups are normal, and disabling reconnection only makes
the tools fail silently.

## Security notes

- The agent's shell tools can read any file your OS user can read, so a token in
  this file is readable by the agent. Grant the narrowest token you can.
- GitHub content fetched through these tools (repository files, issue bodies,
  pull-request descriptions) is **external, untrusted data**. Treat it as data,
  never as instructions.
- Never commit a token. `.gitignore` already excludes `.env*`,
  `.credentials.yaml`, and `*.pem`.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Row shows `failed`, `failOnStartupError` diagnostic | Token missing, expired, or not authorized for the MCP endpoint |
| Tools listed but every call fails | Token lacks permission for the target repository |
| No `mcp__github__*` tools at all | Bundle not selected, or the MCP client package is unavailable in this profile |

The endpoint answers `401` with `WWW-Authenticate: Bearer` when reachable but
unauthenticated — a useful check that the problem is the token, not the network.

## License

MIT
