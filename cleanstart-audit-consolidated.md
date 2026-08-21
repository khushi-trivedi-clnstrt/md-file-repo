# CleanStart Docker Hub audit — consolidated report

**Scope (per Biplab, 20/08 18:12):** review the README documentation file, and registry profile links directly pointing to website pages.
**Primary image:** `cleanstart/python`
**Corroborating image:** `cleanstart/keycloak`
**Run date:** 20 August 2026
**Operator:** Khushi Trivedi
**Out of scope:** test cases (explicitly excluded). CVE scanning, SBOM and signature verification were not requested.

**Target identity — read this first.** The `latest` tag resolved to `sha256:f8b9d255e…` at the start of the session and `sha256:ea66d4c75e…` minutes later. All `cleanstart/python` findings below describe **`ea66d4c75e…`**. The digest has been stable since.

**Headline:** the image is in good shape. The page describing it is not. Of 11 verified findings on `python`, 7 are the documentation contradicting an image that works fine.

| | Count |
|---|---|
| Verified working | 16 |
| Verified not working — `python` | 11 (4 High, 3 Medium, 4 Low) |
| Verified not working — `keycloak` (corroboration) | 9 |
| Identified but unverified | 9 |
| Still open | 7 |

---

# SECTION 1 — What is working well

| # | Area | Check | Result | In plain English |
|---|---|---|---|---|
| **W1** | Links | All 7 documented URLs resolve | All returned HTTP `200`, including LinkedIn | Every website link printed on the Docker Hub page opens a real page. None are dead. |
| **W2** | Runtime | Python interpreter version | **3.14.7**, compiled 20 Aug 2026 | The Python inside is current and well supported — not the outdated 3.9 the page claims. The product is newer than its own documentation. |
| **W3** | Runtime | Entrypoint behaviour | `/usr/bin/python`, arguments pass through correctly | Typing a Python command after the image name works exactly as you'd expect. |
| **W4** | Runtime | Hello World example (`latest-dev`) | Printed `Hello, World!` | The first example on the page works when you run it as written. |
| **W5** | Runtime | Workspace mount example (`latest-dev`) | Directory listing returned correctly | Sharing a folder from your laptop into the container works; the container can read it. |
| **W6** | Runtime | `--user $(id -u):$(id -g)` in the mount example | Worked without permission errors | Files the container writes come out owned by you, not by root. That's the correct pattern and it functions. |
| **W7** | Portability | Multi-platform claim | **Verified.** `linux/amd64` and `linux/arm64/v8` both present in the manifest | The page promises the image works on both Intel servers and Apple/ARM machines. It does. Your Mac picked the right one automatically. |
| **W8** | Size | "Multi-stage builds to minimise image size" | 45.3 MB compressed vs 44.8 MB advertised | The size claim is honest. 45 MB for a complete Python runtime is genuinely small. |
| **W9** | Size | Advertised vs actual | Within 0.5 MB of the registry page figure | The number on the page matches reality. |
| **W10** | Security posture | Default user | Runs as UID **65532**, not root | The image does not run as administrator by default. This is the correct, safe choice — even though it contradicts the documentation (see N4). |
| **W11** | Security posture | `SSL_CERT_FILE` | Set to `/etc/ssl/certs/ca-certificates.crt` | The image knows where its certificate bundle is, so secure web connections from Python should verify properly. *(Set correctly; not functionally tested.)* |
| **W12** | Metadata | `Cleanstart.main.package` label | `python3` — correct | The image correctly names what it contains. Notably better than `keycloak`, which mislabels itself. |
| **W13** | Metadata | Licence label | `PSF-2.0` — correct SPDX identifier for Python | The legal licence is stated, and stated accurately. |
| **W14** | Metadata | Vendor and author labels | Present and correct | The image identifies CleanStart as its publisher. |
| **W15** | Stability | Digest after re-pin | `ea66d4c75e…` held stable across subsequent checks | Once it settled, it stayed settled. Pinning by digest works as a mitigation. |
| **W16** | Catalog | Good patterns exist elsewhere | `metallb-controller` deep-links to its own provenance page: `images.cleanstart.com/images/metallb-controller` | Somebody has already built the better version of this documentation. It just hasn't been applied across the catalog. |

---

# SECTION 2 — What is not working: `cleanstart/python` (verified)

