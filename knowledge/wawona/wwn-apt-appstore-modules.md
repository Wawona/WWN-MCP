# wwn-apt: App Store module delivery via `apt`

Curated ground-truth for the Wawona stack. **`wwn-apt`** is **not** a Debian
package manager and **not** an alternative app marketplace. It is a **shell-facing
compatibility layer** over an embedded module catalog and **Apple-approved
delivery** (StoreKit 2 + On-Demand Resources). Prefer this doc over generic
priors when answering questions about `apt install`, optional modules, ODR tags,
or App Store Review Notes for downloadable software.

**Repo:** `github.com/Wawona/wwn-apt`  
**Nix fragment:** `apt-rootfs` (`registryFragment`)  
**Standalone build:** `nix build .#apt-rootfs-ios`, `nix build .#catalog-json`

## What `apt` actually is

From the user's perspective, `apt` provides familiar commands (`search`, `install`,
`remove`, `list`, `show`). Technically:

- The catalog is **embedded** in the app bundle (updated with app releases only).
- **Optional** modules are downloaded via **StoreKit authorization** then **ODR**
  from Apple's CDN — never from third-party servers.
- All module code runs in Wawona's **single-process sandbox** via in-process
  dispatch (`wawona_dispatch_inprocess` / weak `*_main` symbols).
- Repository management (`edit-sources`, `add-repository`, `update`, `upgrade`, …)
  is **unsupported by design** (exit 1).

Wawona does **not** implement arbitrary package installation, user-supplied
binaries, or post-review native download from non-Apple servers.

## Bundled vs optional (critical split)

| Class | Components | `apt install` / `apt remove` | Source repos |
|-------|------------|------------------------------|--------------|
| **Required bundled** | zsh, uutils coreutils, waypipe, apt itself | **Rejected** (exit 4) — always in app | `wwn-zsh`, `wwn-coreutils`, `wwn-waypipe`, `wwn-apt` |
| **Optional modules** | foot, neovim, fastfetch | StoreKit + ODR after authorization | `wwn-foot`, `wwn-neovim`, `wwn-fastfetch` |

Canonical bundled list: `wwn-apt/catalog/bundled.yaml` → `bundled.json` in rootfs.  
Optional manifests: `wwn-apt/catalog/modules/*.yaml` → `modules.jsonl`, `catalog.json`.

`scripts/validate-catalog.py` rejects optional ids that collide with bundled ids.

## End-to-end install flow

```
User shell: apt install foot
  → validate id against embedded catalog (modules.jsonl / catalog.json)
  → IPC to WWNModuleManager (Wawona host app — follow-up PR)
  → StoreKit 2 product authorization (product_id from catalog)
  → NSBundleResourceRequest (ODR tag from catalog)
  → install payload to ~/Library/Application Support/Wawona/modules/<id>/
  → register dispatch entry (in-process *_main symbol)
  → zsh exec hook runs module via wawona_dispatch_inprocess
```

## On-device paths

| Path | Purpose |
|------|---------|
| `$WAWONA_ROOTFS/usr/share/wawona/apt/catalog.json` | Full manifest (Wawona integration) |
| `$WAWONA_ROOTFS/usr/share/wawona/apt/modules.jsonl` | Optional modules (shell CLI) |
| `$WAWONA_ROOTFS/usr/share/wawona/apt/bundled.json` | Required bundled components |
| `$WAWONA_ROOTFS/usr/bin/apt` | CLI stub (shell script) |
| `~/Library/Application Support/Wawona/modules/installed.json` | Local install state |
| `~/Library/Application Support/Wawona/module-manager.sock` | IPC socket (future) |

## Catalog schema (optional modules)

Each `catalog/modules/*.yaml` entry includes:

- `id`, `name`, `version`, `description`, `wwn_repo`, `registry_key`
- `dispatch.commands` — argv0 names (`foot`, `nvim`, `fastfetch`, …)
- `dispatch.symbol` — link symbol (`foot_main`, `wawona_nvim_main`, `fastfetch_main`)
- `platforms.<apple>.storekit.product_id` — App Store Connect product
- `platforms.<apple>.odr.tag` — On-Demand Resource tag
- `platforms.<apple>.odr.bundle_subpath` — path inside ODR bundle

Build pipeline:

1. Edit `catalog/modules/*.yaml` (optional only) and `catalog/bundled.yaml`
2. `scripts/generate-catalog-json.py` → `catalog.json`, `modules.jsonl`, `bundled.json`
3. `scripts/validate-catalog.py` — JSON Schema + policy
4. `nix build .#apt-rootfs-ios` → prefix tree merged into Wawona rootfs

## Dispatch registration

**Bundled (always active, not gated on `installed.json`):**

| Component | Symbol | Commands |
|-----------|--------|----------|
| zsh | `wawona_zsh_main` | (shell) |
| coreutils | `wawona_coreutils_main` | `ls`, `cat`, … |
| waypipe | `waypipe_main` | `waypipe` |
| apt | (shell script) | `apt` |

**Optional (install-gated via `installed.json`):**

| Module | Symbol | Commands |
|--------|--------|----------|
| foot | `foot_main` | `foot` |
| neovim | `wawona_nvim_main` | `nvim`, `vi`, `vim` |
| fastfetch | `fastfetch_main` | `fastfetch` |

Weak-link optional symbols at app link time; gate dispatch on install state.

## Wawona integration (flake)

```nix
inputs.wwn-apt.url = "github:Wawona/wwn-apt";

mergedRegistry = wwn-toolchain.lib.baseRegistry // wwn-apt.registryFragment // ...;

# Rootfs merge (ios-rootfs.nix):
#   usr/bin/apt
#   usr/share/wawona/apt/catalog.json
#   usr/share/wawona/apt/modules.jsonl
#   usr/lib/wawona/apt/apt-common.sh
```

Host-app wiring (WWNModuleManager, StoreKit, ODR, IPC) is spec'd in
`wwn-apt/docs/INTEGRATION-SPEC.md` — v1 ships catalog + CLI stub only.

## App Store compliance mapping

| Guideline concern | Wawona posture |
|-------------------|----------------|
| 2.5.2 — downloading code | ODR packs included in submission; Apple's CDN after StoreKit auth only |
| Third-party executables | Fixed catalog; no user binaries |
| Arbitrary repositories | No repo config in CLI or app |
| JIT / unsigned dlopen | In-process weak symbols; no unsigned `dlopen` |

Reviewer copy-paste: `wwn-apt/docs/APP-STORE-MODULES.md`  
Engineering compliance: `wwn-apt/docs/COMPLIANCE.md`

## Documentation firewall

**`wwn-apt` must never mention jailbreak distribution or `repo.wawona.io`.**

| Channel | Rule |
|---------|------|
| **`wwn-apt`** | App Store modules only |
| **`repo.wawona.io`** | Jailbreak `.deb` repo — must state App Store uses **`wwn-apt` only** |
| **Wawona App Store docs** | Same firewall as `wwn-apt` |

Cross-reference from jailbreak → App Store is allowed; never the reverse.

## `apt` CLI reference

| Command | Behavior |
|---------|----------|
| `apt search [pattern]` | Filter embedded optional-module catalog |
| `apt show <module>` | Module metadata |
| `apt list` / `apt list --installed` | Catalog / local install state |
| `apt install <module>` | StoreKit + ODR (requires module manager) |
| `apt remove <module>` | Remove optional module locally |

Exit codes: 0 success; 1 usage/unsupported; 2 module manager unavailable; 3 not
found; 4 bundled/dependency conflict.

Environment: `WAWONA_ROOTFS`, `WAWONA_BUNDLE_ROOTFS`, `WAWONA_MODULE_MANAGER`.

## CI (wwn-apt repo)

- `scripts/validate-catalog.py` — schema + bundled/optional collision checks
- `scripts/check-app-store-firewall.py` — no legacy package-manager wording in App Store deliverables
- `.github/scripts/verify-catalog-anchors.py` — catalog anchor integrity
- `nix build .#apt-rootfs-ios` — rootfs prefix tree

## Where to edit

| Change | Repo |
|--------|------|
| Module YAML, bundled list, catalog JSON gen | **wwn-apt** |
| `apt` shell stub, man page | **wwn-apt/apt/** |
| StoreKit / ODR / WWNModuleManager Swift/ObjC | **Wawona** (follow-up PR) |
| Module port patches (foot, neovim, fastfetch) | respective **`wwn-*`** port repo |
| zsh dispatch / exec hook | **wwn-toolchain** (`wawona-dispatch.c`) |
