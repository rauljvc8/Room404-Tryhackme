# Room 404 — TryHackMe Writeup

> **Author:** Raul Velasquez ([@rauljvc8](https://github.com/rauljvc8))
> **Platform:** TryHackMe · **Difficulty:** Easy · **Category:** Web / OSINT
> **Date:** October 2026

🇪🇸 *¿Prefieres español? Mira [README.es.md](README.es.md)*

---

## 📌 Summary

Room 404 is a web enumeration challenge. A web application runs on a non-standard
port (**8080**) and exposes its `.git` directory to the public. By dumping the
exposed Git repository, the full application source code is recovered, and a
staging flag left inside a `README.md` file is disclosed.

This is a textbook case of **Sensitive Data Exposure** (OWASP A02:2021 –
Cryptographic/Sensitive Data Exposure) and **exposed version-control metadata**.

| Field | Value |
|---|---|
| Target service | HTTP (Werkzeug/Flask) on port 8080 |
| Vulnerability | Publicly accessible `.git/` directory |
| Impact | Full source-code disclosure + hardcoded flag |
| Tools | `nmap`, `git-dumper`, `grep` |

---

## 🎯 Objectives

- [x] Dump the exposed source code
- [x] Find the flag

---

## 🔍 1. Reconnaissance — Nmap

The engagement started with a full service/version scan against the target.

```bash
nmap -sV -sC 10.67.149.109
```

![Nmap scan](screenshots/01-nmap-scan.png)

Key findings:

| Port | Service | Version |
|---|---|---|
| 22/tcp | SSH | OpenSSH 9.6p1 (Ubuntu) |
| 8080/tcp | HTTP | Werkzeug httpd 3.0.1 (Python 3.12.3) |

The most important result came from Nmap's `http-git` script, which flagged it
directly:

```
| http-git:
|   10.67.149.109:8080/.git/
|     Git repository found!
|     Last commit message: initial Byte Lotus guest platform
```

The site identifies itself as **"Byte Lotus — Stay Noticed"**, a Flask
application. A `.git/` directory being served publicly is the whole ball game.

---

## 📥 2. Dumping the Exposed Source Code

Because the `.git/` folder is reachable over HTTP, the entire repository can be
reconstructed with [`git-dumper`](https://github.com/arthaud/git-dumper).

**Install:**

```bash
pipx install git-dumper
```

**Dump (note: the DIR argument is required):**

```bash
git-dumper http://10.67.149.109:8080/.git ./room404.src
```

![git-dumper running](screenshots/02-git-dumper.png)

> ⚠️ **Lesson learned:** `git-dumper` needs both a URL **and** an output
> directory. Running it with only the URL returns
> `error: the following arguments are required: DIR`.

---

## 🗂️ 3. Reviewing the Recovered Repository

```bash
cd room404.src
ls -la
```

![Repository contents](screenshots/03-ls-repo.png)

Recovered files:

| File | Purpose |
|---|---|
| `app.js` | Front-end script |
| `index.html` | Landing page |
| `README.md` | Internal staging notes |
| `.git/` | Full version-control history |

---

## 🚩 4. Finding the Flag

The `README.md` inside the repository is an internal staging document that was
never meant to ship:

```bash
cat README.md
```

![Flag in README](screenshots/04-readme-flag.png)

```
# Byte Lotus — Guest Experience Platform

Internal staging repository for the guest app and concierge personalization
service. Do not deploy this folder to production.

Staging flag (remove before launch): THM{byt3_l0tus_...}
```

**Flag found** ✅ — hardcoded in the staging README that shipped to production
along with the `.git` directory.

> 💡 Tip for similar rooms: always grep the whole dump for secrets, since the
> flag may live in old commits rather than the current files:
> ```bash
> grep -rniE "flag|thm\{|secret|password|token" .
> git log -p --all | grep -iE "flag|thm\{|secret"
> ```

---

## 🛡️ Remediation

| # | Recommendation |
|---|---|
| 1 | **Never expose `.git/` over HTTP.** Block it at the web server (e.g. `location ~ /\.git { deny all; }` in Nginx). |
| 2 | **Keep secrets out of source control.** Use environment variables or a secrets manager, never hardcode flags/credentials. |
| 3 | **Separate staging from production.** Internal staging notes and repos must not be deployed to public hosts. |
| 4 | **Scan your own perimeter.** Run tools like `nmap --script http-git` against your assets before an attacker does. |
| 5 | **Purge leaked secrets from history.** Removing a secret in a new commit is not enough — rotate it and rewrite history. |

---

## 🧠 Key Takeaways

- A publicly reachable `.git/` directory is one of the most common and highest-
  impact web misconfigurations — it leaks your entire codebase.
- Nmap's default scripts (`-sC`) often surface the vulnerability for you.
- Source-code disclosure frequently leads to hardcoded credentials, API keys,
  and internal endpoints — not just flags.

---

## 🔗 References

- OWASP Top 10 (2021) — A02: Cryptographic Failures / Sensitive Data Exposure
- `git-dumper` — https://github.com/arthaud/git-dumper
- Nmap `http-git` NSE script — https://nmap.org/nsedoc/scripts/http-git.html

---

*Writeup by Raul Velasquez — SOC / GRC portfolio. Flag partially redacted on purpose.*
