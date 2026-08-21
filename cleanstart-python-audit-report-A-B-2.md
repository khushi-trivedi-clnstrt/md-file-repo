# `cleanstart/python` — command walkthrough & audit report (Parts A–B)

**Run date:** 20 Aug 2026
**Operator:** Khushi Trivedi
**Method:** commands as written in `cleanstart-readme-link-audit.md`, Parts A.2 and B.1–B.2
**Registry page reference digest:** `sha256:f8b9d255e…`

---

# PART 1 — What each command did, and what the output means

## 1.1 First pull

```bash
docker pull cleanstart/python:latest
```
**Does:** downloads the image the `latest` tag currently points to.

```
884340a430a5, 3b67d9fabcc3, a250192799a4     ← 3 layers
Digest: sha256:f8b9d255e113a3e1b386d018b25aa81128cc9524c52481197ef3c6fb79b770c6
```

| Element | Meaning |
|---|---|
| The hex IDs | One filesystem layer each. **3 layers.** |
| `Digest:` | SHA-256 of the image manifest. This is the immutable identifier for exactly this build. |
| "Downloaded newer image" | Nothing matching was in your local cache. |

**Result:** matches the digest shown on the Docker Hub page. Good.

## 1.2 Link resolve loop

```bash
for u in ...; do printf ... "$(curl -o /dev/null -sIL -w '%{http_code}' "$u")"; done
```
**Does:** requests each URL and prints only the final HTTP status.

| Flag | Meaning |
|---|---|
| `-o /dev/null` | Throw away the page body; you only want the status |
| `-s` | Silent — no progress bar |
| `-I` | `HEAD` request: fetch headers only, not the whole page |
| `-L` | Follow redirects, so you get the *final* status, not the 301 |
| `-w '%{http_code}'` | Print the status code |

**Output: all seven returned `200`.**

`200` = the URL resolves and serves a page. It does **not** mean the page is the right one, or that the label describing it is accurate. Axes 2–4 of check A.2 are still unverified.

## 1.3 Second pull — the finding

```bash
docker pull cleanstart/python:latest
```
Same command as 1.1, minutes later, same machine.

```
7390e48adff6, a2aa8d89190d, ae1707b9372b, aa06ab398a75, 99e1c252cd68   ← 5 layers
Digest: sha256:ea66d4c75e3388fbe97bb9336f6a8efec4d76520dc65e7f1008fa0795c16d15d
```

**Meaning:** the `latest` tag moved between your two pulls.

- Different digest → a different image entirely
- **3 layers → 5 layers** → not a package refresh; the build structure changed
- Zero overlapping layer IDs → nothing was reused

**Consequence for this audit:** everything from this point forward describes `ea66d4c75e…`, not the `f8b9d255e…` on the registry page. Your inspect output, the `python-app` container, all of it. This is why the runbook says pin the digest — the failure mode it warns about occurred live.

## 1.4 Dev tag pull

```bash
docker pull cleanstart/python:latest-dev
```
```
365a58960383, fd02645e5d20, a2649aeaa8c8    ← 3 layers
Digest: sha256:2cf18ce098d03756bdb3a8ffe19c260193b88398138bb87e9def3eb53b4074a1
```

**Meaning:** dev has **3** layers; prod has **5**. A `-dev` variant normally *adds* tooling to prod. Fewer layers means these are two separate builds, not one derived from the other. `cleanstart/keycloak` shows the same pattern (dev 3, prod 5).

## 1.5 Hello World

```bash
docker run --rm cleanstart/python:latest-dev -c 'print("Hello, World!")'
```
**Does:** runs the dev image, passing `-c 'print(...)'` as arguments. `--rm` deletes the container on exit.

Because the entrypoint is the Python binary, `-c` is received by Python, not by Docker.

**Output:** `Hello, World!` — **PASS.** Interpreter works, entrypoint is wired correctly.

## 1.6 Workspace mount

