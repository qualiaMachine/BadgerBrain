# BadgerInfer Quickstart (API key only)

For anyone who's been handed a **BadgerInfer API key** — a hackathon
participant, a lab member, someone building an app. You call the models
over HTTP from your own laptop, notebook, or server. You don't need a
Run:ai account, and you never log into the cluster.

>**Getting a key:** 
>If you don't have a key yet, you can request one through the
>[BadgerInfer access form](https://forms.gle/vkcLzApNrX7KbkTP9). It asks
>which group or project you're with, so your usage lands under the right
>team, and your NetID, so the firewall rule can be updated. The key
>arrives as a **1Password share link** — that's the only way keys go out
>here.
>The link is locked to your `@wisc.edu` address and expires. If it has
>expired or you lose it, request another.

BadgerInfer is UW-Madison's gateway to open-weight models hosted on
campus GPUs. It's at **`https://llm-gw01.doit.wisc.edu/v1`** and speaks the
OpenAI API, so any client that lets you set a base URL works unmodified:
the `openai` Python package, `httr2` in R, LangChain, LlamaIndex, curl,
Postman.

| Model | Type | Availability | Use it for |
|-------|------|-------------|-----------|
| `qwen3.8-27b` | chat | always on | General text: writing, reasoning, code, summarization |
| `churro-3b` | chat + vision | **wakes on demand** | OCR of historical documents and handwriting; send page images |
| `qwen3-vl-embedding-8b` | embeddings | **wakes on demand** | 4096-dim vectors for search / RAG; handles text and images |

**"Wakes on demand" means your first call can take a few minutes.** The
pilot has two GPUs: one is dedicated to `qwen3.8-27b`, and everything
else shares the second and releases it when idle. If you're the first
person to prompt a sleeping model in a while, your request starts it up
and waits — usually a minute or two, sometimes longer. The connection is
held open and nothing is lost; it just sits there.

So: **use a client timeout of 300 seconds** (the examples in the
language guides do) —
a 30- or 60-second default gives up mid-startup and looks like a
failure. A slow first call isn't a fault; if the second comes back
quickly, that was a cold start. Models stay warm for a few minutes after
the last request, so you won't hit this repeatedly during active work,
but it's worth sending a throwaway request a few minutes before a demo.

That's the catalogue as of September 2026. It changes: check this doc for
the latest, or once you're connected, run the command under
[Listing models](#listing-models) to see exactly what your key can call.
Want a model that isn't hosted? Email Chris at
[endemann@wisc.edu](mailto:endemann@wisc.edu).

> **A key gets you these models, not the cluster.** Training and
> fine-tuning belong on
> [CHTC](https://chtc.cs.wisc.edu/uw-research-computing/gpu-jobs), not
> here. Running your own model or getting a GPU workspace on this
> cluster is a separate request, and GPU time isn't guaranteed — see
> [Scope](#scope).

## PowerShell and bash

This document includes commands for both **PowerShell** and **bash/zsh**.

Most MacOS and Linux systems use bash as their default shell. Windows users should
use **PowerShell**, NOT **Git Bash**. The 1Password desktop app is not reliable for 
Git Bash.

## First Time Set-Up: Saving Your Key

>If you don't already have 1Password, you will need to request an
>account and download the desktop app. Request the account at 
><https://profile.wisc.edu/doit-services> (1Password under Eligible Services → Request Activation).
>For instructions on downloading the desktop app, see <https://kb.wisc.edu/security/144301#install>.

Once you have 1Password set up, go to the email you were sent, follow the **1Password share link**,
and save the shared key. For details on this step, see [KB 144574](https://kb.wisc.edu/security/144574).
This makes the key yours. Don't paste the key into a file, a notebook, or a chat
message — it identifies you, and everything you run is recorded against it.

## First Time Set-Up: Installing the 1Password Command-Line Interface (CLI)

The **1Password CLI** reads the key from your vault, ensuring the private
credential never appears in your shell history or code. It's called using the command `op`.

**First**, install the CLI in your shell:

```powershell
# Windows
winget install AgileBits.1Password.CLI
```

```bash
# macOS
brew install 1password-cli
```

Full instructions, including Linux:
<https://developer.1password.com/docs/cli/get-started/>

**Second**, enable the desktop integration. In the **1Password desktop app**,
go to **Settings → Developer → "Integrate with 1Password CLI"**, then quit and reopen the app.
This lets `op` unlock through the app instead of asking for a password.


## Connecting to BadgerInfer

## Step 1 — Log in and connect to the **GlobalProtect** vpn.

This is mandatory even if you're already on campus wifi.
If you don't have the vpn set up yet, find instructions here:
<https://kb.wisc.edu/ns/90370>.

The VPN alone isn't enough: your NetID also has to be added to the
gateway's firewall rule. That's arranged when you request a key, but it's
a manual step with a lead time. If you're on the VPN and still get
`Unable to connect to the remote server`, the rule probably isn't in
place yet — email Chris.

## Step 2 — Log in to 1Password in the shell

Run the following code chunk. The first line will unlock 1Password.
With the desktop integration on it may print nothing and returns immediately —
that's fine. The second line, `op whoami`, is the check to ensure the login
was successful. It should print your account and email; if it says *"account is not signed in"*,
see the fallback below.

```
op signin
op whoami
```

>**If `op` won't authenticate**, open the item in the 1Password app or
>browser extension, copy the key (likely saved under the term "credential"),
>and set the variable for this session only. Replace "copied key" with your
>specific key:

```powershell
$env:OPENAI_API_KEY = "sk-..."      # PowerShell
```

```bash
export OPENAI_API_KEY="sk-..."      # bash / zsh
```

>If you use this fallback, `op run` in Step 4 won't work either. Instead,
>start your coding environment from this same shell (the language guides
>show how) so it inherits the variable.

## Step 3 — Load the key

If you were able to log in to 1Password in your shell, load the key.
(Skip this step if you copy and pasted the key directly.)

Your item's path is determined by how it is saved in your 1Password.
It will start with `op://`, followed by the name of the vault. If you get
1Password through the University of Wisconsin and haven't customized your set-up,
the vault name is likely `Employee`, but you can check by looking at the Vault tab on the left-hand
menu of 1Password. Then give the name of the item, followed by `credential`. A complete
path might read something like: "op://Employee/BadgerInfer/credential". In the following code,
replace `Your Pathway` with your item's actual path.

```powershell
# PowerShell
$env:OPENAI_API_KEY = op read "Your Pathway"
```

```bash
# bash / zsh
export OPENAI_API_KEY=$(op read 'Your Pathway')
```

To confirm the key is set, ask your shell to print the first few characters:

```powershell
$env:OPENAI_API_KEY.Substring(0,6)     # PowerShell
```

```bash
echo ${OPENAI_API_KEY:0:6}             # bash / zsh
```

An error or blank line means it isn't set. Don't put the key in your
code.

## Steps 4–5 — Pick your language

Everything up to here is the same whatever you call the models from.
The last two steps — launching your editor so it inherits the key, and
the code that talks to the models — depend on the language, so they
live in their own guides:

- **[Quickstart in Python](gateway-quickstart-python.md)** — JupyterLab
  and the `openai` package
- **[Quickstart in R](gateway-quickstart-R.md)** — RStudio with
  `ellmer` or `httr2`

Any other OpenAI-compatible client works the same way: point it at
`https://llm-gw01.doit.wisc.edu/v1`, give it `OPENAI_API_KEY`, and set
a 300-second timeout.

# Additional Information

## Listing models

The table at the top is a snapshot. To see exactly what your key can call:

```powershell
# PowerShell
(Invoke-RestMethod https://llm-gw01.doit.wisc.edu/v1/models `
   -Headers @{ Authorization = "Bearer $env:OPENAI_API_KEY" }).data.id
```

```bash
# bash / zsh
curl -s https://llm-gw01.doit.wisc.edu/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

## Use the gateway URL, not model hostnames

You may come across a direct model hostname ending in
`deepthought.doit.wisc.edu`. **Don't use it.** Those answer without a key,
so calls that bypass the gateway don't appear in usage reporting. Use
`https://llm-gw01.doit.wisc.edu/v1` for everything.

## Check your own usage

Your key can query itself:

```powershell
# PowerShell
Invoke-RestMethod https://llm-gw01.doit.wisc.edu/key/info `
  -Headers @{ Authorization = "Bearer $env:OPENAI_API_KEY" }
```

```bash
# bash / zsh
curl -s https://llm-gw01.doit.wisc.edu/key/info \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

That shows your key's limits and what it's spent so far.

## Troubleshooting

| What you see | What it usually means |
|---|---|
| Hang, or `Unable to connect to the remote server` | Not on GlobalProtect, or your NetID isn't in the firewall rule yet — see Step 1 |
| `Malformed API Key ... Ensure Key has 'Bearer ' prefix` | Your key never made it into the header. In PowerShell, check you wrote `$env:OPENAI_API_KEY` and not `$OPENAI_API_KEY`, and that any `$headers` variable was built *after* setting it — it captures the value at assignment. In Python, restart the process after setting the variable |
| `Cannot bind parameter 'Headers'` or `A drive with the name 'https' does not exist` | You ran a bash `curl` command in PowerShell, where `curl` aliases `Invoke-WebRequest`. Use `Invoke-RestMethod`, or `curl.exe` |
| `Invalid proxy key` / 401 | Wrong key, or it expired — ask for a new share link |
| `404 ... Model Group=...` | Model name typo, or it was removed. Re-check `/v1/models` |
| Timeout on the first call | Cold start — raise your client timeout to 300s and retry |
| 429 | Rate limited. Back off and retry; if it's persistent, ask for a higher limit |

Lost your key? Request a replacement through the
[access form](https://forms.gle/vkcLzApNrX7KbkTP9). Anything else:
contact Chris with the model name and the exact error text.

## Scope

A gateway key lets you **call** the models in the catalogue. It doesn't
give you a Run:ai account, a GPU, storage on the cluster, or the ability
to host your own model.

**Training and fine-tuning go to CHTC.** The gateway is for inference.
If you need GPUs to train or fine-tune a model, submit the job to
[CHTC's GPU pool](https://chtc.cs.wisc.edu/uw-research-computing/gpu-jobs)
(HTCondor, 1–4 GPUs per job, no BadgerInfer account needed). The
[GPU Lab](https://chtc.cs.wisc.edu/uw-research-computing/gpu-lab) page
covers what's available and how to request it. If you've fine-tuned a
model on CHTC and want it served through the gateway, ask Chris.

**Cluster access is available on request, not by default.** If you want
to run something the gateway can't do and CHTC doesn't fit, talk to
Chris. The pilot has two RTX Pro 6000s (96 GB each) for at least the next
six months, and the shared endpoints already live on them, so GPU time
for your own workload can't be promised.

**Asking for another model is fine.** It's a config change plus a
pipeline deploy on our side, not a rebuild of anything. A model already
on the cluster can usually be exposed within minutes; one that has to be
downloaded and given its own GPU workload takes longer. Either way, ask
sooner rather than at the moment you need it.

If you do get an account, the
[Cluster Guide](../README.md#cluster-guide) describes what's involved,
starting with [00 Overview](00-overview.md).

This is a **pilot**. Read the
[Usage Policy](usage-policy.md) before putting real data through it —
public data only, and no availability guarantees.
