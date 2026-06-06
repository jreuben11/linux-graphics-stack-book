# Appendix E: Contributing Quick-Reference Checklists

**Scope.** This appendix is a desk reference for developers who have already read Chapter 32 and need a rapid recall tool when preparing a patch, merge request, or protocol proposal. It distils the contribution workflows for three projects — the Linux kernel DRM subsystem, Mesa, and wayland-protocols — into binary pass/fail checklists that can be copy-pasted into a local file or a GitLab MR description template. A fourth checklist covers the coordination work required when a single feature spans multiple layers of the stack. Each section identifies the automation tooling that enforces the gates, documents the failure modes that most commonly force a re-spin, and names the reviewers and communication channels that are authoritative for each project area.

---

## Table of Contents

- [E.1 Kernel DRM Patch Checklist](#e1-kernel-drm-patch-checklist)
  - [E.1.1 Required Gates](#e11-required-gates)
  - [E.1.2 Automation Tips](#e12-automation-tips)
  - [E.1.3 Common Failure Modes](#e13-common-failure-modes)
  - [E.1.4 Who to Ping for Review](#e14-who-to-ping-for-review)
- [E.2 Mesa Merge Request Checklist](#e2-mesa-merge-request-checklist)
  - [E.2.1 Required Gates](#e21-required-gates)
  - [E.2.2 Automation Tips](#e22-automation-tips)
  - [E.2.3 Common Failure Modes](#e23-common-failure-modes)
  - [E.2.4 Who to Ping for Review](#e24-who-to-ping-for-review)
- [E.3 Wayland Protocol Checklist](#e3-wayland-protocol-checklist)
  - [E.3.1 Required Gates](#e31-required-gates)
  - [E.3.2 Automation Tips](#e32-automation-tips)
  - [E.3.3 Common Failure Modes](#e33-common-failure-modes)
  - [E.3.4 Who to Ping for Review](#e34-who-to-ping-for-review)
- [E.4 Cross-Project Coordination Checklist](#e4-cross-project-coordination-checklist)
- [E.5 Version Verification Footers](#e5-version-verification-footers)
- [Integrations](#integrations)
- [References](#references)

---

## E.1 Kernel DRM Patch Checklist

**Submission target:** `dri-devel@lists.freedesktop.org`; tree targets are `drm-misc-next` (for most drivers and core changes), `drm-intel-next` (for i915/xe), `drm-amd-next` (for amdgpu/amdkfd), or a driver-specific tree as indicated by `scripts/get_maintainer.pl`.

**Merge-window timing.** Feature work must reach `linux-next` by the `-rc6` release of the current cycle. Patches must land in `drm-next` by `-rc7` at the very latest. After `-rc1`, only bugfixes are accepted; new drivers, platform enabling, and new UAPI additions are prohibited until the next merge window.

### E.1.1 Required Gates

Each item is a binary pass/fail gate. A patch that fails any required item will either be NAK'd explicitly or, more likely, silently ignored. Items marked *(stable)* are additionally required for patches targeting stable kernel branches.

- [ ] **`checkpatch.pl` clean.** Run `scripts/checkpatch.pl --strict` and resolve all `ERROR`-level findings. `WARNING`-level findings that are pre-existing or deliberately accepted must be mentioned in the cover letter. `CHECK`-level findings (enabled only by `--strict`) must also be addressed or documented.

  ```bash
  ./scripts/checkpatch.pl --strict -g HEAD
  ```

- [ ] **Subject line format correct.** The format is `subsystem/driver: concise description in imperative mood`. Use the abbreviated subsystem prefix, not the full Kconfig name or filesystem path. Examples of correct and incorrect forms:

  | Correct | Incorrect |
  |---------|-----------|
  | `drm/amdgpu: fix memory leak in amdgpu_bo_create_reserved` | `drivers/gpu/drm/amd/amdgpu: fix memory leak in amdgpu_bo_create_reserved` |
  | `drm/i915: add support for pipe D watermarks` | `i915: add support for pipe D watermarks` |
  | `drm/panel: samsung-s6e3ha2: drop bogus reset delay` | `drm/panel/samsung-s6e3ha2: drop bogus reset delay` |

  Subject lines must be 70–75 characters maximum. Use `git log --oneline` to verify the summary fits on one line without truncation.

- [ ] **`Signed-off-by:` present and matching.** Every patch must carry `Signed-off-by: Your Real Name <email@example.org>`. The email address must match the `From:` header of the submitted message; a mismatch causes `get_maintainer.pl` to emit validation warnings and will cause reviewers to request a respin. Use `git commit -s` or the `b4` toolchain to populate this tag automatically. See `Documentation/process/submitting-patches.rst`, section "Sign your work" for the Developer Certificate of Origin text that the tag certifies.

- [ ] **`Fixes:` tag present** *(required for bug fixes and regressions)*. When a patch corrects a bug introduced by a specific earlier commit, include:

  ```
  Fixes: abcdef123456 ("exact subject line of the broken commit")
  ```

  The commit hash must be at least twelve hex digits. The quoted subject must exactly match `git log --oneline` output for that commit. Use `b4 prep --auto-fixes` to populate this tag from `git log` automatically. The tag is used by the stable kernel team to determine backport candidates and is checked by `scripts/checkpatch.pl`.

- [ ] **`Cc: stable@vger.kernel.org`** *(required for security fixes and data-corruption bugs)*. The tag is placed in the sign-off area of the commit message — not in the email `Cc:` header — so it survives the patch into the kernel history. To limit backport to a specific kernel version series, use the extended form:

  ```
  Cc: <stable@vger.kernel.org> # 6.6.x
  ```

  The criteria for a fix being suitable for the stable tree are defined in `Documentation/process/stable-kernel-rules.rst`. The key requirement is that the fix must already be merged into `linux-next` or Linus's tree before it can be cherry-picked to stable. [Source: stable-kernel-rules.rst](https://docs.kernel.org/process/stable-kernel-rules.html)

- [ ] **`Reported-by:` and `Closes:` tags present** *(required when the patch resolves a publicly reported bug)*. The `Reported-by:` tag acknowledges the person who reported the issue; `Closes:` provides the URL of the bug report so maintainers and the stable team can track resolution:

  ```
  Reported-by: Jane Smith <jsmith@example.org>
  Closes: https://gitlab.freedesktop.org/drm/kernel/issues/1234
  ```

- [ ] **UAPI documentation updated** *(required for any new DRM property, IOCTL, or struct field visible to userspace)*. The `Documentation/gpu/` directory contains the canonical DRM UAPI reference. The documentation policy formalised in 2019 requires that any addition or modification to the UAPI — including new `DRM_IOCTL_*` numbers, new `drm_mode_*` struct fields, and new KMS properties — be accompanied by kerneldoc comments in the corresponding source file and a corresponding RST entry in `Documentation/gpu/drm-kms.rst` or `Documentation/gpu/drm-uapi.rst`.

- [ ] **IGT test written or cited.** For driver changes and UAPI additions, either write a new test in the [IGT GPU Tools](https://gitlab.freedesktop.org/drm/igt-gpu-tools) repository or identify in the cover letter which existing test demonstrably exercises the changed code path. The IGT repository is at `https://gitlab.freedesktop.org/drm/igt-gpu-tools`.

- [ ] **No new compiler warnings.** Build the affected driver with:

  ```bash
  make W=1 C=1 M=drivers/gpu/drm/<driver>/ 2>&1 | grep -E 'warning:|error:'
  ```

  The `W=1` flag enables additional GCC warnings; `C=1` runs the Sparse checker. The build must produce zero new warnings not present on the current `drm-misc-next` HEAD.

- [ ] **Tested on real hardware.** The cover letter (or the patch description for single-patch submissions) must state the GPU model, driver, and kernel version on which the patch was tested. Untested patches — those without a "Tested-on" or "Test hardware:" note — are routinely asked to provide one before review proceeds.

- [ ] **Cover letter present** *(required for series of two or more patches)*. A cover letter (`[PATCH 0/N]`) must summarise the purpose of the series, link to any companion changes in other trees (libdrm, Mesa), and include a changelog for respin versions (`v2:`, `v3:`, etc.) below the `---` separator.

### E.1.2 Automation Tips

**`checkpatch.pl` as a commit-msg hook.** Install the script as a pre-commit hook to catch formatting errors before they reach the mailing list:

```bash
cd /path/to/linux
cat > .git/hooks/commit-msg << 'EOF'
#!/bin/sh
./scripts/checkpatch.pl --strict --no-signoff "$1"
EOF
chmod +x .git/hooks/commit-msg
```

**`b4` for patch series management.** The `b4` tool (`pip install b4`) implements the full kernel email-based patch workflow, including cover letter management, automatic `Fixes:` tag population, signed attestation, and submission via the `lore.kernel.org` web endpoint (bypassing mail server mangling):

```bash
b4 prep -n my-drm-fix           # initialise a patch series
b4 prep --auto-to-cc             # collect Cc addresses from get_maintainer.pl
b4 send                          # send via web endpoint; auto-increments version
```

The web endpoint used for Linux kernel projects is `https://lkml.kernel.org/_b4_submit`, which avoids patch corruption by mail servers. [Source: b4 documentation](https://b4.docs.kernel.org/en/latest/contributor/overview.html)

**`dim` for DRM Intel maintainer workflow.** The `dim` scripts (DRM Intel Maintainer) provide `dim checkpatch` and `dim check-patch-format` for validating patch series against DRM-specific requirements before submission. They are available at `https://cgit.freedesktop.org/drm/dim/`.

**`scripts/get_maintainer.pl`** should always be run on the changed files to identify the correct mailing lists and reviewers:

```bash
./scripts/get_maintainer.pl --norolestats -f drivers/gpu/drm/amdgpu/amdgpu_bo.c
```

### E.1.3 Common Failure Modes

The following are the most frequently cited reasons for first-review respins in `dri-devel` archive threads:

**Missing `Fixes:` tag on a bug fix.** This is the single most common reason for a reviewer to request a respin before providing technical feedback. Even when the commit being fixed is obvious from context, the formal `Fixes:` tag is required because it is parsed programmatically by the stable kernel scripts. The reviewer will ask for it in the first reply, burning a respin with no code review performed.

**Incorrect subject line prefix.** The `drm/amdgpu:` prefix must match the abbreviated form used in the existing history (`git log --oneline drivers/gpu/drm/amdgpu/ | head -20` shows the correct form). Using the full filesystem path (`drivers/gpu/drm/amd/amdgpu:`) or the Kconfig symbol (`CONFIG_DRM_AMDGPU:`) will trigger an immediate correction request.

**`Signed-off-by` email does not match `From:`.** When a developer submits patches from a corporate email but has configured `git config user.email` with a personal address, the mismatch causes `get_maintainer.pl` to emit `Signed-off-by mismatch` warnings. The fix is to ensure `git config user.email` matches the email used to subscribe to `dri-devel`.

**Multi-patch series without a cover letter.** Any series with two or more patches requires a `[PATCH 0/N]` cover letter. Sending a four-patch series as `[PATCH 1/4]`...`[PATCH 4/4]` without a `[PATCH 0/4]` will result in an explicit request for one. The cover letter must also contain a respin changelog for subsequent versions.

**UAPI change without `Documentation/gpu/` update.** Since 2019, any addition to the DRM UAPI must be accompanied by a documentation update. Patches adding a new `DRM_IOCTL_*`, a new KMS property, or a new field in a UAPI struct that lack the corresponding RST documentation are hard-rejected. The kernel documentation policy is described in `Documentation/doc-guide/kernel-doc.rst`.

**No `W=1 C=1` build performed.** Patches that introduce Sparse warnings or GCC `-W1` warnings are frequently asked to resolve them before technical review begins. Run `make W=1 C=1 M=drivers/gpu/drm/<driver>/` before submission.

### E.1.4 Who to Ping for Review

**Authoritative source:** Always run `scripts/get_maintainer.pl` first. Its output is definitive for which lists and individuals to add.

| Subsystem | Maintainers | Channel |
|-----------|------------|---------|
| `drm-misc` (core, helpers, new small drivers) | Thomas Zimmermann, Jani Nikula | `#dri-devel` on irc.oftc.net |
| `drm/amdgpu`, `drm/amdkfd` | Alex Deucher (`agd5f`), Christian König (`ckoenig`) | `#amd-gfx` on irc.oftc.net; `amd-gfx@lists.freedesktop.org` |
| `drm/i915` | Rodrigo Vivi, Matt Roper, Tvrtko Ursulin | `#intel-gfx` on irc.oftc.net; `intel-gfx@lists.freedesktop.org` |
| `drm/xe` | Rodrigo Vivi, Thomas Hellström | `#intel-xe` on irc.oftc.net |
| `drm/nouveau` | Ben Skeggs, Lyude Paul | `dri-devel@lists.freedesktop.org` |
| `drm/msm` | Rob Clark, Abhinav Kumar | `freedreno@lists.freedesktop.org` |
| `drm/etnaviv` | Lucas Stach | `dri-devel@lists.freedesktop.org` |
| KMS core | Daniel Vetter (emeritus, advisory), Maarten Lankhorst | `dri-devel@lists.freedesktop.org` |

*Note: Maintainer assignments change with every kernel cycle. Verify against the `MAINTAINERS` file in the kernel tree before each submission.*

---

## E.2 Mesa Merge Request Checklist

**Submission target:** GitLab merge request at `https://gitlab.freedesktop.org/mesa/mesa`. Email patches are not accepted. Submit from a personal fork, not from a branch on the main repository. Mesa uses `Marge`, an automated merge bot (`@marge-bot`), to perform the final merge after approval; do not use the GitLab "Merge" button.

### E.2.1 Required Gates

- [ ] **CI pipeline green.** All mandatory CI jobs must pass before a MR can be assigned to `@marge-bot`. The CI pipeline runs `meson`-based builds across multiple configurations (x86-64, ARM, RISC-V cross-compile), static analysis (Mesa's custom `glsl` type-checker, Clang static analyzer), and driver-specific test suites on physical hardware farms. A failing CI job in a tier unrelated to the change (for example, an ARM cross-compilation failure in an x86-only commit) must still be triaged: either fix it or document in the MR description that the failure is pre-existing and unrelated.

- [ ] **`shader-db` run neutral or improved** *(required for changes to any NIR pass, GLSL compiler, or backend code generator)*. Run Mesa's shader-db tool against a representative corpus and paste the summary into the MR description. A single-line summary of shader count changes, instruction count deltas, and register pressure changes is the minimum required:

  ```bash
  # Fast approximate run (~2 minutes)
  SHADER_COUNT=1 ./bin/run_shader_db -p ./shaders/ -r results/
  # Full corpus run (~30 minutes)
  ./bin/run_shader_db -p ./shaders/ -r results/
  ```

  Reviewers will not approve compiler MRs without shader-db evidence. This requirement was formalised in 2022 and applies to NIR passes, GLSL compiler changes, and any hardware backend (RADV, ANV, Turnip, etc.) that affects instruction selection or register allocation.

- [ ] **`dEQP` and/or `piglit` regression-free** *(required for driver and state-tracker changes)*. Run the relevant conformance suite against the affected driver on the target hardware and paste a pass/fail summary in the MR description. CI hardware farm results are acceptable as evidence if the farm exercises the changed code path. For changes in `src/mesa/main/` (the GL state tracker), run `piglit`. For Vulkan driver changes, run `dEQP-VK.*` for the affected feature area.

- [ ] **Commit prefix format correct.** The first line of each commit message must start with the component name followed by a colon:

  | Component | Example prefix |
  |-----------|---------------|
  | RADV (Radeon Vulkan) | `radv: fix descriptor buffer alignment on GFX11` |
  | ANV (Intel Vulkan) | `anv: bump descriptor set limit for BDW` |
  | Turnip (Qualcomm Vulkan) | `tu: fix depth stencil attachment layout` |
  | NIR compiler | `nir: add lowering pass for `fsin`` |
  | Zink | `zink: handle multiview rendering for resolve ops` |
  | Gallium helpers | `util: fix hash_table_foreach edge case` |
  | freedreno | `freedreno/a6xx: fix occlusion query on 8gen` |

  The prefix is case-sensitive. Mixing `RADV:` and `radv:` in the same series will produce reviewer feedback requesting consistency. See `docs/submittingpatches.rst` in the Mesa repository for the full list of canonical prefixes. [Source: Mesa submitting patches docs](https://docs.mesa3d.org/submittingpatches.html)

- [ ] **No new compiler warnings.** The build must be `-Wall -Wextra` clean for the affected component. Run `ninja -C build 2>&1 | grep -E ': warning:'` after a full build to verify.

- [ ] **Copyright header in any new file.** New `.c`, `.h`, `.py`, and `.comp` files must carry a copyright header in the first line or two, followed by an SPDX identifier appropriate for the component. Mesa uses a mix of MIT and LGPL-2.1-or-later depending on the component; consult existing files in the same directory for the correct licence:

  ```c
  /* Copyright © 2025 Author Name <author@example.org>
   * SPDX-License-Identifier: MIT
   */
  ```

  Missing copyright headers trigger a failure in the REUSE compliance CI lint job, which was added to Mesa's pipeline and has been a hard CI gate since Mesa 24.x.

- [ ] **New environment variable documented in `docs/envvars.rst`.** Any new `MESA_`, `RADV_`, `ANV_`, `TU_`, `ZINK_`, `NOUVEAU_`, or similar variable that affects runtime behaviour must be documented in `docs/envvars.rst` before the MR lands. This policy was introduced around Mesa 23.1; reviewers will request the documentation entry if it is absent, and CI may flag undocumented variables depending on the component. For variables with driver-specific debug semantics (e.g. `RADV_DEBUG=syncobj`), document the flag values as well as the variable name. [Source: Mesa envvars docs](https://docs.mesa3d.org/envvars.html)

- [ ] **New `driconf` option documented.** Any new option added to Mesa's `driconf` configuration system must be documented in `docs/driconf.rst` and declared in `src/util/driconf.h` with the appropriate type and default value.

- [ ] **`~gitlab::needs_review` label applied when ready.** After CI is green and the MR is ready for maintainer attention, apply the `~gitlab::needs_review` label. Do not apply this label while CI is still running, as it signals to maintainers that the MR is ready to land.

- [ ] **AI assistance disclosed** *(policy introduced 2024)*. If AI tools (GitHub Copilot, Claude, GPT-4, etc.) assisted with code generation in a non-trivial way, include an `Assisted-by:` or `Generated-by:` tag in the commit message. Trivial and mechanical changes (renaming, formatting) are exempted.

### E.2.2 Automation Tips

**Mesa pre-commit hooks.** Install Mesa's pre-commit hook configuration to run `clang-format`, `meson format`, REUSE compliance, and copyright header checks automatically before every commit:

```bash
pip install pre-commit
cd /path/to/mesa
pre-commit install
```

This was added to the Mesa repository in 2024 and catches the majority of formatting and licence compliance failures before they reach CI. [Source: Mesa pre-commit configuration](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/.pre-commit-config.yaml)

**Shader-db fast path.** For a quick sanity check during development, the `SHADER_COUNT=1` environment variable limits the shader-db run to one shader per program, giving a proportionally scaled result in roughly one-tenth of the full corpus time:

```bash
export MESA_GLSL_CACHE_DISABLE=true
SHADER_COUNT=1 ./bin/run_shader_db -p ./shaders/ -r results_before/
# make your change
SHADER_COUNT=1 ./bin/run_shader_db -p ./shaders/ -r results_after/
./bin/diff_shader_db results_before/ results_after/
```

**Marge-bot assignment.** To request a merge, assign the MR to `@marge-bot`. Marge will rebase onto the current `main`, re-run CI, and merge if all jobs pass. Do not assign to Marge while CI is failing; it will reject the assignment immediately.

**`./bin/pick-ui`** identifies which commits are candidates for stable branch cherry-picks. Run it on your MR commits after landing to see if the stable team has nominated them.

**`scripts/get_reviewer.py`** (Mesa's equivalent of the kernel's `get_maintainer.pl`) suggests reviewers based on file history:

```bash
python3 ./scripts/get_reviewer.py src/amd/vulkan/radv_device.c
```

### E.2.3 Common Failure Modes

**CI failure in an unrelated job.** Mesa's CI infrastructure runs jobs for every supported architecture and driver. An x86-only NIR change that breaks an ARM cross-compilation job will still block the MR. The contributor is responsible for triaging every red CI job; a comment explaining why the failure is pre-existing and unrelated — ideally with a link to an earlier MR that introduced the failure — is required before maintainers will approve.

**Missing `shader-db` result for a compiler change.** Reviewers will not approve NIR pass additions or backend changes without shader-db evidence. The absence of a `shader-db` summary in the MR description is the first thing compiler reviewers check. Even a "no change" result must be explicitly stated; it tells reviewers the change is not a performance regression.

**Commit message missing the component prefix.** The missing prefix breaks `git log --oneline` filtering and makes it harder to identify which commits affect which component in regression bisects. It is the first non-code comment that reviewers make.

**New environment variable without `docs/envvars.rst` entry.** This is a hard policy introduced around Mesa 23.1. Reviewers will request the entry if it is absent, and CI may also flag it depending on the driver component. Ensure any new `MESA_`, `RADV_`, `ANV_`, `TU_`, or similar variable is documented in `docs/envvars.rst` before requesting review.

**Copyright header missing from new `.c`/`.h` file.** The REUSE compliance CI check runs `reuse lint` on every MR. Any file without a recognised copyright header or SPDX identifier fails the check. This is a hard CI gate.

**Wrong licence for the component.** Mesa uses MIT for code in `src/util/`, `src/gallium/`, and most Vulkan drivers, and LGPL-2.1-or-later for the OpenGL API headers and some state tracker code. Using the wrong SPDX identifier will also trigger a REUSE lint failure. Check existing files in the same directory.

**Stable branch patch too large.** Mesa's stable branch policy restricts cherry-picks to patches under 100 lines. A driver fix that exceeds this limit will be declined for stable even if it is otherwise correct.

### E.2.4 Who to Ping for Review

| Component | Reviewers | Channel |
|-----------|----------|---------|
| NIR / compiler infrastructure | Connor Abbott (`cwabbott0`), Jason Ekstrand (`jekstrand`) | `#dri-devel` on irc.oftc.net |
| RADV (Radeon Vulkan) | Samuel Pitoiset (`hakzsam`), Rhys Perry | `#radv` on irc.oftc.net |
| ANV (Intel Vulkan) | Faith Ekstrand, Lionel Landwerlin | `#intel-3d` on irc.oftc.net |
| Turnip (Qualcomm Vulkan) | Danylo Piliaiev, Connor Abbott | `freedreno@lists.freedesktop.org` |
| Zink (GL over Vulkan) | Mike Blumenkrantz (`zmike`) | `#zink` on irc.oftc.net |
| Gallium / state tracker | Roland Scheidegger, Marek Olšák | `dri-devel@lists.freedesktop.org` |
| llvmpipe / softpipe | Roland Scheidegger | `dri-devel@lists.freedesktop.org` |
| freedreno (Qualcomm open source) | Rob Clark, Chia-I Wu | `freedreno@lists.freedesktop.org` |
| Panfrost (Mali) | Alyssa Rosenzweig, Boris Brezillon | `#panfrost` on irc.oftc.net |

*Note: Maintainer assignments are recorded in `src/*/meson.build` maintainer entries and GitLab project member lists. Verify before each submission.*

---

## E.3 Wayland Protocol Checklist

**Submission target:** GitLab merge request at `https://gitlab.freedesktop.org/wayland/wayland-protocols`. New protocols must also be discussed on the `wayland-devel@lists.freedesktop.org` mailing list before or concurrent with the MR, to surface concerns from compositor authors who do not monitor GitLab.

**Protocol lifecycle.** As of wayland-protocols 1.21 (2021), new protocols enter the `staging/` directory. The older `unstable/` path is deprecated; no new protocols should use it. A protocol may be promoted to `stable/` only after it has been proven adequate in production, the GOVERNANCE section 2.3 requirements have been met (including a 30-day minimum discussion period), and the implementation count thresholds have been satisfied. [Source: wayland-protocols GOVERNANCE.md](https://chromium.googlesource.com/external/anongit.freedesktop.org/git/wayland/wayland-protocols/+/HEAD/GOVERNANCE.md)

**Namespace and implementation requirements (from GOVERNANCE section 2.3):**
- `xdg` and `wp` namespaces: at least 3 open-source implementations (either 1 client + 2 servers, or 2 clients + 1 server) before inclusion in `staging/`
- `ext` namespace: at least 1 open-source client and 1 open-source server implementation
- Graduation to `stable/`: subject to the standard 30-day discussion period and no outstanding objections from project members

### E.3.1 Required Gates

- [ ] **Reference implementation for both compositor and client sides.** A protocol proposal submitted without a reference implementation will not advance through review. The conventional minimum is: a prototype compositor patch (a wlroots patch is the most widely used reference) and a working test client that exercises every request and event in the protocol. The implementation does not need to be production-quality but must be sufficient for reviewers to assess correctness and usability.

- [ ] **Correct lifecycle stage and directory.** New protocols must be submitted to `staging/` (not `unstable/`, which is deprecated, and not directly to `stable/`). The directory path format is `staging/<protocol-name>/<protocol-name>-v<N>.xml`. The version number starts at 1 and is incremented for backward-incompatible revisions during the staging period.

- [ ] **All XML descriptions filled in.** Every `<interface>`, `<request>`, `<event>`, and `<enum>` element in the protocol XML must carry both a `summary` attribute (one-line description) and a full-text `<description>` child element. Incomplete descriptions are rejected in the first review pass.

  ```xml
  <request name="set_maximized">
    <description summary="make the surface maximized">
      Requests that the surface be maximized. The client should
      not assume that the surface will actually be maximized after
      this request is processed.
    </description>
  </request>
  ```

- [ ] **Security considerations section present.** The MR description or the protocol XML `<description>` must include a substantive analysis of the protocol's threat model: what can a malicious client do with this protocol? what can a malicious compositor do? what prevents abuse? A one-line "no security implications" statement will be challenged by reviewers; the analysis must address the specific capabilities the protocol grants. This requirement was formalised in 2023 following the `xdg-activation` security review post-mortem.

- [ ] **Rationale document in the MR description.** The MR description must answer: why does this protocol need to exist? What user-visible problem does it solve? What alternatives were considered (Wayland core extensions, X11 mechanisms, D-Bus protocols) and why were they rejected? Without a rationale, reviewers cannot evaluate whether the protocol's design is appropriate for its stated purpose.

- [ ] **Implementation count satisfied before requesting `staging/` inclusion** *(for `xdg`/`wp` namespaces)*. For protocols in the `xdg` and `wp` namespaces, at least 3 open-source implementations (1 client + 2 servers, or 2 clients + 1 server) must exist before the protocol can be added to `staging/`. If the count is not yet met, the `Needs implementations` label is applied and the MR remains open until the count is reached.

- [ ] **30-day discussion period completed before `stable/` graduation.** A proposal to declare a protocol stable requires a 30-day minimum discussion period on `wayland-devel` and/or the GitLab MR. This period allows compositor authors and application developers not actively watching GitLab to raise concerns.

- [ ] **`wayland-devel` mailing list discussion initiated.** Because many compositor authors and protocol reviewers do not monitor GitLab MRs, the `wayland-devel@lists.freedesktop.org` mailing list must be notified of a new protocol proposal. The convention is to send a brief "RFC: new protocol for X" message linking to the MR when the MR is opened.

- [ ] **No absolute screen coordinates.** Protocol designs that use absolute screen coordinates (in compositor pixel space) rather than surface-relative coordinates or output-relative logical coordinates are the most common design-level objection. Reviewers will request a redesign if the coordinate system embeds assumptions about compositor geometry.

- [ ] **Namespace collision check performed.** Before submission, verify that the protocol name and namespace do not collide with existing protocols in `stable/`, `staging/`, and `deprecated/`. Check the canonical list at `https://wayland.app/protocols/`.

### E.3.2 Automation Tips

**`wayland-scanner` well-formedness check.** Run `wayland-scanner` against the XML to verify it parses correctly before review:

```bash
wayland-scanner client-header staging/new-protocol/new-protocol-v1.xml /dev/null
wayland-scanner server-header staging/new-protocol/new-protocol-v1.xml /dev/null
wayland-scanner private-code staging/new-protocol/new-protocol-v1.xml /dev/null
```

Any of these commands failing indicates a structural or namespace error in the XML that will also cause the wayland-protocols CI pipeline to fail.

**`xmllint` schema validation.** The wayland-protocols repository ships an XML schema (`protocol.xsd`) that can be used to catch structural errors beyond what `wayland-scanner` checks:

```bash
xmllint --noout --schema protocol.xsd staging/new-protocol/new-protocol-v1.xml
```

**CI pipeline observation.** The wayland-protocols GitLab CI runs `wayland-scanner` on every XML file in the repository. Push the MR branch and observe the CI pipeline output before requesting review; XML parse errors visible in CI are the first thing reviewers check.

**Protocol catalogue.** Use `https://wayland.app/protocols/` to browse all existing protocols with their current lifecycle stage, namespace, and description. This is the fastest way to perform a namespace collision check.

### E.3.3 Common Failure Modes

**No reference implementation.** This is the single most common reason protocol proposals stall for months. Reviewers cannot evaluate correctness without seeing the protocol in use, and the implementation count requirements cannot be satisfied without code. Submit the wlroots compositor patch and a test client as `Related MR:` links in the description when the main protocol MR is opened.

**Perfunctory security analysis.** Simon Ser and Pekka Paalanen, the most active protocol reviewers on GitLab, will reject a security section that merely says "clients cannot abuse this" without explaining why. The analysis must explicitly enumerate what capabilities the protocol grants, what compositors can verify, and what a sandboxed client cannot do. Reference the `security-context-v1` protocol as an example of the expected depth.

**Attempting to go directly to `stable/`.** Every new protocol must spend time in `staging/` with sufficient implementation coverage before stability can be declared. Submitting a MR targeting the `stable/` directory for a brand-new protocol will be closed immediately with a request to re-target `staging/`.

**Enum values or argument types encoding compositor internals.** The most common design-level objection is the use of absolute screen coordinates, raw buffer formats that encode display hardware assumptions, or seat serial numbers beyond their intended scope. These assumptions make protocols fragile across different compositor architectures. Reviewers will request a redesign before the XML is accepted.

**Protocol name collision.** The `xdg-`, `wl-`, `zwp-`, `wp-`, and `ext-` namespaces each have precedent and conventions. A new protocol whose name conflicts with an existing one (including deprecated protocols) will be asked to rename. The `wayland.app` catalogue is the definitive reference.

**Missing mailing list notification.** Some compositor authors only track `wayland-devel` and not GitLab. A protocol that accumulates GitLab approvals without a mailing list discussion may face late objections from compositor authors during the 30-day stable graduation period. Send the mailing list notification when the MR opens, not when requesting stable graduation.

### E.3.4 Who to Ping for Review

| Topic | Reviewers | Channel |
|-------|----------|---------|
| General protocol review | Simon Ser (`emersion`), Pekka Paalanen (`ppaalanen`) | GitLab MR; `wayland-devel@lists.freedesktop.org` |
| Display / HDR protocols | Sebastian Wick, Xaver Hugl (KWin) | `#wayland` on irc.oftc.net |
| Input protocols | Carlos Garnacho (`carlosg`), Derek Foreman | `wayland-devel@lists.freedesktop.org` |
| Compositor security / sandboxing | Simon Ser, Ryan Walklin | `wayland-devel@lists.freedesktop.org` |
| wlroots compositor implementation | Kenny Levinsen, Simon Ser | `#sway-devel` on irc.libera.chat |
| KWin compositor implementation | Xaver Hugl, Vlad Zahorodnii | `#kde-plasma` on irc.libera.chat |
| Mutter compositor implementation | Carlos Garnacho, Jonas Ådahl | `wayland-devel@lists.freedesktop.org` |

---

## E.4 Cross-Project Coordination Checklist

Many significant features — new display hardware capabilities, GPU memory management improvements, new synchronisation mechanisms — require coordinated changes across the kernel, libdrm, Mesa, and possibly the Wayland protocol stack. The following checklist applies in addition to the per-project checklists above.

**Submission target:** Each change is submitted independently to its respective project's review channel, but the cover letters and MR descriptions must cross-reference each other so reviewers can evaluate the full picture.

- [ ] **Identify all stack layers affected.** Before writing a single line of patch, map out the full dependency chain: kernel uAPI change → libdrm wrapper → Mesa driver code → Wayland protocol → compositor implementation → application API. Any layer that is not updated degrades the feature to a partial or unusable state in production.

- [ ] **Kernel uAPI merged (or at minimum in `drm-next`) before Mesa changes are submitted.** Mesa's CI must be able to build against a stable kernel header. Submitting a Mesa MR that `#include`s a uAPI header not yet present in the kernel tree will fail CI and will not be accepted. The standard practice is to carry the kernel header addition as a local copy in `include/drm/` within the Mesa tree until the kernel change is in a tagged release, then update to use the system header.

- [ ] **`libdrm` wrapper submitted concurrently with or before the Mesa MR.** New kernel IOCTLs must have a corresponding `libdrm` wrapper function before Mesa can use them through the standard `libdrm` API. The `libdrm` repository is at `https://gitlab.freedesktop.org/mesa/drm`. Submit the `libdrm` MR first or simultaneously, and link to it from the Mesa MR.

- [ ] **Wayland protocol in `staging/` before the compositor implementation lands in a distribution.** Compositors should not ship support for a protocol that is not yet in `staging/`. If a compositor ships an internal protocol (with a vendor prefix like `_wl_foo`) that later needs to be standardised, the standardisation process must handle backward compatibility explicitly.

- [ ] **Cover letter / MR description links all companion changes.** The kernel cover letter must link to the libdrm MR and the Mesa MR. The Mesa MR must link to the kernel patchset and the libdrm MR. Without cross-references, reviewers cannot verify that the full stack is consistent.

  Example cover letter fragment:

  ```
  Companion changes:
    libdrm:  https://gitlab.freedesktop.org/mesa/drm/-/merge_requests/NNNN
    Mesa:    https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/MMMM
    Wayland: https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/PPPP
  ```

- [ ] **Feature flag or compile-time guard for pre-new-uAPI kernels.** Mesa code that uses a new kernel IOCTL must degrade gracefully on kernels that predate the new uAPI. The conventional pattern is a runtime `drmGetVersion()` check or a compile-time `#ifdef DRM_IOCTL_MY_NEW_IOCTL` guard that falls back to a software path or disables the feature. Do not assume the runtime kernel matches the build-time kernel headers.

- [ ] **IGT test covers the new uAPI.** New kernel uAPI additions should have an IGT test that exercises the full round-trip: userspace library call → IOCTL → kernel handler → result validation. The IGT test should be submitted to `https://gitlab.freedesktop.org/drm/igt-gpu-tools` and linked from the kernel cover letter.

- [ ] **Announce on `dri-devel` before work begins.** For features that require significant coordination across multiple projects, send an RFC email to `dri-devel@lists.freedesktop.org` outlining the design before writing implementation code. This avoids the scenario where a large patchset arrives for review and the design is rejected, requiring a complete rewrite.

---

## E.5 Version Verification Footers

Each checklist in this appendix was verified against the project versions listed below. Check these versions against the current project state before relying on specific item details (tool flags, policy requirements, CI job names) for a submission:

| Checklist | Last verified against |
|-----------|----------------------|
| E.1 Kernel DRM | Linux kernel **6.15-rc7** (May 2025); `checkpatch.pl` from that tree; `MAINTAINERS` file at tag `v6.14` |
| E.2 Mesa | Mesa **25.1.0** (released 2025-05-07); CI job structure from `.gitlab-ci.yml` at that tag; `docs/submittingpatches.rst` at that commit |
| E.3 Wayland protocols | `wayland-protocols` **1.38** (2025 Q1); `GOVERNANCE.md` and `CONTRIBUTING.md` at that tag |
| E.4 Cross-project | Reflects conventions current as of kernel 6.15 / Mesa 25.1 / wayland-protocols 1.38 |

**Recommended update cadence:** Review the kernel DRM checklist at the start of each new kernel cycle (approximately every 9–10 weeks). Review the Mesa checklist after each major Mesa release (quarterly). Review the Wayland protocol checklist annually unless a protocol process overhaul is announced at XDC or via the wayland-devel mailing list.

---

## Integrations

This appendix is a companion to **Chapter 32 (Contributing to the Linux Graphics Stack)**, which provides the full narrative context behind each checklist item, including the history of how policies evolved and the rationale for requirements that might otherwise seem bureaucratic.

The kernel DRM checklist's UAPI documentation requirement connects to **Chapter 2 (DRM Architecture)** and **Chapter 3 (KMS: Kernel Mode Setting)**, which describe the DRM property and IOCTL system whose documentation is required. The `checkpatch.pl` and coding-style requirements are part of the broader kernel culture described in the kernel process documentation referenced in Chapter 32.

The Mesa checklist's `shader-db` requirement connects to **Chapter 14 (The NIR Intermediate Representation)** and **Chapter 15 (Mesa Compiler Infrastructure)**, where the shader compilation pipeline that shader-db exercises is described in detail. The REUSE compliance and copyright header requirements are part of the Mesa project governance described in Chapter 32.

The Wayland protocol checklist connects to **Chapter 20 (Wayland Architecture)** and **Chapter 21 (Wayland Protocols and Extensions)**, which describe the protocol system and explain why the lifecycle stages and security requirements exist. The cross-project coordination checklist connects to **Chapter 30 (Stack-Wide Feature Integration)**, which walks through a concrete example of a feature spanning all layers.

---

## References

1. Linux kernel patch submission guide: [https://docs.kernel.org/process/submitting-patches.html](https://docs.kernel.org/process/submitting-patches.html)

2. Linux kernel stable-tree rules: [https://docs.kernel.org/process/stable-kernel-rules.html](https://docs.kernel.org/process/stable-kernel-rules.html)

3. DRM subsystem introduction and submission requirements: [https://docs.kernel.org/gpu/introduction.html](https://docs.kernel.org/gpu/introduction.html)

4. Linux kernel patch submission checklist: [https://dri.freedesktop.org/docs/drm/process/submit-checklist.html](https://dri.freedesktop.org/docs/drm/process/submit-checklist.html)

5. `checkpatch.pl` documentation: [https://dri.freedesktop.org/docs/drm/dev-tools/checkpatch.html](https://dri.freedesktop.org/docs/drm/dev-tools/checkpatch.html)

6. `b4` kernel patch tool documentation: [https://b4.docs.kernel.org/en/latest/contributor/overview.html](https://b4.docs.kernel.org/en/latest/contributor/overview.html)

7. `b4` GitHub repository: [https://github.com/mricon/b4](https://github.com/mricon/b4)

8. Mesa submitting patches documentation: [https://docs.mesa3d.org/submittingpatches.html](https://docs.mesa3d.org/submittingpatches.html)

9. Mesa CI documentation: [https://docs.mesa3d.org/ci/index.html](https://docs.mesa3d.org/ci/index.html)

10. Mesa environment variables documentation: [https://docs.mesa3d.org/envvars.html](https://docs.mesa3d.org/envvars.html)

11. Mesa licence and copyright documentation: [https://docs.mesa3d.org/license.html](https://docs.mesa3d.org/license.html)

12. Mesa 25.1.0 release notes: [https://docs.mesa3d.org/relnotes/25.1.0.html](https://docs.mesa3d.org/relnotes/25.1.0.html)

13. wayland-protocols GOVERNANCE.md (via Chromium mirror): [https://chromium.googlesource.com/external/anongit.freedesktop.org/git/wayland/wayland-protocols/+/HEAD/GOVERNANCE.md](https://chromium.googlesource.com/external/anongit.freedesktop.org/git/wayland/wayland-protocols/+/HEAD/GOVERNANCE.md)

14. wayland-protocols project on GitLab: [https://gitlab.freedesktop.org/wayland/wayland-protocols](https://gitlab.freedesktop.org/wayland/wayland-protocols)

15. Wayland protocol catalogue (wayland.app): [https://wayland.app/protocols/](https://wayland.app/protocols/)

16. IGT GPU Tools repository: [https://gitlab.freedesktop.org/drm/igt-gpu-tools](https://gitlab.freedesktop.org/drm/igt-gpu-tools)

17. libdrm repository: [https://gitlab.freedesktop.org/mesa/drm](https://gitlab.freedesktop.org/mesa/drm)

18. drm-misc tree (cgit): [https://cgit.freedesktop.org/drm/drm-misc/](https://cgit.freedesktop.org/drm/drm-misc/)

19. REUSE compliance tool: [https://reuse.software/](https://reuse.software/)

20. Phoronix — Linux 6.16 DRM misc-next: [https://www.phoronix.com/news/Linux-6.16-DRM-Misc-Next](https://www.phoronix.com/news/Linux-6.16-DRM-Misc-Next)