```bash
docker run --rm -v $(pwd):/app -w /app --user $(id -u):$(id -g) \
  cleanstart/python:latest-dev -c 'import os; print(os.listdir("."))'
```

| Flag | Meaning |
|---|---|
| `-v $(pwd):/app` | Mount your current directory at `/app` inside the container |
| `-w /app` | Set the working directory to that mount |
| `--user $(id -u):$(id -g)` | Run as *your* UID/GID so written files aren't root-owned |

**Output:** a listing of your home directory — `.claude.json`, `.gitconfig`, `.zsh_history`, `.m2`, `.npm`, `.cargo`, `.docker`, `Documents`, `Downloads`…

**PASS** functionally: the mount works and the container can read it.

**But note what happened.** You ran it from `~`, so `$(pwd)` was your home directory, and the container got read access to your Docker config, git config, shell history, SSH-adjacent dotfiles and credential caches. The README gives this command with no warning about where to run it from. That's a documentation finding, not a bug.

## 1.7 "Application Server" example

```bash
docker run -d --name python-app -p 8000:8000 -v $(pwd):/app -w /app \
  cleanstart/python:latest
```

| Flag | Meaning |
|---|---|
| `-d` | Detached — run in background, print container ID |
| `-p 8000:8000` | Publish container port 8000 to host port 8000 |

**Output:** `a9968d382926…` — a container ID. **This looks like success and is not.**

## 1.8 What that container actually did

```bash
docker ps -a --filter name=python-app --format '{{.Status}}'
docker logs python-app --tail 20
```
**Does:** `ps -a` includes stopped containers; `--format '{{.Status}}'` prints only the status column. `logs` prints stdout/stderr.

```
Exited (0) About a minute ago
```
…and **`docker logs` printed nothing at all.**

| Element | Meaning |
|---|---|
| `Exited` | The container is dead |
| `(0)` | Exit code 0 — a *clean* exit, so no restart policy or alert would flag it |
| Empty logs | It produced no output before dying |

**Why:** entrypoint is `/usr/bin/python`, the image sets no `Cmd`, and the command line supplies no script. Python launched, found no argument, tried to read from stdin, hit EOF immediately (no `-it`), and exited cleanly. Port 8000 was published to a container that was already gone.

The README's "Application Server — Run application with port forwarding" example starts nothing.

## 1.9 Image config

```bash
docker inspect cleanstart/python:latest --format '{{json .Config}}' | jq
```
**Does:** reads the image's stored configuration. `--format '{{json .Config}}'` extracts just the config object; `jq` pretty-prints it.

Element by element:

| Field | Value | What it means | Verdict |
|---|---|---|---|
| `User` | `65532` | The UID the container runs as by default. 65532 is the conventional "nonroot" UID in distroless-style images. | **Contradicts the README**, which prescribes `runAsUser: 1000` |
| `Entrypoint` | `/usr/bin/python` | The binary that always runs; your arguments get appended to it. Note the prefix: **`/usr/bin`**, not `/usr/local`. | Works, but see PYTHONPATH below |
| `Cmd` | *absent* | Default arguments when you supply none. There are none. | **Root cause of 1.8** |
| `ExposedPorts` | *absent* | Ports the image declares. None. | `docker run -P` does nothing; README documents port 8000 |
| `PATH` | standard | Where the shell looks for binaries | Normal |
| `SSL_CERT_FILE` | `/etc/ssl/certs/ca-certificates.crt` | Tells Python where the CA bundle is, so HTTPS verification works | Normal — untested |
| `PIP_BREAK_SYSTEM_PACKAGES=1` | set | Overrides PEP 668. Without it, pip refuses to install into a distro-managed Python. With it, pip will write into system site-packages. | **Undocumented.** A deliberate safety-guard removal that appears nowhere in the README |
| `PYTHONPATH` | **absent** | — | **The README documents `PYTHONPATH=/usr/local/lib/python3.9/site-packages`. It is not set in the image.** Wrong version *and* wrong prefix, for a variable that doesn't exist |
| `Cleanstart.main.package` | `python3` | Vendor label naming the primary package | Correct here (`keycloak` mislabels itself `keycloak-operator`) |
| `image.authors` / `.licenses` / `.vendor` | present | OCI metadata. `PSF-2.0` is the correct Python licence ID | Fine |
| `image.version` / `.revision` / `.source` / `.created` | **absent** | Standard OCI provenance labels: which version, which commit, which repo, built when | **Missing.** Combined with 1.3, there is no way to identify which build you hold |

