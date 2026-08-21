# DockerHub Audit for CleanStart Python Image

## Image config
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

---

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

---



