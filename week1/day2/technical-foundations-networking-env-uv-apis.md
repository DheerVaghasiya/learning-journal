# Technical Foundations — Networking, Environments, Command Line & APIs

*Personal notes on the "behind the scenes" stuff every Python/AI project quietly depends on. None of this is glamorous, but every single error message I'll hit later (SSL errors, "command not found", "API key not found") traces back to one of these basics — so worth actually understanding once instead of copy-pasting fixes forever.*

---

## 1. What actually happens when I hit a URL

Every time I run `requests.get("https://openai.com")`, a small chain of events fires off before I see any data:

1. **DNS lookup** — my machine asks a DNS server "what's the actual address behind this name?" and translates `openai.com` into an IP like `104.18.10.10`.
2. **TCP handshake** — my computer opens a connection to that IP.
3. **HTTP request** — it sends a text request that looks roughly like:
   ```
   GET / HTTP/1.1
   Host: openai.com
   ```
4. **Server response** — comes back with headers + a body, e.g. `HTTP/1.1 200 OK`.
5. **Rendering / parsing** — a browser turns that into a page; `requests` just hands me the raw text/JSON.

Analogy that made this click: DNS is the **phonebook** (name → number), TCP is **dialing the call**, and HTTP is the actual **conversation** once someone picks up.

```python
import requests
r = requests.get("https://openai.com")
print(r.text)
```

This is exactly what's happening under the hood every time I call an API — I'm just doing steps 3–4 in Python instead of a browser.

## 2. HTTP vs HTTPS

- **HTTP** — plain text. Anyone snooping the network can read it.
- **HTTPS** — the same thing, wrapped in **TLS encryption**, so it can't be read or tampered with in transit.

Every API I'll use (OpenAI, SendGrid, etc.) requires HTTPS — no exceptions. Good habit: if I ever see `http://` where I expect `https://`, that's a red flag.

## 3. DNS — the internet's phonebook