---

## 1.10 Re-pin, and Python version

```bash
export IMG=$(docker inspect cleanstart/python:latest --format '{{index .RepoDigests 0}}')
echo "$IMG"
```
**Does:** reads back the digest your local `latest` currently resolves to.
**Output:** `ea66d4c75e…` — unchanged since 1.3. The tag has held still since. All findings below describe this digest.

```bash
docker run --rm cleanstart/python:latest -c 'import sys;print(sys.version)'
```
**Does:** asks the interpreter to state its own version.

```
3.14.7 (main, Aug 20 2026, 10:26:10) [GCC 13.2.1 20240309]
```

| Element | Meaning |
|---|---|
| `3.14.7` | The actual Python version inside the image — current, well supported |
| `main` | Built from the `main` branch |
| `Aug 20 2026, 10:26:10` | **The interpreter was compiled today.** This is the only build timestamp available anywhere in the image |
| `[GCC 13.2.1]` | The compiler used |

**Two conclusions.** The interpreter is modern — good, and it kills the earlier worry that this might be an EOL Python. But the README's environment table names `python3.9`, which is **five major versions behind reality**. And note the irony: the compile timestamp is the closest thing to provenance this image has, and you can only get it by running the container. It isn't in any label.

## 1.11 pip

```bash
docker run --rm --entrypoint="" cleanstart/python:latest pip --version
```
**Does:** `--entrypoint=""` clears the default entrypoint so `pip` runs as the command instead of being passed to Python as an argument.

```
exec: "pip": executable file not found in $PATH
```

**Meaning:** pip is not installed in the production image. This isn't a permissions problem or a path quirk — the binary does not exist.

The README lists "pip package manager" among the image's features.

This also recontextualises `PIP_BREAK_SYSTEM_PACKAGES=1` from 1.9: the image sets a configuration variable for a tool it doesn't ship.

## 1.12 Architecture support

```bash
docker manifest inspect cleanstart/python:latest | jq '.manifests[].platform'
```
**Does:** reads the multi-architecture index — the list of per-CPU builds hiding behind one tag — and prints just the platform of each.

```json
{ "architecture": "amd64", "os": "linux" }
{ "architecture": "arm64", "os": "linux", "variant": "v8" }
```

| Element | Meaning |
|---|---|
| `amd64` | Intel/AMD 64-bit — most cloud servers |
| `arm64` / `v8` | ARM 64-bit — Apple Silicon, AWS Graviton |

**PASS.** The README's multi-platform claim is accurate. Your Mac pulled the arm64 build automatically.

## 1.13 Sizes

```bash
docker images cleanstart/python
```
**Does:** lists local images with their sizes.

```
cleanstart/python:latest       ea66d4c75e33   188MB   45.3MB   U
cleanstart/python:latest-dev   2cf18ce098d0   281MB   66.1MB
```

| Column | Meaning |
|---|---|
| `DISK USAGE` | Uncompressed size on your disk |
| `CONTENT SIZE` | Compressed size — what you download, and what the registry reports |
| `U` | In use by a container right now |

The two numbers differing is normal, not a finding: layers ship compressed and unpack larger.

**Two readings.**

Against the registry page (44.8 MB) the 45.3 MB content size is a close match — a different digest, a slightly different build, expected drift. The "minimises image size" claim holds up: 45 MB for a full Python runtime is genuinely small.