| ID | Sev | Type | Finding | What the README promised vs what the image did — plain English |
|---|---|---|---|---|
| **N1** | High | Provenance | `latest` moved mid-session. Two pulls of the same tag, minutes apart, returned different digests (`f8b9d255e…` → `ea66d4c75e…`) and different layer counts (3 → 5), with zero shared layers | **Does not match.** The page advertises one specific version. You asked for it twice and got two different images. Like ordering the same dish twice and receiving two different meals — you can't state which one you reviewed. |
| **N2** | High | Provenance | No `org.opencontainers.image.version`, `.revision`, `.source` or `.created` labels. The only build timestamp anywhere is inside the Python version banner, visible only by starting the container | **Missing entirely.** The image carries no version stamp, build date, or link to the source it was built from. Combined with N1, if anyone asks next month which build was tested, there is no honest answer. |
| **N3** | High | Doc ≠ image | README documents `PYTHONPATH=/usr/local/lib/python3.9/site-packages`. The variable is **not set**; the interpreter is **3.14.7** at `/usr/bin/python` | **Wrong three ways.** The page names a setting that doesn't exist, points at a folder that isn't there, and states a Python version five majors behind what's actually inside. |
| **N4** | High | Doc ≠ image | README's Kubernetes `securityContext` prescribes `runAsUser: 1000` / `runAsGroup: 1000`. Image runs as **65532** | **Does not match.** Containers run as a numbered user. The page tells customers to configure user 1000; the image is built for 65532. Anyone who copy-pastes the block gets an app that can't read or write its own files. |
| **N5** | High | Broken example | "Application Server" command exits immediately with code `0`, produces no log output. Image has no `Cmd`; the example passes no script. Port 8000 published to a container that is already gone | **Does not match, and fails silently.** The page presents this as how to run an app with a web port open. It printed a container ID — looks like success — then shut down instantly with no error and no output. A customer would assume it worked. |
| **N6** | High | Doc ≠ image | README lists "pip package manager" as a key feature. `pip` is **not installed** in `latest`: `exec: "pip": executable file not found in $PATH` | **Does not match.** The page advertises pip, the standard tool for installing Python software. It isn't there. Anyone building on this image hits a hard failure on their first install command. |
| **N7** | Medium | Undocumented config | `PIP_BREAK_SYSTEM_PACKAGES=1` is baked in, documented nowhere — **and configures a tool that isn't installed** (N6) | **Extra, undisclosed, and pointless.** The image switches off a safety feature that stops installers overwriting Python's core files. But the installer it's configuring isn't present. It's a leftover from an earlier build step. |
| **N8** | Medium | Missing metadata | No `ExposedPorts` declared, while the README documents a port-8000 workflow | **Does not match.** Images are supposed to declare which ports they use so tooling can discover them. This one declares none, so anything that asks "what port does this need?" gets silence — and `docker run -P` does nothing. |
| **N9** | Low | Build integrity | *Corrected finding.* `latest-dev` has fewer layers (3 vs 5) but **more content** (66.1 MB vs 45.3 MB; 281 MB vs 188 MB on disk) | **Partially matches.** A "dev" image should be production plus extra tools, and by size it is — about 20 MB bigger. It's just assembled from fewer, chunkier pieces, suggesting two separate build recipes rather than one built on the other. Worth a question, not a defect. *(An earlier draft read this as "dev is not a superset." The size evidence contradicts that; corrected here.)* |
| **N10** | Low | Doc safety gap | Workspace-mount example uses `$(pwd)` with no guidance on where to run it | **Works as written, but incomplete.** The command shares "whatever folder you're standing in" with the container. Run from your home folder — the natural default — it exposed `.docker`, `.gitconfig`, `.npm`, `.m2`, `.cargo` and shell history. The page never warns about this. |
| **N11** | Low | Convention | Label key `Cleanstart.main.package` uses non-standard capitalisation; OCI convention is lowercase reverse-DNS | **Not mentioned either way.** A cosmetic naming inconsistency in the image's internal labels. Harmless alone; just out of step with industry convention. |

---

# SECTION 3 — What is not working: `cleanstart/keycloak` (verified, corroborating)

Audited separately. Included because it proves which `python` findings are template-wide rather than image-specific.

