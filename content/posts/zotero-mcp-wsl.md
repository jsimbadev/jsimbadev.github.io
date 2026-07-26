+++
title = "Connecting Zotero MCP from WSL"
date = "2026-07-26"
description = "A small tutorial on connecting Zotero on Windows to Codex running inside WSL."
weight = 1
tags = ["research tooling", "zotero", "mcp", "scientific software", "agents"]
+++

Using agents for research usually comes down to a question of trust.

Papers get handed to agents by URL. Abstracts get pasted into chats. A question starts with one source, drifts into another, and by the end it is not always obvious whether the model is still attached to the paper it claims to be discussing. Around the centre of my current research interests, I can usually catch that. I know the core papers well enough to notice when two sources are being blurred together, or when a citation is being used a little too conveniently.

The harder case is the edge of the library. There are papers saved because they looked relevant, papers read once, papers meant for later, papers that are only half-remembered. With these, hallucination is harder to detect, let alone source confusion. 

Zotero is my choice of database, and so it's only natural to get that context into my agentic workflows. Papers, metadata, PDFs, notes, tags, collections, and the small bits of judgement that accumulate around a research library. [`zotero-mcp`](https://github.com/54yyyu/zotero-mcp) makes that library available through MCP.

The setup below assumes this shape:

```text
Codex CLI      -> running inside WSL2
zotero-mcp     -> installed inside WSL2
Zotero desktop -> installed on the Windows host
Zotero data    -> stored under C:\Users\<username>\Zotero
```

Zotero has both a remote API and a local API. For local agents, the local API is the more natural starting point. Zotero desktop is open on Windows, the MCP server runs in WSL, and the agent can read library context without starting from web API credentials or write access.

## WSL and Zotero

The first boundary is the local API. Zotero desktop has to allow other applications on the machine to communicate with it:

```text
Settings -> Advanced -> Allow other applications on this computer to communicate with Zotero
```

With Zotero open, the local API is served on port `23119`. Windows side should be able to reach the connector ping endpoint:

```powershell
curl.exe http://127.0.0.1:23119/connector/ping
```

If that fails, the issue is still on the Zotero or Windows side: Zotero is not running, local communication is disabled, or something is blocking the port.

Codex and `zotero-mcp` are running inside WSL, while Zotero desktop is running on Windows. A WSL process talking to `127.0.0.1:23119` needs to land on the Windows Zotero process. Mirrored networking makes that localhost behaviour line up.

The relevant Windows file is:

```text
C:\Users\<username>\.wslconfig
```

Open it from PowerShell:

```powershell
notepad $env:USERPROFILE\.wslconfig
```

Add:

```ini
[wsl2]
networkingMode=mirrored
```

Then restart WSL:

```powershell
wsl --shutdown
```

After WSL starts again, the same ping test should work from inside WSL:

```bash
curl -s http://127.0.0.1:23119/connector/ping
```

A Linux process in WSL can reach the Zotero desktop process through localhost. If this still fails, check `wsl --version`, Windows version, and firewall settings before debugging the MCP server.

The next boundary is the filesystem. Zotero is installed on Windows, so the data directory is usually:

```text
C:\Users\<username>\Zotero
```

From WSL, the same directory is normally visible at:

```text
/mnt/c/Users/<username>/Zotero
```

The important database file is:

```text
Zotero/zotero.sqlite
```

The tooling can be pointed at that file explicitly with `ZOTERO_DB_PATH`, but if that is unset it eventually looks for the default Zotero directory under `~/Zotero`. In a WSL setup, `~/Zotero` and the Windows Zotero directory are usually different places.

If `~/Zotero` already exists as a real directory, do not overwrite it casually. Otherwise, a symlink makes the expected path resolve to the real one:

```bash
ls -ld "$HOME/Zotero"
ln -s /mnt/c/Users/<username>/Zotero "$HOME/Zotero"
```

Or use the explicit database path instead:

```bash
export ZOTERO_DB_PATH="/mnt/c/Users/<username>/Zotero/zotero.sqlite"
```