- Translates domain names → IP addresses.
- My OS checks a local cache first, then asks a DNS server (usually my ISP's, or I can manually point it at Google `8.8.8.8` or Cloudflare `1.1.1.1`).
- When DNS fails I get errors like `Name or service not known` — that's step 1 above breaking, before anything else even gets a chance to run.

## 4. VPNs & Firewalls (just enough to reason about errors)

- **VPN** — routes my traffic through a remote server, encrypts it, hides my IP. Useful for privacy, but it can also quietly rewrite my DNS requests or break certificates if it's misconfigured.
- **Firewall** — filters traffic in/out, can block specific ports, IPs, or even whole apps (like Python or Jupyter). Common on corporate networks.

Takeaway: if something works on my phone hotspot but not on office wifi, it's almost always one of these two.

## 5. SSL/TLS — what it actually is (and what corporate networks do to it)

**What it is:** SSL/TLS is the encryption layer behind HTTPS. To trust a server, my computer checks a **Certificate Authority (CA)** — basically a trusted third party vouching "yes, this server is who it claims to be." Think of it like a passport stamp: the border agent doesn't personally know me, but they trust the authority that issued my passport.

**Where I actually ran into this:** corporate networks (via tools like Zscaler) sit in the middle of every HTTPS connection, decrypt it, inspect it, then re-encrypt it with *their own* certificate. My browser trusts that certificate because the OS was told to trust it — but Python often ships its own separate certificate bundle (`certifi`) that has no idea this corporate cert exists. That's why the exact same URL works fine in Chrome but throws `SSL: CERTIFICATE_VERIFY_FAILED` in a Python script.

**What I learned fixing it (in order of what actually worked):**
1. Disconnect the VPN first — cheapest thing to try, fixes it surprisingly often.
2. `uv --native-tls sync` — tells `uv` to trust the OS certificate store instead of its own bundle. This alone fixed most corporate-network cases for me.
3. Upgrade `certifi` (`uv add certifi`) — sometimes it's just an outdated cert bundle, nothing to do with corporate networks at all.
4. As a last resort, get the corporate root cert from IT and point `SSL_CERT_FILE` at it.
5. `verify=False` exists but I'm treating it as a "never unless truly desperate" option — it just turns off the security check instead of fixing the trust issue.

Good habit for debugging any of this: test the same URL with `curl -v <url>` outside Python first — narrows down whether it's a Python-specific trust issue or the network itself is broken.

## 6. Environment variables — config that lives outside my code

An environment variable is just a key–value pair the OS keeps around that any running program can read. Instead of hardcoding secrets or settings into a script, I store them *outside* the code and read them at runtime.

The most famous one: **`PATH`** — a list of folders the shell searches when I type a command. That's the entire reason typing `python` works — its install folder is listed in `PATH`.

**Setting them:**

| | Mac/Linux | Windows |
|---|---|---|
| Temporary (this session only) | `export MY_VAR=hello` | `set MY_VAR=hello` |
| Read it back | `echo $MY_VAR` | `echo %MY_VAR%` |
| Persistent | add `export MY_VAR=hello` to `~/.zshrc`, then `source ~/.zshrc` | Start → "Environment Variables" → add under User variables |

**Reading them in Python:**
```python
import os
value = os.getenv("FAVORITE_COLOR")   # returns None if it isn't set — no crash
```

**The `.env` file pattern** (this is the one I'll actually use daily): instead of setting variables manually every session, I dump them into a plain-text file called `.env` in my project root:
```
OPENAI_API_KEY=sk-abc123...
DEBUG=True
```
Then load it once at the top of my script:
```python
from dotenv import load_dotenv
load_dotenv(override=True)   # override=True means .env values win over any already-set ones
```

**Non-negotiable rule:** `.env` goes in `.gitignore`, always. It holds secrets (API keys, passwords) and must never get committed to a public repo. If I want to share the *shape* of what's needed with someone else, I commit a `.env.example` with placeholder values instead.

`.env` is a **hidden file** (starts with a dot) — won't show up by default. `ls -a` reveals it on Mac/Linux; enable "hidden items" in File Explorer on Windows.

## 7. Command line basics

The command line is just a text way of doing what I'd normally do by clicking — navigating folders, running programs, managing files. On Windows, **PowerShell** (or the terminal built into Cursor/VS Code) supports most Unix-style commands, so the table below works pretty much everywhere:

| What I want | Command |
|---|---|
| Where am I? | `pwd` |
| List what's here | `ls` (add `-l` for details, `-a` to include hidden files) |
| Move into a folder | `cd foldername` |
| Go up one level | `cd ..` |
| Go home | `cd ~` |
| Read a file | `cat filename.txt` |
| Make a folder | `mkdir my_project` |
| Delete a file | `rm file.txt` |
| Delete a folder + contents | `rm -r folder` (no recycle bin — be careful) |
| Copy | `cp file.txt backup.txt` (`cp -r` for folders) |
| Rename / move | `mv oldname.txt newname.txt` |

A couple of things that actually tripped me up conceptually at first:
- **Hidden files** (`.env`, `.gitignore`) don't show in a plain `ls` — need `ls -a`.
- **Permission denied** usually means one of: the file's open elsewhere, the folder is protected, or I need admin rights (right-click → "Run as Administrator" on Windows).
- Working inside a OneDrive-synced folder can cause weird file-lock errors and save delays — better to keep dev projects in a plain local folder.

## 8. Managing Python packages & environments — and why `uv`

**The core problem this whole category solves:** every Python project needs specific packages (like `requests` or `openai`), often at specific versions, and different projects on the same machine can need *conflicting* versions of the same package. Without isolation, installing one project's dependencies can quietly break another project. This is the "dependency hell" problem, and it's not unique to Python — anyone who's used `npm` for JavaScript has seen the same pattern (a `package.json`/lockfile pinning versions, packages pulled from a central registry).

**The traditional tools:**
- **`pip`** — Python's package installer, pulls packages from **PyPI** (pypi.org), the central public registry — the Python equivalent of npm's registry.
- **`venv` / `virtualenv`** — creates an isolated folder with its own Python + packages, so project A's dependencies never collide with project B's.
- **Conda** and **pyenv** exist too — Conda manages both packages *and* entire Python versions (popular in data science because it also handles non-Python dependencies like C libraries); pyenv just manages switching between Python versions. Worth knowing they exist, but I don't need to go deep on them right now.

**Why `uv` is what I'm actually using:** it's a newer tool that does what `pip` + `venv` + `pyenv` do together, but written in Rust — so it's dramatically faster and handles version resolution much more reliably (fewer "works on my machine" surprises). The habit I'm building:

- `uv add <package>` instead of `pip install <package>` — adds it to the project *and* records it properly.
- `uv run <script>` instead of `python <script>` — runs inside the project's isolated environment automatically, no manual "activate the venv" step.
- `uv sync` — installs everything the project needs to match its lockfile exactly.

The files `uv` manages for me so I don't have to think about them: `pyproject.toml` (what the project depends on), `.python-version` (which Python version), and `uv.lock` (exact locked versions of everything — this is what makes environments reproducible on a different machine).

## 9. APIs — how my code talks to other services

**GET vs POST**, in one line: GET is like *reading* (fetch data), POST is like *writing* (send/create data).

**Endpoint** = a specific URL that represents one action, e.g. `https://api.openai.com/v1/chat/completions`. Think of it as a labeled doorway into one function of a service.

**JSON** is the data format basically every API speaks — lightweight, readable, and Python has a `json` module built in:
```json
{ "name": "Alice", "age": 25, "isStudent": false }
```

**API keys** are how a service identifies me — think of it as a password that's also tied to billing/rate limits. Same rule as any secret: lives in `.env`, never hardcoded, never committed.

**Client libraries** exist so I don't have to hand-build raw HTTP requests + headers + JSON every time — they wrap all that:
```python
# raw way
import requests
requests.post("https://api.sendgrid.com/v3/mail/send", headers={...}, json={...})

# with a client library
import sendgrid
sg = sendgrid.SendGridAPIClient(api_key)
sg.send(message)
```

**The pattern I'll repeat for basically every API I use, from now on:**
1. Sign up on the provider's site → get an API key.
2. Drop the key into `.env`.
3. `uv add <library>` for that service.
4. Call it.

Worked example (OpenAI):
```python
from openai import OpenAI
from dotenv import load_dotenv
import os

load_dotenv()
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What's the capital of France?"}
    ]
)
print(response.choices[0].message.content)
```

---

## Takeaway

None of this is "AI" yet — it's the plumbing underneath. But almost every confusing error I'll hit going forward (auth failures, connection timeouts, "why isn't my key loading") is going to trace back to one of: DNS/network, certificates, an env variable not being set, or an environment mismatch. Worth having debugged these once on purpose instead of panicking the first time they show up mid-project.
