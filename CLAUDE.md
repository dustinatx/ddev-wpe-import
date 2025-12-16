# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a single-file Bash script (`wpe-import`) that imports WP Engine backup zips into new DDEV WordPress projects. It handles path conversion (Windows/WSL), extracts backups, cleans up WPE-specific files, preserves relevant wp-config constants, and configures DDEV with proper database import and URL replacement.

## Testing

No automated tests. Test manually with a real WP Engine backup zip file:
```bash
./wpe-import
```

Check bash syntax without running:
```bash
bash -n wpe-import
```

## Installation

Symlink to DDEV global commands:
```bash
ln -s /path/to/wpe-import ~/.ddev/commands/host/wpe-import
```

Then run from anywhere: `ddev wpe-import`

## Architecture

The script follows a 5-phase flow defined in `main()`:

1. **Input Collection** - Prompts for backup path, project directory, project name, admin account options
2. **Project Setup** - Extracts backup, cleans WPE files, analyzes wp-config.php, configures DDEV
3. **DDEV & Database** - Starts DDEV, imports database, runs search-replace on URLs
4. **Admin Account Management** - Optional: create default admin (admin:password) OR reset existing admin password
5. **Completion** - Shows success message with cd instruction or useful commands

## Key Implementation Details

### Constants Handling (lines 38-71)
- `BLACKLIST` array: WPE-specific constants, security keys, DB settings - always excluded
- `WHITELIST` array: Safe constants to auto-preserve (debug flags, performance settings)
- Unknown constants trigger interactive prompt

### wp-config.php Generation (lines 408-500)
Creates custom wp-config.php with:
- Local dev defaults (WP_DEBUG, AUTOMATIC_UPDATER_DISABLED, etc.)
- Fresh authentication keys/salts fetched from `https://api.wordpress.org/secret-key/1.1/salt/`
- Extracted table prefix
- Preserved constants from original
- DDEV database config inclusion

### Special Cases
- `WP2FA_ENCRYPT_KEY` in whitelist triggers auto-adding `DISABLE_2FA_LOGIN = true`
- Windows paths (`C:\...`) auto-converted to WSL format (`/mnt/c/...`)
- Surrounding quotes stripped from pasted paths
- Existing DDEV projects in target directory cause immediate abort
- Old URL extracted via direct MySQL query (bypasses wp-config-ddev.php WP_SITEURL override)
- Uses `--skip-themes --skip-plugins` on WP-CLI commands for speed/reliability

### DDEV Global Command
The script includes annotations for DDEV global command support:
```bash
## CanRunGlobally: true
## Description: Import WP Engine backups into new DDEV WordPress projects
```

## Reference

See `PLAN.md` for detailed implementation decisions and the complete blacklist/whitelist rationale.