| ID | Sev | Finding | Matches a `python` finding? |
|---|---|---|---|
| **K1** | High | Image has **no `Cmd`**, so every `docker run` command in the README is a no-op that prints help text and exits | **Yes — same as N5** |
| **K2** | High | "Production Deployment" command combines no-`Cmd` with `--restart unless-stopped`, producing an **infinite restart loop**. The flagship documented command crash-loops | Aggravated version of N5 |
| **K3** | High | Image runs as UID **65532**; README and `securityContext` both prescribe `1000` | **Yes — identical to N4** |
| **K4** | High | Label `Cleanstart.main.package: keycloak-operator`. The Keycloak Operator is a Kubernetes controller — a different artifact from the Keycloak server this image actually contains | Opposite of W12, where `python` gets this right |
| **K5** | High | No `org.opencontainers.image.version`, `.revision`, `.source` or `.created` labels | **Yes — identical to N2** |
| **K6** | Medium | Volume-mount example documents `-v /app:/app`. Keycloak's actual data directory is elsewhere; `/app` is template text with no meaning for this image | Same class as N3 — template values not matched to the image |
| **K7** | Medium | Non-standard install path `/usr/share/java/keycloak` instead of the conventional `/opt/keycloak`, undocumented. Every upstream Keycloak tutorial and volume path is therefore wrong for this image | New |
| **K8** | Medium | No `HEALTHCHECK` declared, despite `KC_HEALTH_ENABLED=true` being baked in | Same class as N8 |
| **K9** | Low | No `ExposedPorts` declared | **Yes — identical to N8** |

**Conclusion from this section:** N2, N4, N5 and N8 appear identically in both images. They are **template defects, not image defects**. Fixing the template generator fixes them across the entire catalog at once.

---

# SECTION 4 — Identified but NOT yet verified

Surfaced from reading the catalog and website. **Each needs confirming before it goes in front of anyone.** These are the items closest to the stated scope — "registry links directly pointing to website pages."

| ID | Sev est. | Finding | Where | Verify by |
|---|---|---|---|---|
| **U1** | Medium | "Community Images:" appears as a label with **no URL after it** — sub-links nest under an empty parent | `python`, `keycloak`, `cadvisor`, `minio` | Read the rendered page |
| **U2** | Medium | **Two different README templates in circulation.** `openldap` uses a different link set — "Container Registry: `cleanstart.com`" and "How-to-Run: `github.com/orgs/cleanstart-containers/`" — while others use `images.cleanstart.com` + `github.com/cleanstart-containers` | `openldap` vs `python` | Open both, diff |
| **U3** | High | **Mislabelled link.** `openldap` labels `https://www.cleanstart.com/` as "Container Registry". That is the marketing homepage, not a registry | `openldap` | Follow the link |
| **U4** | High | **Registry path contradiction.** GitHub repos document `ghcr.io/cleanstart-containers/<name>`; Docker Hub documents `cleanstart/<name>`. Same image, two pull paths, no explanation of which is authoritative | `github.com/cleanstart-containers/nats`, `/openldap` vs Docker Hub | Compare both READMEs |
| **U5** | Medium | **Most images link to the catalog root, not their own page.** The good deep-link pattern exists on `metallb-controller` (W16) but isn't applied to `python` | `metallb-controller` vs `python` | Compare |
| **U6** | High | **Domain contradiction.** `cleanstart.com/knowledge-hub/image-catalog` references `images.cleanstart.dev`, `docs.cleanstart.dev/images`, `github.com/cleanstart/images/issues` and `gcr.io/cleanstart-images/…` — a different domain, different GitHub org and different registry from everything on Docker Hub | knowledge-hub page | Open it; check whether the `.dev` domains resolve at all |
| **U7** | High | **"zero-CVE" vs "near-zero CVEs" vs a badge showing 1.** `cleanstart.com` homepage says zero-CVE; `images.cleanstart.com` says near-zero CVEs; the Docker Hub `python` page badge showed **1**. Three surfaces, three different claims | all three | Screenshot each, dated |
| **U8** | Medium | **FIPS wording inconsistent.** Homepage says "FIPS-compliant"; knowledge-hub says "FIPS 140-3 readiness". Different claims, and neither names a CMVP certificate number | website | Read both |
| **U9** | Low | **Catalog size inconsistent.** Knowledge-hub says "1,200+ images"; internal product-facts flags this as unresolved against a "350+" figure | knowledge-hub | Flag to BD; do not resolve yourself |

