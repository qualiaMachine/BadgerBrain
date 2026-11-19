# BadgerInfer Quickstart — Python

This is the Python half of the
[BadgerInfer Quickstart](gateway-quickstart.md). Do that page first: it
covers what's hosted, getting a key, saving it in 1Password, connecting
over GlobalProtect, and loading the key into your shell as
`OPENAI_API_KEY` (Steps 1–3). This page picks up from there with
JupyterLab and the `openai` package. Using R instead? See the
[Quickstart in R](gateway-quickstart-R.md).

## Step 4 — Connect to BadgerInfer in your coding environment

This step will vary depending on the environment you're using. This page
covers Python in JupyterLab; other configurations are possible as well.

**1. In the shell, navigate to where you want your project folder**


**2. Create your juypter.env text file**

This will hold the path to your security key and pass it to Jupyter.
First, create and open a new text file:

```powershell
#Powershell
notepad jupyter.env
```
```bash
#bash / zsh
nano jupyter.env
```

A text editor should open. (If you use a different text editor, replace the command
here as needed.) Enter the following in the text file, replacing "vault" and "your item"
to create the same credential pathway you used above:

```
OPENAI_API_KEY=op://<vault>/<your item>/credential
```
Save and close the file using the drop-down menu and/or text editor shortcuts.

**3. Open JupyterLab, passing it the .env**

```powershell
# PowerShell
op run --env-file=.\jupyter.env -- jupyter lab
```

```bash
# bash / zsh
op run --env-file=./jupyter.env -- jupyter lab
```
JupyterLab should open and automatically authenticate with 1Password.

If you set the key directly in
[Step 2](gateway-quickstart.md#step-2--log-in-to-1password-in-the-shell)
instead, skip the `.env` file and
start JupyterLab from that same shell with plain `jupyter lab`.

## Step 5 — Talking to BadgerInfer models

Start by (if necessary) installing the openai package, importing it,
and specifying the value for the "client" variable.

```python
%pip install openai  # skip if openai is already installed
```

```python
from openai import OpenAI

client = OpenAI(base_url="https://llm-gw01.doit.wisc.edu/v1", timeout=300)
```

Now you are ready to start querying the BadgerInfer models! Below are a few 
pieces of code that will do that, depending on how you want to communicate
and what data you're working with. The content variables here are examples only;
change them to ask the models what you want to know. Also, change the model variable 
to query the model you're looking for.

**A. Default approach**
```python
resp = client.chat.completions.create(
    model="qwen3.8-27b",
    messages=[
        {"role": "system", "content": "You are a concise assistant."},
        {"role": "user", "content": "Explain PCA in two sentences."},
    ],
)
print(resp.choices[0].message.content)
```

**B. Streaming**, so long answers appear as they're generated:

```python
stream = client.chat.completions.create(
    model="qwen3.8-27b",
    messages=[{"role": "user", "content": "Write a haiku about badgers."}],
    stream=True,
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

**C. Embeddings** — one call, many texts:

```python
texts = ["the mitochondria is the powerhouse", "badgers dig burrows"]
resp = client.embeddings.create(model="qwen3-vl-embedding-8b", input=texts)
vectors = [d.embedding for d in resp.data]
print(len(vectors), "vectors of", len(vectors[0]), "dimensions")
```

**D. Images** (CHURRO for document OCR) — images go inline as base64:

```python
import base64

with open("scan.png", "rb") as f:
    b64 = base64.b64encode(f.read()).decode()

resp = client.chat.completions.create(
    model="churro-3b",
    messages=[{"role": "user", "content": [
        {"type": "text", "text": "Transcribe this page."},
        {"type": "image_url",
         "image_url": {"url": f"data:image/png;base64,{b64}"}},
    ]}],
)
print(resp.choices[0].message.content)
```

## Troubleshooting

Common errors and what they mean are in the main quickstart's
[Troubleshooting](gateway-quickstart.md#troubleshooting) table. The
[Scope](gateway-quickstart.md#scope) section there covers what a key
does and doesn't give you.