**And this corrects finding F7.** Dev is *larger* than prod — 66.1 MB vs 45.3 MB content, 281 MB vs 188 MB on disk — despite having fewer layers. So dev does contain more, packed into fatter layers. The layer-count inversion still says the two tags are built by different recipes, but my earlier inference that dev "isn't a superset" was wrong on the evidence. Downgraded below.

---

# PART 2 — Audit report

## Scope and method

Parts A.2 and B.1–B.2 of the README & link audit, executed 20 Aug 2026 against `cleanstart/python`. Link resolution by `curl -sIL`. Image behaviour by `docker run` / `docker inspect`. No CVE scanning, SBOM or signature verification — out of scope.

**Target ambiguity, unavoidable:** the `latest` tag resolved to `sha256:f8b9d255e…` at the start of the session and `sha256:ea66d4c75e…` a few minutes later. All Part B findings describe `ea66d4c75e…`.

## Findings

| ID | Sev | Type | Finding | What the README promised vs what your image actually did (plain English) |
|----|-----|------|---------|---|
| **F1** | High | Provenance | `latest` moved mid-session — two pulls of the same tag returned different digests and different layer counts (3 → 5) | **Does not match.** The Docker Hub page advertises one specific version of the image. You asked for that same version twice, minutes apart, and got two different images. Like ordering the same dish twice and getting two different meals — you can't tell anyone which one you actually reviewed. |
| **F2** | High | Provenance | No `org.opencontainers.image.version`, `.revision`, `.source` or `.created` labels. Given F1, the image cannot be identified after the fact | **Missing entirely.** The image carries no version stamp, no build date and no link back to the code it was built from. Nothing on the page warns you about this. Combined with F1, if someone asks next month "which build did you test?", there's no honest answer. |
| **F3** | High | Doc ≠ image | README documents `PYTHONPATH=/usr/local/lib/python3.9/site-packages`. The variable is not set in the image; the interpreter is **3.14.7** at `/usr/bin/python`. Wrong version, wrong prefix, and the variable doesn't exist | **Does not match, on three counts.** The README lists a setting called `PYTHONPATH` and gives its value. First, your image has no such setting. Second, it names a folder that doesn't exist here. Third, it says Python 3.9 — your image runs 3.14.7, five major versions newer. The image itself is fine and current; the page describing it is badly out of date. |
| **F4** | High | Doc ≠ image | README's Kubernetes `securityContext` prescribes `runAsUser: 1000` / `runAsGroup: 1000`. Image runs as **65532**. Copy-pasting the documented block will break file permissions | **Does not match.** Every container runs as a numbered user. The README tells customers to configure user number 1000. Your image is actually built for user number 65532. Anyone who follows the instructions gets a mismatch, and the app fails to read or write its own files. |
| **F5** | High | Broken example | "Application Server" command exits immediately, code 0, no output. No `Cmd` in the image and no script argument in the example. Publishes port 8000 to a dead container | **Does not match.** The README presents this as the way to run an application with a web port open. You ran it exactly as written. The container printed an ID (looks like it worked), then shut down instantly and logged nothing. The web port was opened onto something that no longer existed. |
| **F6** | Medium | Undocumented config | `PIP_BREAK_SYSTEM_PACKAGES=1` is baked in and appears nowhere in the docs. It disables PEP 668's guard against pip writing to system site-packages — **and pip is not installed in this image** (F11) | **Extra, undisclosed, and pointless here.** Your image carries a setting the README never mentions. It switches off a safety feature that stops software installers overwriting Python's core files. But the installer it configures isn't in the image at all — so it's a leftover from an earlier build stage that was never cleaned up. |
| **F7** | ~~Medium~~ **Low** | Build integrity | *Corrected.* `latest-dev` has fewer layers (3 vs 5) but **more content** (66.1 MB vs 45.3 MB). Dev does carry more than prod, packed into fatter layers. The layer-count inversion still indicates two different build definitions rather than one derived from the other, but the "not a superset" reading was wrong | **Partially matches.** A "dev" image should be the production one plus developer tools, and by size yours is — it's about 20 MB bigger. It's just assembled from fewer, chunkier pieces. That points to the two being built by separate recipes rather than one on top of the other. Worth a question to the build team; not a defect on its own. |
| **F8** | Medium | Missing metadata | No `ExposedPorts` declared, while the README documents a port-8000 workflow. `docker run -P` and port-discovery tooling get nothing | **Does not match.** The README walks through using port 8000. Images are supposed to declare which ports they use so tools can find them automatically. Yours declares none, so any tool that asks the image "what port do you need?" gets silence. |
| **F9** | Low | Doc safety | Workspace-mount example uses `$(pwd)` with no guidance. Run from `~` — the natural default — it exposes `.docker`, `.gitconfig`, `.npm`, `.m2`, `.cargo` and shell history to the container | **Matches, but incomplete.** The command worked as written — no defect. The problem is what it doesn't say: it shares "whatever folder you're standing in" with the container. You ran it from your home folder, so the container could read your Docker login config, git settings and command history. The README never mentions this. |
| **F10** | Low | Convention | Label key `Cleanstart.main.package` uses non-standard capitalisation; OCI convention is lowercase reverse-DNS | **Matches the README (it isn't mentioned), but breaks convention.** A cosmetic naming difference in the image's internal labels. Harmless on its own; just inconsistent with how the rest of the container industry names these. |
| **F11** | High | Doc ≠ image | README lists "pip package manager" as a key feature. `pip` is **not installed** in `latest` — `exec: "pip": executable file not found in $PATH` | **Does not match.** The page advertises that the image comes with pip, the standard tool for installing Python software. It isn't there. A customer building on this image expecting to install anything will hit a hard failure on their first command. (May exist in `latest-dev` — untested.) |

**F3, F4, F5 and F11 are the ones to lead with** — all four are cases where the page tells a customer something the image contradicts. F4 and F5 are near-certainly catalog-wide, since the README is templated: `keycloak` has the identical `runAsUser: 1000` defect against an actual UID of 65532, and the identical no-`Cmd` crash.

## The whole thing in plain English

Think of the Docker Hub page as the label on a product, and the image as what's actually in the box.

**The label promises things that aren't in the box.** It advertises pip — the standard tool for installing Python software — and pip isn't installed. It names a Python setting called `PYTHONPATH` and gives its value; that setting doesn't exist. It says the image is Python 3.9; it's actually 3.14.7, five versions newer. It tells customers to run the container as user number 1000; the image is built for user 65532. Follow that last one and the app can't read its own files.

**Something is in the box that isn't on the label — and it's useless.** The image switches off a safety feature that stops software installers from overwriting Python's core files. Except the installer it's configuring isn't in the image. It's a leftover setting from an earlier build step.

**One of the printed instructions doesn't work.** The "run an application with a web port" example looks successful: it prints an ID, no error appears. Underneath, the container shut down immediately and produced no output, and the web port was opened onto nothing. It fails silently, which is worse than failing loudly, because a customer would assume it worked.

**And the box changed while you were looking at it.** You downloaded the image, then downloaded it again a few minutes later using the identical command. The second one was a different image. Because the image carries no version stamp or build date inside it, there's no way to prove which of the two you tested — the only build date anywhere is buried in the Python version string, and you have to start the container to see it.

**What went right, and it's a real list.** All seven website links work. Python runs, and it's a current, well-supported version — not the stale one the page claims. Both CPU architectures the page promises are genuinely there. The image is small, at 45 MB compressed, so the "minimises size" claim is honest. Hello World works. The folder-sharing example works. The image correctly identifies itself as a Python package and states the right licence.

**So the pattern is clear:** the image is in decent shape. The page describing it is not. Almost every finding is the documentation being wrong about a product that's fine.

**The bigger point for Biplab:** this README is a template used across the catalog. The `keycloak` image has the exact same user-number error and the exact same silently-failing run command. These aren't Python's problems — they're the template's problems, and fixing the template fixes them everywhere at once.

## Passed

| Check | Result |
|---|---|
| A.2 — 7 documented links resolve | All `200`, including LinkedIn |
| B.1 — Hello World on `latest-dev` | Printed correctly |
| B.1 — Workspace mount on `latest-dev` | Mount, `-w` and `--user` all worked |
| B.2 — Entrypoint | `/usr/bin/python`, behaves as documented |
| B.2 — **Python version currency** | **3.14.7 — current and well supported. Not the EOL 3.9 the README implies** |
| B.2 — **Multi-platform claim** | **Verified: `linux/amd64` and `linux/arm64/v8` both present** |
| B.2 — **Size claim** | **45.3 MB compressed vs 44.8 MB advertised. "Minimises image size" holds up for a full Python runtime** |
| B.2 — `Cleanstart.main.package` label | Correct for this image |
| B.2 — Licence label | `PSF-2.0`, correct for Python |

## Recommendations

1. **Decide on pip, then say so.** Either ship it in `latest`, or remove it from the feature list and document that `latest-dev` is the tag for installing packages. Right now the page promises a tool the image doesn't have.
2. **Stop shipping `runAsUser: 1000` in the README template.** Generate it from the image's actual `User` field. Affects every image using this template.
3. **Set a `Cmd`, or fix the examples.** Either give the image a sensible default, or rewrite every `docker run -d` example to pass a real command.
4. **Add OCI provenance labels** at build time: `version`, `revision`, `source`, `created`. Without them, a moving `latest` tag is unauditable — and right now the only build date in the image is inside the Python version banner.
5. **Generate the environment-variable table from the image**, not from a template. `PYTHONPATH` is documented and absent; `PIP_BREAK_SYSTEM_PACKAGES` is present and undocumented. The table is inverted.
6. **Drop `PIP_BREAK_SYSTEM_PACKAGES=1` from `latest`**, since it configures a tool that isn't installed. If pip is added back, document the variable and justify it.
7. **State the Python version on the page** and generate it from the build. "3.9" against an actual 3.14.7 is the kind of error that costs a customer evaluation.
8. **Declare `EXPOSE`** where the README documents a port.
9. **Ask the build team about the dev/prod layer split** — dev is larger but built from fewer layers, suggesting separate build definitions. Low priority; worth understanding.

---

# PART 3 — Still open

| Ref | Check | Why it matters |
|---|---|---|
| A.2 axes 2–4 | Do the links go where their labels promise? Deep-link or catalog root? Consistent across images? | `200` only proves the URL resolves |
| A.3 | Registry profile pages: `hub.docker.com/u/cleanstart`, `gallery.ecr.aws/cleanstart/`, `github.com/cleanstart-containers` | This is the explicit ask — "registry profile links" |
| A.4 | Cross-image diff: `kafka`, `curl`, `mongodb`, `openldap`, `metallb-controller` | Determines whether findings are systemic |
| **New** | `docker run --rm --entrypoint="" cleanstart/python:latest-dev pip --version` | Does pip exist on the dev tag? Decides whether F11 is "wrong tag documented" or "pip missing entirely" |
| **New** | `docker run --rm --entrypoint="" cleanstart/python:latest-dev python -c 'import sys;print(sys.version)'` | Do both tags ship the same interpreter version? |
| B.3 | Read-through for template leakage and unverifiable copy | Manual, no commands |
| Part C | F1–F9 in the runbook — the website/registry link contradictions | Highest-value items for the stated scope |

**Completed since the last revision:** Python version (3.14.7), pip presence (absent), multi-platform (verified), image sizes (45.3 / 66.1 MB), digest re-pin (stable at `ea66d4c75e…`).

**Before continuing, re-pin.** The tag moved once already:

```bash
export IMG=$(docker inspect cleanstart/python:latest --format '{{index .RepoDigests 0}}')
echo "$IMG"
```
If that returns anything other than `ea66d4c75e…`, it moved again — note the new digest and say so in the report.