## MCP Setup

For a `uv`-based Python workflow, the install is:

```bash
uv tool install zotero-mcp-server
```

That provides both `zotero-mcp` and `zotero-cli`. The project setup command is:

```bash
zotero-mcp setup
```

For local read-only use, the key environment setting is:

```text
ZOTERO_LOCAL=true
```

Local read-only mode does not need `ZOTERO_API_KEY` or `ZOTERO_LIBRARY_ID`. That is a useful default for this workflow: let the agent read bibliographic context first, without giving it mutation privileges.

The command-line smoke test:

```bash
which zotero-mcp
zotero-mcp setup-info
ZOTERO_LOCAL=true zotero-cli library info
ZOTERO_LOCAL=true zotero-cli search "hamiltonian" --limit 5
```

The search term is not important. The useful signal is whether a process running in WSL can see the Zotero library expected from the Windows desktop app.

Codex reads MCP servers from `config.toml`. For a user-level setup:

```text
~/.codex/config.toml
```

The server can be added from the CLI:

```bash
codex mcp add zotero --env ZOTERO_LOCAL=true -- zotero-mcp serve --transport stdio
```

or configured directly:

```toml
[mcp_servers.zotero]
command = "zotero-mcp"
args = ["serve", "--transport", "stdio"]

[mcp_servers.zotero.env]
ZOTERO_LOCAL = "true"
```

After restarting Codex, `/mcp` should show the Zotero server. Now you can start asking your agent questions against a curated library:

```text
Search my Zotero library for papers about adaptive MCMC.
```

```text
Find papers in my library that are relevant to Markovian noise in stochastic approximation.
Give me the Zotero metadata for the most relevant ones.
```

That is the part that changes the research loop. The agent is no longer only producing a literature-shaped answer from general model knowledge. It can inspect a local bibliography, return concrete Zotero items, and make the citation trail easier to audit.

## Why This Matters

There is a difference between asking:

```text
What are common references for adaptive MCMC?
```

and asking:

```text
Which papers in my Zotero library are relevant to adaptive MCMC,
and which of them have notes or annotations?
```

This matters for scientific agentic workflows because the work moves across tooling. An agent might inspect code, summarize an experiment, help draft a methods paragraph, and then suggest citations. Without library access, the citation step is disconnected from the rest of the loop. With Zotero connected, the bibliography becomes part of the same working context.

Rather than chasing citations around manually, the agent should have access to these. 

## Cons

The cost is context. Every MCP server adds some overhead. Every tool response competes with the code, the experiment, and the actual question. A Zotero library can be large, and even a small search result can become noisy if it brings back too many titles, abstracts, etc...

That is not only a token problem. It is also a reasoning problem. A model with twenty loosely related papers in context is not necessarily in a better position than a model with three good ones. Sometimes it is in a worse position, because the task now includes sorting through irrelevant bibliography. So you ought to be responsible with how you navigate your harness.

So it's not:

```text
dump Zotero into the prompt
```

More like:

```text
query the library
return a small number of relevant items
include enough metadata to audit the result
only pull notes or full text when needed
```

## TODOs
- Leverage custom skills to constrain the zotero-cli
  - Use that `--limit` to keep resource pressure at bay
- With semantic search enabled, zotero-cli uses [chromadb](https://docs.trychroma.com/docs/overview/introduction). I haven't dug deep into it, but perhaps there are some levers for optimization. This is where RAG comes in. Index the useful parts of the Zotero library, retrieve a small number of passages or notes, and return them with enough citation metadata that the agent's answer can be checked.


## Sources

- [`54yyyu/zotero-mcp`](https://github.com/54yyyu/zotero-mcp)
- [Zotero local API documentation](https://www.zotero.org/support/dev/web_api/v3/local_api)
- [Microsoft WSL `.wslconfig` settings](https://learn.microsoft.com/en-us/windows/wsl/wsl-config#configuration-settings-for-wslconfig)

You can also read [about me](../../about/) or reach me through the [contact page](../../contact/).