**U6, U7 and U8 are the highest-value items for the stated scope** — they are website pages that the registry links to, disagreeing with the registry and with each other.

---

# SECTION 5 — Still open

| Ref | Check | Why it matters |
|---|---|---|
| **O1** | Do the 7 links land where their **labels promise**? Deep-link or catalog root? Consistent across images? | HTTP `200` only proves a URL resolves. Axes 2–4 of the link check are untested — and this is the core of the assignment |
| **O2** | Registry **profile** pages: `hub.docker.com/u/cleanstart`, `gallery.ecr.aws/cleanstart/`, `github.com/cleanstart-containers` | Biplab's wording was "registry profile links". Only image-page links have been checked so far |
| **O3** | Cross-image diff of the Documentation Resources block: `kafka`, `curl`, `mongodb`, `openldap`, `metallb-controller` | Confirms which findings are systemic vs image-specific |
| **O4** | `docker run --rm --entrypoint="" cleanstart/python:latest-dev pip --version` | Decides whether N6 is "pip missing entirely" or "the page documents the wrong tag" — materially different fixes |
| **O5** | `docker run --rm --entrypoint="" cleanstart/python:latest-dev python -c 'import sys;print(sys.version)'` | Do both tags ship the same interpreter version? |
| **O6** | Full README read-through for template leakage and unverifiable adjective copy | Manual. Candidates already noted: "enterprise-ready configurations", "advanced security features", "integrated security scanning" |
| **O7** | Verification of all nine Section 4 items | Highest-value work remaining |

---

# SECTION 6 — Recommended fixes

Ordered by impact per unit of effort. Items 1–5 are template changes that fix every image in the catalog at once.

| # | Fix | Addresses | Scope of effect |
|---|---|---|---|
| 1 | Generate `runAsUser` in the README from the image's actual `User` field instead of hardcoding `1000` | N4, K3 | Entire catalog |
| 2 | Set a `Cmd` on images, or rewrite every `docker run -d` example to pass a real command | N5, K1, K2 | Entire catalog |
| 3 | Add OCI provenance labels at build time: `version`, `revision`, `source`, `created` | N1, N2, K5 | Entire catalog |
| 4 | Generate the environment-variable table from the image, not from a template | N3, N7, K6 | Entire catalog |
| 5 | Deep-link each image to its own catalog page, following the `metallb-controller` pattern | U5, W16 | Entire catalog |
| 6 | Decide on pip: ship it in `latest`, or remove it from the feature list and document `latest-dev` as the tag for installing packages | N6 | `python` |
| 7 | Drop `PIP_BREAK_SYSTEM_PACKAGES=1` from `latest`, since it configures an absent tool | N7 | `python` |
| 8 | State the Python version on the page, generated from the build | N3 | `python` |
| 9 | Declare `EXPOSE` wherever the README documents a port | N8, K9 | Entire catalog |
| 10 | Add a one-line warning to the workspace-mount example about what `$(pwd)` shares | N10 | Entire catalog |
| 11 | Reconcile the `.dev` / `.com` domain split and the ghcr.io / Docker Hub path split | U4, U6 | Website + catalog |
| 12 | Align the CVE claim across homepage, catalog and registry badges | U7 | Website |
| 13 | Ask the build team why dev and prod have inverted layer counts | N9 | Build pipeline |
| 14 | Correct the `keycloak` package label, and document its non-standard install path | K4, K7 | `keycloak` |

---

# Evidence and reproducibility

Every finding above is reproducible against:

```
Image:        cleanstart/python@sha256:ea66d4c75e3388fbe97bb9336f6a8efec4d76520dc65e7f1008fa0795c16d15d
Dev tag:      cleanstart/python@sha256:2cf18ce098d03756bdb3a8ffe19c260193b88398138bb87e9def3eb53b4074a1
Prior digest: cleanstart/python@sha256:f8b9d255e113a3e1b386d018b25aa81128cc9524c52481197ef3c6fb79b770c6
Keycloak:     cleanstart/keycloak@sha256:c3b27a283295c8aa516eae5d4c410e2ecef26133ab19667f2a793dc3b2e12d2b
Date:         20 August 2026
Host:         macOS, arm64
Method:       docker pull / run / inspect / manifest inspect / images; curl -sIL for links
```

No vulnerability scanner was run, so this report makes **no claim about CVE counts** in either image.
