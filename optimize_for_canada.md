# Canada CDN Optimization

Fork-local changes to prefer **GitHub** (and other non-China mirrors) over **ddsrem.com**, **gitee**, and China-oriented ghproxy URLs. Intended for users outside mainland China (e.g. Canada) where GitHub is faster and more reliable.

## CDN priority (after changes)

| Tier | Source | Base URL |
|------|--------|----------|
| 1 | GitHub raw | `https://raw.githubusercontent.com/xiaoyaDev/xiaoya-alist/master/` |
| 2 | jsDelivr (fastly) | `https://fastly.jsdelivr.net/gh/xiaoyaDev/xiaoya-alist@latest/` |
| 3 | DDSRem mirror | `https://ddsrem.com/xiaoya/` |
| 4 | Gitee (base modules only) | `https://gitee.com/ddsrem/xiaoya-alist-base/raw/master/` |

**Exceptions:**

- **Xiaoyahelper** (`aliyun_clear.sh`): `xiaoyahelper.zengge99.eu.org` → `xiaoyahelper.ddsrem.com`
- **macOS Homebrew**: official `Homebrew/install` (GitHub) → gitee `HomebrewCN` fallback
- **Emby lovechen DB** (`library.db`, `temp.sql`): GitHub → jsDelivr → `cdn.jsdelivr.net` (legacy third fallback)

## New helpers in `all_in_one.sh`

Added near the top of `all_in_one.sh` (after log functions):

| Symbol | Purpose |
|--------|---------|
| `XIAOYA_GITHUB_RAW` | GitHub raw base for repo files |
| `XIAOYA_JSdelivr` | jsDelivr CDN base |
| `XIAOYA_DDSREM` | DDSRem mirror base |
| `XIAOYA_GITEE_BASE` | Gitee base-module mirror |
| `XIAOYA_NOTIFY_FETCH` | Inline shell snippet: curl fallback chain for `xiaoya_notify.sh` |
| `curl_xiaoya_to_file dest path` | Download a repo file to disk |
| `curl_xiaoya_script path` | Stream-execute a repo script via curl fallback |
| `curl_xiaoya_base_module file dest` | Download a `base/*.sh` module |

## File-by-file changes

### `all_in_one.sh`

| Area | Before | After |
|------|--------|-------|
| `base/*.sh` bootstrap | gitee → GitHub | GitHub → jsDelivr → gitee |
| macOS re-download of `all_in_one.sh` | ddsrem → jsDelivr → GitHub | `curl_xiaoya_to_file` (GitHub first) |
| `first_init` — `xiaoya_alist` CLI | ddsrem → jsDelivr → GitHub; alias pointed at ddsrem on success | `curl_xiaoya_to_file`; alias always uses GitHub raw |
| Emby config editor (menu 2→4) | ddsrem only | `curl_xiaoya_script emby_config_editor.sh` |
| Emby lovechen DB conversion | jsDelivr only | GitHub → jsDelivr → cdn.jsdelivr |
| Xiaoyahelper install / once-run | ddsrem → zengge99 | zengge99 → ddsrem |
| macOS Homebrew auto-install | gitee HomebrewCN only | GitHub official installer → gitee fallback |

### `main.sh`

Bootstrap download of `all_in_one.sh`:

- Default: **GitHub → jsDelivr → ddsrem** (was ddsrem → jsDelivr → GitHub)
- With `XIAOYA_BRANCH`: **GitHub → jsDelivr** (was jsDelivr → GitHub)

### `xiaoya_install.sh` (repo root sibling copy at `~/code/xiaoya_install.sh`)

Same bootstrap order as `main.sh`.

### `emby_config_editor.sh`

Download of `xiaoya_alist` helper: **GitHub → jsDelivr → ddsrem** (was ddsrem → jsDelivr → GitHub).

### `base/deprecation.sh`

- Added `XIAOYA_NOTIFY_FETCH` constant.
- All cron / one-shot commands that fetched `xiaoya_notify.sh` from ddsrem only now use the three-tier fallback chain (6 call sites).

### `cron/rootfs/etc/s6-overlay/s6-rc.d/init-config/run`

Container init writes cron/command scripts using `XIAOYA_NOTIFY_FETCH` instead of hard-coded ddsrem URL (2 call sites).

## Not changed

These remain upstream / China-oriented by design or low impact in Canada:

| Item | Reason |
|------|--------|
| `README.md` install examples | Still show upstream ddsrem one-liners |
| Docker Hub mirror probe list (`mirrors` in `all_in_one.sh`) | Auto speed-test; `docker.io` is included and usually wins abroad |
| Emby metadata downloads | Pulled from local Xiaoya Alist instance (`/d/元数据/...`), not GitHub |
| `xiaoya_data_downloader.sh` | Deprecated (exits immediately); already had GitHub first |
| Aliyun / Quark / 115 API endpoints | Service APIs, not script CDN |
| `base/jellyfin.sh` mention of `xy.ggbond.org` | Third-party install hint only |

## How to use

Run the **local** fork instead of the remote bootstrap:

```bash
sudo bash /path/to/xiaoya-alist/all_in_one.sh
```

Direct function invocation (example — install Xiaoya Alist):

```bash
sudo bash /path/to/xiaoya-alist/all_in_one.sh install_xiaoya_alist
```

Using the upstream one-liner still downloads **upstream** scripts from ddsrem unless you host your own mirror:

```bash
bash -c "$(curl --insecure -fsSL https://ddsrem.com/xiaoya_install.sh)"
```

## Caveats

1. **Existing crontab entries** written before this fork still point at old URLs. Re-create notify/sync cron via the menu, or edit crontab manually.
2. **`base/` in the cloned repo** is still not read directly at runtime; modules are downloaded via `curl_xiaoya_base_module` (GitHub first). Local `base/` edits only apply if you change the loader logic.
3. **macOS non-root path** re-downloads `all_in_one.sh` to `/tmp` before `sudo`; with this fork that download is GitHub-first, but it still fetches remote copy rather than the local file path.
4. **Branch pinning**: GitHub/jsDelivr URLs use `master` / `@latest`. For a non-master branch, set `XIAOYA_BRANCH` when using `main.sh` / `xiaoya_install.sh` bootstrap.

## Summary

| Category | Sites touched |
|----------|---------------|
| Modified files | 6 (`all_in_one.sh`, `main.sh`, `emby_config_editor.sh`, `base/deprecation.sh`, `cron/.../init-config/run`, plus `~/code/xiaoya_install.sh` outside repo) |
| Fallback chains added/rewired | ~15 |
| gitee demoted to | Last resort (base modules) or Homebrew fallback (macOS only) |
| ddsrem demoted to | Third tier for repo scripts; second tier for xiaoyahelper |
