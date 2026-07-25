+++
title = "Connecting Zotero MCP from WSL"
date = "2026-07-25"
description = "A small tutorial on connecting Zotero on Windows to Codex running inside WSL."
weight = 1
tags = ["research tooling", "zotero", "mcp", "scientific software", "agents"]
+++

I have been using agents day to day as I research. Mostly this has been useful, but the workflow can get messy in exactly the places where I would like the most confidence.

I hand papers to agents by URL. I paste abstracts. I ask for comparisons across papers. Sometimes I ask for candidate citations. For the core papers in my current research interests, I can usually tell when the model is being sloppy. I know the nearby literature well enough to notice when two sources are being blurred together, or when a claim sounds a little too convenient.

The problem is the edge of the library. There are papers I know less well, papers I saved for later, papers I read once and half-remember, papers whose relevance is still unclear. In those cases, it is harder to tell whether the agent is using the source carefully or just producing something plausible-looking. The failure mode is not always dramatic hallucination. Sometimes it is just a soft confusion of sources.

In order to increase trust in the agents, I decided to let them peek into my Zotero libraries.

For this I used [`zotero-mcp`](https://github.com/54yyyu/zotero-mcp). The setup was fairly easy, but WSL adds a couple of details that are worth writing down.

## My setup

The shape of my setup is:

```text
Codex CLI      -> running inside WSL2
zotero-mcp     -> installed inside WSL2
Zotero desktop -> installed on the Windows host
Zotero data    -> stored under C:\Users\<username>\Zotero
```

Zotero offers both a remote web API and a local API. For now I am only interested in local agents working against my local library. The local route is attractive because Zotero desktop already has the data, and I do not need an API key for read-only use.

There are two practical consequences:

1. The MCP server running in WSL needs to reach Zotero's local API on the Windows host.
2. The tooling needs to find the Zotero data directory, especially the `zotero.sqlite` database.

## Enable the Zotero local API

First, Zotero itself has to allow local API access.

In Zotero, go to:

```text
Settings -> Advanced -> Allow other applications on this computer to communicate with Zotero
```

The `zotero-mcp` README notes that the local API must be enabled for local mode. Zotero's own local API documentation says the desktop client exposes the local API at:

```text
http://localhost:23119/api/
```

The connector server also has a simple ping endpoint. On the Windows side, with Zotero open, this should return a small "Zotero is running" style response:

```powershell
curl.exe http://127.0.0.1:23119/connector/ping
```

If this does not work on Windows, the problem is not WSL yet. Zotero is either not running, the local communication setting is disabled, or something on Windows is blocking the port.

## Make WSL localhost reach the Windows host

By default, a process inside WSL2 is not quite the same thing as a process running directly on Windows. The `zotero-cli` and `zotero-mcp` commands run inside the WSL network space, while Zotero desktop is listening from the Windows side.

The fix I used was WSL mirrored networking. Microsoft documents this in the `.wslconfig` settings. The file lives on the Windows side at:

```text
C:\Users\<username>\.wslconfig
```

or, from PowerShell:

```powershell
notepad $env:USERPROFILE\.wslconfig
```

I used:

```ini
[wsl2]
networkingMode=mirrored
```

Then restart WSL from PowerShell:

```powershell
wsl --shutdown
```

After starting WSL again, test from inside WSL:

```bash
curl -s http://127.0.0.1:23119/connector/ping
```

If that works, WSL can see the Zotero desktop process through localhost. That was the main bridge I needed.

One note: the Microsoft docs mark mirrored networking as a WSL2 setting and note Windows version requirements around newer networking options. If this does not work on a machine, I would check `wsl --version`, Windows version, and firewall settings before debugging `zotero-mcp` itself.

## Install zotero-mcp with uv tool

I followed the project README and installed with `uv tool`, which I would recommend if you already use `uv`:

```bash
uv tool install zotero-mcp-server
```

That gives you the `zotero-mcp` command, and also the standalone `zotero-cli` command.

I then ran setup:

```bash
zotero-mcp setup
```

For local read-only use, the important environment setting is:

```text
ZOTERO_LOCAL=true
```

The README is explicit that local read-only mode does not need `ZOTERO_API_KEY` or `ZOTERO_LIBRARY_ID`. If you want write operations, that becomes a hybrid setup: local reads plus Zotero web API writes. I am not starting there. I would rather first make the read path boring and predictable.

You can check where the command is installed with:

```bash
which zotero-mcp
zotero-mcp setup-info
```

And you can smoke-test the CLI with something simple:

```bash
ZOTERO_LOCAL=true zotero-cli library info
ZOTERO_LOCAL=true zotero-cli search "hamiltonian" --limit 5
```

The exact search term does not matter. The point is to check that the WSL process can talk to Zotero and that it is seeing the expected library.

## Point WSL at the Zotero data directory

The other small problem is the filesystem.

Zotero is installed on Windows, so my data directory is on the Windows side:

```text
C:\Users\<username>\Zotero
```

From WSL, that is usually:

```text
/mnt/c/Users/<username>/Zotero
```

The `zotero-mcp` tooling can be configured with `ZOTERO_DB_PATH`, but if that is unset it tries to locate the database and eventually falls back to the default `~/Zotero` location. In my setup, `~/Zotero` was not where the real Zotero directory lived, because my WSL home is not my Windows home.

So I made the WSL path look like what the tool expected:

```bash
ln -s /mnt/c/Users/<username>/Zotero "$HOME/Zotero"
```

Before doing that, check whether `~/Zotero` already exists:

```bash
ls -ld "$HOME/Zotero"
```

If it exists and is not already the right thing, do not blindly overwrite it. Either use `ZOTERO_DB_PATH` instead, or decide deliberately what that directory is supposed to be.

The critical file is:

```text
Zotero/zotero.sqlite
```

That is the SQLite database under the Zotero data directory. As far as I have peeked under the hood, a lot of the local-library behaviour depends on finding that file and then resolving the surrounding attachments, notes, and metadata correctly.

If you prefer an explicit environment variable instead of a symlink, use:

```bash
export ZOTERO_DB_PATH="/mnt/c/Users/<username>/Zotero/zotero.sqlite"
```

For a persistent shell setup, put that in your shell profile. For Codex, I prefer putting only the MCP-specific environment in the MCP config.

## Configure Codex

Codex reads MCP servers from `config.toml`. The user-level file is:

```text
~/.codex/config.toml
```

You can add the server with the Codex CLI:

```bash
codex mcp add zotero --env ZOTERO_LOCAL=true -- zotero-mcp serve --transport stdio
```

Or configure it directly in TOML:

```toml
[mcp_servers.zotero]
command = "zotero-mcp"
args = ["serve", "--transport", "stdio"]

[mcp_servers.zotero.env]
ZOTERO_LOCAL = "true"
```

After restarting Codex, `/mcp` should show the Zotero server. At that point I can ask things like:

```text
Search my Zotero library for papers about adaptive MCMC.
```

or:

```text
Find papers in my library that are relevant to Markovian noise in stochastic approximation.
Give me the Zotero metadata for the most relevant ones.
```

That second kind of question is the reason I care about this. I do not just want the model to answer from general memory. I want it to inspect the bibliography I actually have, use the notes and metadata I have accumulated, and make citation-bearing claims that I can audit.

## Why this helps

This is useful for a few related reasons.

First, it turns my Zotero library into agent-accessible research context. The agent can search across papers I have already curated, rather than treating the public web or its parametric memory as the default source of truth.

Second, it makes citation workflows less bolted-on. If an agent helps draft an experiment description or background paragraph, it can look for candidate citations in the library while it is doing the work. Citations become constraints on the writing, not decoration after the fact.

Third, it gives me a better way to inspect the agent's behaviour at the edge of my knowledge. When I know the literature well, I can catch more mistakes myself. When I do not, I want the agent to show its connection back to concrete Zotero items: title, authors, DOI, notes, citation key, and why it thinks the paper is relevant.

That does not make the agent automatically trustworthy. It makes the workflow more inspectable.

## The downside: more context

The main downside is context bloat.

Every tool added to an agent has a cost. There is the visible token cost of tool schemas and tool responses. There is also a less visible reasoning cost: if a search returns too many loosely relevant papers, the model now has to carry that noise around while still solving the original task.

A Zotero library can be large. Metadata can be verbose. Notes can be uneven. PDFs can be very long. If the agent drags too much of that into the conversation, the setup gets worse rather than better.

So I do not want this to become "dump Zotero into the prompt". The useful version is narrower:

```text
query the library
return a small number of relevant items
include enough metadata to audit the result
only pull full text or notes when needed
```

This is also why I think retrieval is probably the right shape for the next step. A RAG layer over Zotero could reduce context bloat by indexing notes, metadata, and PDF text, then returning only compact, citation-aware matches. The point is not to make the system more fashionable. The point is to make context smaller and more relevant.

Something like this would be much easier for an agent to use responsibly:

```text
title
citation key or DOI
short reason for match
one relevant note or passage
link back to the Zotero item
```

That is the version I want: Zotero-backed context, not Zotero-shaped prompt bloat.

## Sources

- [`54yyyu/zotero-mcp`](https://github.com/54yyyu/zotero-mcp)
- [Zotero local API documentation](https://www.zotero.org/support/dev/web_api/v3/local_api)
- [Microsoft WSL `.wslconfig` settings](https://learn.microsoft.com/en-us/windows/wsl/wsl-config#configuration-settings-for-wslconfig)

You can also read [about me](../../about/) or reach me through the [contact page](../../contact/).
