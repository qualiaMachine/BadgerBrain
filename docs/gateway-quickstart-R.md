# BadgerInfer Quickstart — R

This is the R half of the
[BadgerInfer Quickstart](gateway-quickstart.md). Do that page first: it
covers what's hosted, getting a key, saving it in 1Password, connecting
over GlobalProtect, and loading the key into your shell as
`OPENAI_API_KEY` (Steps 1–3). This page picks up from there with
RStudio and `ellmer` or `httr2`. Using Python instead? See the
[Quickstart in Python](gateway-quickstart-python.md).

## Step 4 — Connect to BadgerInfer in your coding environment

This step will vary depending on the environment you're using. This page
covers R in RStudio; many other set-ups are also possible.

**1. In the shell, navigate to where you want your project folder**

**2. Create your rstudio.env text file**

This will hold the path to your security key and pass it to RStudio.
First, create and open a new text file:

```powershell
#Powershell
notepad rstudio.env
```
```bash
#bash / zsh
nano rstudio.env
```

A text editor should open. (If you use a different text editor, replace the command
here as needed.) Enter the following in the text file, replacing "vault" and "your item"
to create the same credential pathway you used above:

```
OPENAI_API_KEY=op://<vault>/<your item>/credential
```
Save and close the file using the drop-down menu and/or text editor shortcuts.

**3. Open RStudio from the console, passing it the .env**

```powershell
# Windows PowerShell — close RStudio first
op run --env-file=.\rstudio.env -- "C:\Program Files\RStudio\rstudio.exe"
```

```bash
# macOS — run the app binary directly; `open -a` would not pass the key through
op run --env-file=./rstudio.env -- /Applications/RStudio.app/Contents/MacOS/RStudio
```

If you set the key directly in
[Step 2](gateway-quickstart.md#step-2--log-in-to-1password-in-the-shell)
instead, skip the `.env` file and
start RStudio from that same shell with the same path, minus the
`op run --env-file=... --` prefix.

## Step 5 — Talking to BadgerInfer models

There are two recommended packages for communicating with remote models in R:

| | Use when |
|---|---|
| **`ellmer`** | Chat. Handles conversation state, streaming, tool calling and structured output. **No embeddings support.** |
| **`httr2`** | Embeddings, or when you want no LLM dependency and full control over the request |


Customize the code below by changing the chat text string, and/or the model name.

### A. ellmer (chat)

```r
install.packages("ellmer") #Exclude this line if ellmer is already installed
library(ellmer)

chat <- chat_openai_compatible(
  base_url = "https://llm-gw01.doit.wisc.edu/v1",
  model    = "qwen3.8-27b",
  api_key  = Sys.getenv("OPENAI_API_KEY")
)

chat$chat("Explain PCA in two sentences.")

# the object keeps the conversation, so follow-ups have context
chat$chat("Now give an example with gene expression data.")
```

Check `?chat_openai_compatible` if an argument name doesn't match —
ellmer changes between releases. `chat_vllm()` is the same thing reading
`VLLM_API_KEY` instead; either works.

### B. httr2 (can handle embeddings)

```r
install.packages("httr2") #Exclude this line of code if httr2 is already installed 
library(httr2)

gateway <- "https://llm-gw01.doit.wisc.edu/v1"
key     <- Sys.getenv("OPENAI_API_KEY")
stopifnot(nzchar(key))   # fails loudly if RStudio didn't inherit it

chat <- function(prompt, model = "qwen3.8-27b") {
  resp <- request(paste0(gateway, "/chat/completions")) |>
    req_auth_bearer_token(key) |>
    req_timeout(300) |>                       # cold starts, see below
    req_body_json(list(
      model    = model,
      messages = list(list(role = "user", content = prompt))
    )) |>
    req_perform() |>
    resp_body_json()

  resp$choices[[1]]$message$content
}

cat(chat("Explain PCA in two sentences."))
```

**Embeddings** in R:

```r
embed <- function(texts, model = "qwen3-vl-embedding-8b") {
  resp <- request(paste0(gateway, "/embeddings")) |>
    req_auth_bearer_token(key) |>
    req_timeout(300) |>
    req_body_json(list(model = model, input = as.list(texts))) |>
    req_perform() |>
    resp_body_json()

  # one row per input text
  do.call(rbind, lapply(resp$data, \(d) unlist(d$embedding)))
}

m <- embed(c("badgers dig burrows", "the mitochondria is the powerhouse"))
dim(m)   # 2 x 4096
```

## Troubleshooting

One error you might see could look like this:

**`Error: nzchar(key) is not TRUE` means `Sys.getenv("OPENAI_API_KEY")`
 came back empty** 
 
 This happens when RStudio doesn't correctly inherit your shell
 environment (Step 4). Your key is fine. To fix it:

```r
 usethis::edit_r_environ()      # opens ~/.Renviron
```

Add one line with the key itself — no quotes, no `export`. (R reads
this file directly, so a 1Password `op://` path won't work here.)

```
OPENAI_API_KEY=sk-...
```

Then **Session → Restart R** — `.Renviron` is read only at startup.
To check that it worked: `nchar(Sys.getenv("OPENAI_API_KEY"))`

`.Renviron` stores the key in plain text. To avoid that, set it for the
current session only and paste the key when prompted:

```r
Sys.setenv(OPENAI_API_KEY = rstudioapi::askForSecret("OPENAI_API_KEY"))
```

Other errors — connection hangs, `Malformed API Key`, 401, 404, cold-start
timeouts, 429 — are in the main quickstart's
[Troubleshooting](gateway-quickstart.md#troubleshooting) table. The
[Scope](gateway-quickstart.md#scope) section there covers what a key
does and doesn't give you.
