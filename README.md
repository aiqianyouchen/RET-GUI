# dirscan GUI User Guide

> A dictionary-based, concurrent web directory scanner — GUI edition (Rust + egui, single-file portable app)

**[CLI Guide](./README.md) | [简体中文 (Simplified Chinese)](./README.gui.zh-CN.md) | GUI English**

------

## Table of Contents

1. [Introduction](#1-introduction)
2. [Interface Overview](#2-interface-overview)
3. [Quick Start](#3-quick-start)
4. [Field Reference](#4-field-reference)
5. [Advanced Options](#5-advanced-options)
6. [Understanding the Live Output](#6-understanding-the-live-output)
7. [Common Scenarios](#7-common-scenarios)
8. [Extension (`%EXT%`) Expansion](#8-extension-ext-expansion)
9. [Error Handling](#9-error-handling)
10. [Differences vs. the CLI Version](#10-differences-vs-the-cli-version)
11. [Building & Running](#11-building--running)
12. [Legal Use Disclaimer](#12-legal-use-disclaimer)
13. [License](#13-license)

------

## 1. Introduction

dirscan GUI is the graphical front-end for the command-line `dirscan.exe`. Both share the same scanning core (the `dirscan` crate) and provide identical capabilities:

| Feature               | Description                                                  |
| --------------------- | ------------------------------------------------------------ |
| Dictionary-driven     | Reads a list of paths from a wordlist file and probes them one by one |
| Async concurrency     | Built on the tokio async runtime; multiple workers pull tasks dynamically with automatic load balancing |
| Real-time coloring    | Every result is color-coded by status class (2xx green / 3xx cyan / 4xx yellow / 5xx red / ERR purple) |
| Status code filtering | "Show only" + "Exclude" — two fields, exclusion takes precedence (same syntax as the CLI) |
| Identity spoofing     | Multi-UA rotation, cookies and HTTP Basic auth               |
| Proxy support         | HTTP / HTTPS / SOCKS5 proxies (multi-proxy rotation, embedded credentials, bypass list) |
| Recursive redirects   | Same-origin redirects are automatically enqueued for scanning, strictly deduplicated |
| `%EXT%` expansion     | A placeholder in the wordlist is expanded into multiple requests based on an extension list (22+ built-in suffixes by default, or a custom external file) |
| Controllable speed    | Tunable via concurrency, timeout and retries                 |

**Extra advantages of the GUI over the CLI:**

- Form-based input — no parameters to memorize;
- "Browse" buttons for wordlist / proxy list / extension files;
- Log area auto-scrolls, and a scan can be stopped at any time;
- Color-coded results are easy to eyeball.

------

## 2. Interface Overview

```text
┌────────────────────────────────────────────────────────────────────┐
│  Scan Configuration                                                │
│  Target URL: [  https://example.com/                               ] │
│  Wordlist:  [  wordlist.txt          ] [ … ]                       │
│            Concurrency: [10]  Timeout: [10]  Retries: [1]          │
│  Show only: [            ]  Exclude: [            ]   ▶ Start  ■ Stop│
│                                                                     │
│  ▼ Advanced Options (User-Agent / Cookie / Auth / Proxy)           │
│    User-Agent:  [                                          ]       │
│    Cookie:      [                                          ]       │
│    Basic auth:  [                                          ]       │
│    Proxy:       [                                          ]       │
│    Proxy list:  [                              ] [ … ]             │
│              Bypass: [                              ]              │
│    Extensions:  [                              ] [ … ]             │
│              %EXT% in the wordlist expands to these; empty = built-in default. │
│                                                                     │
├────────────────────────────────────────────────────────────────────┤
│  Live Output                    [Clear]                            │
│  [i] Target: https://example.com/                                  │
│  [i] Wordlist: wordlist.txt (220 paths)                            │
│  [i] %EXT% expansion: +4752 entries (built-in, 22 suffixes)        │
│  [200] https://example.com/admin [1024B]                           │
│  [404] https://example.com/staff [226B]                            │
│  [302] https://example.com/login -> https://example.com/login/     │
│  …                                                                 │
└────────────────────────────────────────────────────────────────────┘
```

- **Top area**: the configuration form, Start / Stop buttons and a status indicator (idle / scanning…).
- **Advanced Options**: collapsed by default; expand to use UA / Cookie / Auth / Proxy / Extensions.
- **Bottom area**: the live log, auto-anchored to the bottom; the "Clear" button wipes it.

------

## 3. Quick Start

1. Double-click `dirscan-gui.exe` (Windows 64-bit, no dependencies to install).
2. Enter a **Target URL** (e.g. `https://example.com/`).
3. Click the **`…`** button next to the wordlist and pick a dictionary file (e.g. the included `wordlist.txt`).
4. Click **`▶ Start`**.
5. The output area scrolls in real time; the last two lines are the summary statistics when the scan finishes.
6. To abort, click **`■ Stop`** (tasks are cancelled gracefully; already-completed results still count).

------

## 4. Field Reference

| Field       | Default        | Description                                                  |
| ----------- | -------------- | ------------------------------------------------------------ |
| Target URL  | (required)     | Base URL, `http://` or `https://`; trailing `/` optional — the tool normalizes it |
| Wordlist    | (required)     | Path to the dictionary file; one path per line; lines starting with `#` are comments; use the `…` button to browse |
| Concurrency | `10`           | Number of concurrent workers (CLI: `-t`)                     |
| Timeout     | `10` seconds   | Per-request timeout (CLI: `-T`)                              |
| Retries     | `1`            | Retry count on request failure (CLI: `-r`)                   |
| Show only   | (empty = all)  | Show only matching status codes; comma-separated, ranges allowed (CLI: `-f`) |
| Exclude     | (empty = none) | Hide matching status codes; same syntax as "Show only"; combinable, **exclusion wins** (CLI: `-e`) |

**Status-code filter examples:**

| Input         | Effect                                   |
| ------------- | ---------------------------------------- |
| `200`         | Show only 200 OK                         |
| `200,301,403` | Show only these three exact status codes |
| `200-299`     | Show only 2xx                            |
| `200-499`     | Show only 2xx–4xx                        |
| `404,500-599` | In the "Exclude" field: hide 404 and 5xx |

> Filtering / exclusion affects **display only**, never the scan itself: hidden paths are still requested and counted; same-origin redirect targets are still enqueued.

------

## 5. Advanced Options

Expand the "Advanced Options" panel to reveal:

| Field      | Example                                             | Description                                                  |
| ---------- | --------------------------------------------------- | ------------------------------------------------------------ |
| User-Agent | `Mozilla/5.0 ...` or `UA1,UA2,UA3`                  | Empty defaults to `dirscan/0.1`; **comma-separated values rotate across requests** |
| Cookie     | `SESSION=abc` or `SESSION=abc; TOKEN=xyz`           | Semicolons merge values into one Cookie header; useful for carrying login state |
| Basic auth | `user:pass`                                         | When set, every request carries an `Authorization` header    |
| Proxy      | `http://127.0.0.1:8080` or `socks5://10.0.0.2:1080` | Supports `http://` `https://` `socks5://` `socks5h://`; multiple proxies separated by commas (rotated); falls back to environment variables when empty |
| Proxy list | `proxies.txt`                                       | One proxy URL per line, `#` for comments; usable together with the "Proxy" field |
| Bypass     | `localhost,192.168.0.0/16`                          | Proxy bypass list; matching hosts connect directly; defaults to `localhost,127.0.0.1,::1`, also reads the `NO_PROXY` env var |
| Extensions | `ext.txt`                                           | `%EXT%` in the wordlist expands to these extensions; leave empty for the built-in 22 |

> Proxy precedence: form / file **takes precedence over** environment variables (`ALL_PROXY` > `HTTPS_PROXY` > `HTTP_PROXY`). A proxy URL without a scheme is treated as `http://`; credentials are embedded in the URL (`user:pass@host:port`). With multiple proxies and retries > 0, a failed request automatically switches to the next proxy.

------

## 6. Understanding the Live Output

### Live line format

```text
[STATUS] FULL-URL [SIZE] [-> REDIRECT-TARGET (extra note)]
```

- **Size**: the HTTP `Content-Length` header in bytes; shown as `-` when missing.
- **Redirect**: a `->` means a 3xx; if the target is **same-origin** it is automatically enqueued and tagged "(added to scan queue)"; cross-origin targets are only annotated, not enqueued.

### Color reference

| Color  | Status code | Meaning                                                  |
| ------ | ----------- | -------------------------------------------------------- |
| Green  | 2xx         | Path exists — the findings that matter most              |
| Cyan   | 3xx         | Redirect; the target is shown                            |
| Yellow | 4xx         | Client errors (403 / 404 / 405, ...)                     |
| Red    | 5xx         | Server errors                                            |
| Purple | `ERR`       | Request failed (timeout / connection refused / DNS, ...) |

> The GUI output is stripped of ANSI escape sequences at the engine layer — **no stray `[33m` / `[0m` residue** ever appears.

### Final statistics

```text
Scan finished in 1.8s | Total requests: 15 | 2xx found: 3 | Redirects: 2 | Errors: 0
Scan finished in 2.7s | Total requests: 4  | 2xx found: 2 | Redirects: 1 | Errors: 0 | Matching filter shown: 2
Scan finished in 5.5s | Total requests: 2  | 2xx found: 1 | Redirects: 0 | Errors: 0 | Retries: 1
```

- When "Show only" / "Exclude" are used, the summary ends with "Matching filter shown: N" (the **full** counts are unaffected);
- "Retries: N" is the total number of failed-request retries during the scan (a successful retry still counts normally; this does not mean the scan ultimately failed).

------

## 7. Common Scenarios

| Scenario                        | How to fill it in                                            |
| ------------------------------- | ------------------------------------------------------------ |
| Regular scan                    | Target URL + wordlist; leave everything else at defaults     |
| Only 200s, cut 404 noise        | "Show only": `200`                                           |
| Focus on 200 / 301 / 302 / 403  | "Show only": `200,301,302,403`                               |
| Only 2xx and 3xx                | "Show only": `200-299,300-399`                               |
| Inverse-exclude 404             | "Exclude": `404`                                             |
| Combined: 2xx–4xx but drop 403  | "Show only" `200-499` + "Exclude" `403,404` (exclusion wins) |
| Fast intranet scan              | Concurrency `100`, timeout `5`                               |
| Slow, low-profile scan          | Concurrency `5`, timeout `30`                                |
| Browser UA + multi-UA rotation  | Advanced → User-Agent: `Mozilla/5.0 A,Mozilla/5.0 B`         |
| Session-protected paths         | Advanced → Cookie: `SESSION=abc; TOKEN=xyz`                  |
| Basic-auth-protected target     | Advanced → Basic auth: `admin:S3cr3t!`                       |
| HTTP proxy                      | Advanced → Proxy: `127.0.0.1:8080` or `http://user:pass@10.0.0.1:8080` |
| SOCKS5 proxy (e.g. `ssh -D`)    | Advanced → Proxy: `socks5://127.0.0.1:1080`                  |
| Multi-proxy rotation + failover | Advanced → Proxy: `http://10.0.0.1:8080,socks5://10.0.0.2:1080`, Retries `2` |
| Proxy list file                 | Advanced → Proxy list: `proxies.txt`, Bypass: `*.internal.com,10.0.0.0/8` |
| Extension backup sweep          | Use `%EXT%` in the wordlist (see next section); leave Extensions empty to expand the 22 built-in suffixes |

------

## 8. Extension (`%EXT%`) Expansion

When a wordlist entry contains the literal placeholder `%EXT%`, it is automatically expanded into multiple requests before the scan starts.

**Sample wordlist:**

```text
admin.%EXT%
login.%EXT%
config.%EXT%
```

**Default expansion (when the extension file is empty) — 22 built-in suffixes:**

```text
(empty = original path), .html, .htm, .php, .php3, .php4, .php5, .phtml,
.asp, .aspx, .ashx, .asmx, .jsp, .jspx, .jspa, .cer,
.txt, .log, .bak, .old, .orig,
.gz, .tgz, .zip, .tar, .db, .sqlite, .mdb, .swp, .swo
```

**Custom extension file** (`ext.txt`, one per line, `#` for comments, leading dots optional):

```text
# Common web suffixes
.php
.asp
.jsp
.bak
.gz
.zip
```

The startup log prints an expansion notice:

```text
[i] %EXT% expansion: +4752 entries (built-in, 22 suffixes)
```

or

```text
[i] %EXT% expansion: +800 entries (custom, ext.txt)
```

> **Note**: If the placeholder were left unexpanded, the literal `%EXT%` would be sent to the server (`%` is a URL reserved character) and most servers would return a short 400. Both the GUI and the CLI expand first, so this residue never appears.

------

## 9. Error Handling

The GUI distinguishes two kinds of errors:

- **Startup / configuration errors**: after clicking "Start", if the configuration is invalid, a red `[!] ...` line appears at the top of the output area and the scan never begins.
- **Runtime errors**: a failed preflight prints a red `[!] Target URL unreachable ...` and aborts; a single failed request is recorded as a purple `ERR` and does not affect the rest of the scan.

| Message                                                      | Cause                                                        | Suggestion                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `Please enter a target URL`                                  | URL field is empty                                           | Fill in the URL                                              |
| `Please select a wordlist file`                              | Wordlist field is empty                                      | Pick an existing dictionary file                             |
| `Cannot read wordlist file 'xxx': ...`                       | Wordlist path does not exist or is unreadable                | Check the path                                               |
| `Wordlist file 'xxx' is empty or contains only comments`     | No valid entries in the wordlist                             | Check the wordlist content                                   |
| `Cannot read extension file 'xxx': ...`                      | Extension file path is wrong                                 | Fix the path, or leave it empty to use the built-ins         |
| `Extension file 'xxx' is empty or contains only comments`    | No valid lines in the extension file                         | At least one extension per line (e.g. `.php` / `php`)        |
| `Invalid extension 'xxx'`                                    | Extension contains illegal characters                        | Only letters, digits and `. - _`                             |
| `Invalid target URL 'xxx': ...`                              | Malformed URL                                                | Check the spelling; must include `http://` or `https://`     |
| `Invalid status filter item 'xxx'`                           | "Show only" expression has illegal characters or a reversed range (e.g. `999-200`) | Use valid forms like `200`, `200,301`, `400-499`             |
| `Invalid status exclude item 'xxx'`                          | Same issue in the "Exclude" expression                       | Same syntax as "Show only"                                   |
| `Invalid credentials 'xxx'`                                  | Basic auth is missing a colon or has an empty username       | Use `user:pass`                                              |
| `Illegal characters in cookie: ...`                          | Cookie value contains characters illegal in HTTP headers (e.g. CJK text, control chars) | Use valid cookie key-value pairs                             |
| `Unsupported scheme 'ftp'`                                   | Only http/https are supported                                | Use an http/https URL                                        |
| `Unsupported proxy scheme 'xxx'`                             | Invalid proxy protocol prefix                                | Use `http://` `https://` `socks5://` `socks5h://` or omit the prefix |
| `Invalid proxy 'xxx': ...`                                   | Malformed proxy URL (bad port / credentials)                 | Check the URL spelling and port                              |
| `Cannot read proxy file 'xxx': ...`                          | Proxy list file path is wrong                                | Check the file path                                          |
| `Target URL unreachable: (proxy) connection failed: ... (a proxy is configured — please verify it)` | The proxy itself is unreachable (all preflight tests failed) | Confirm the proxy address, port, credentials and that the proxy process is running |
| `Target URL unreachable (connection failed / timeout): ...`  | Preflight failed; target unresponsive                        | Confirm the target is online, reachable and on the right port |
| `[ERR] ... - timeout: ...`                                   | A single request timed out                                   | Raise "Timeout", or ignore the occasional slow path          |
| `[ERR] ... - connection failed: ...`                         | Connection reset / refused                                   | The target may be rate-limiting; lower "Concurrency"         |
| `Concurrency must be ≥ 1`                                    | Concurrency was set to 0 or negative                         | Use `1` or higher                                            |
| `Timeout (seconds) must be ≥ 1`                              | Timeout is not a positive integer                            | Use `1` or higher                                            |

------

## 10. Differences vs. the CLI Version

| Aspect        | CLI (`dirscan.exe`)            | GUI (`dirscan-gui.exe`)                                      |
| ------------- | ------------------------------ | ------------------------------------------------------------ |
| Interface     | Command-line arguments         | Form + file-browse buttons                                   |
| Option names  | `-u / -w / -t / -T / ...`      | Form fields (one-to-one mapping)                             |
| Output        | Color-coded terminal text      | GUI log area (ANSI stripped, colors decided by the engine)   |
| Stopping      | `Ctrl+C`                       | "■ Stop" button                                              |
| Session reuse | Each invocation is independent | The window can be reused for repeated scans; logs accumulate (use "Clear" to wipe) |
| Scripting     | Suited to CI / shell           | Suited to desktop, manual assessment                         |
| Core          | `dirscan` crate                | The same `dirscan` crate (fully equivalent capabilities)     |

------

## 11. Building & Running

### Direct run (recommended)

Download a release, extract it, and double-click:

- Windows 64-bit: `dirscan-gui.exe` (no dependencies to install)

### Build from source

```powershell
# After installing Rust 1.75+ (https://rustup.rs):
git clone https://github.com/<your-username>/dirscan.git
cd dirscan

# GUI binary (pulls in egui/winit dependencies)
cargo build --release --bin dirscan-gui
# Artifact: target\release\dirscan-gui.exe

# Also build the CLI, if needed
cargo build --release --bin dirscan

```

> The GUI requires a graphical environment; use the CLI on headless systems.

------

## 12. Legal Use Disclaimer

This tool is for **authorized security testing** and **inspection of your own assets** only, for example:

- auditing your own websites for exposed paths;
- target assessment within the written scope of a penetration test;
- internal security patrols.

Running directory scans against other people's systems without authorization is illegal; users bear full legal responsibility. Always comply with local laws and use this tool lawfully and ethically.

------

## 13. License

This project is open-sourced under the [MIT License](LICENSE). You are free to use, modify and distribute it, provided the copyright notice is retained. Users are solely responsible for any consequences arising from the use of this tool.
