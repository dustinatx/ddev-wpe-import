# wpe-import: Import WP Engine backups into DDEV

A DDEV command that imports WP Engine backup zips into local WordPress development environments.

## Features

- Accepts Windows or Linux paths (auto-converts `C:\...` to `/mnt/c/...`)
- Extracts backup and cleans up WPE-specific files
- Preserves relevant wp-config.php constants (with interactive prompts for unknowns)
- Configures DDEV and imports database
- Runs search-replace to update URLs
- Optional: Create test admin account (admin:password) or reset existing admin password

## Installation

```bash
# Clone or download the script, then symlink to DDEV global commands:
ln -s /path/to/wpe-import ~/.ddev/commands/host/wpe-import
```

## Usage

```bash
ddev wpe-import
```

The script will prompt you for:
1. Path to WP Engine backup zip
2. Project directory
3. Project name
4. Admin account options (create new or reset existing)

## Requirements

- [DDEV](https://ddev.com/)
- `unzip`
- `curl`

## What Gets Cleaned Up

WPE-specific files are automatically removed:
- `wp-content/advanced-cache.php`
- `wp-content/object-cache.php`
- `wp-content/mu-plugins/wpengine-common/` and other WPE mu-plugins

## wp-config.php Handling

The original `wp-config.php` is saved as `wp-config-backup.php`. A new one is generated with:
- Local development defaults (WP_DEBUG enabled, etc.)
- Fresh authentication keys from WordPress.org
- Original table prefix
- Any preserved constants from the original

## License

MIT
