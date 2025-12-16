# Implementation Plan for `ddev wpe-import` Command

## Overview
A Bash script that will be executable as `ddev wpe-import` from anywhere, importing WP Engine backups into new DDEV WordPress projects.

## File Structure
- **Location**: `/home/dustinatx/dev/projects/ddev-wpe-import/wpe-import` (you'll symlink to `.ddev/commands/host/wpe-import`)
- **Type**: Bash script with executable permissions

---

## Implementation Flow

### Phase 1: Input Collection & Validation

#### 1.0 Prerequisites Check
- Check if `unzip` command is available: `command -v unzip`
- Check if `curl` command is available: `command -v curl` (needed for fetching auth keys)
- If not found, abort with error
- Assume DDEV is installed (will fail naturally if not)

#### 1.1 Backup Zip Location
- Prompt user for backup zip file path
- Accept both formats:
  - Windows: `C:\Users\delat\Downloads\site-archive.zip`
  - Linux: `/mnt/c/Users/delat/Downloads/site-archive.zip` or `~/site-archive.zip`
- Convert Windows paths to WSL2 format: `C:\` → `/mnt/c/`
- Validate file exists and is a zip file
- Expand `~` to full home directory path

#### 1.2 Project Directory
- Display current working directory
- Ask: "Create project here? (y/n)"
- If no, prompt for target directory path
- Handle relative and absolute paths
- **If `.ddev/config.yaml` exists**: Abort with error "This directory already contains a DDEV project"
- If directory exists and is not empty (including hidden files like `.git`), ask if they want to use it anyway

#### 1.3 Project Name
- Prompt for project name
- Sanitize automatically:
  - Convert to lowercase
  - Replace spaces and special characters with hyphens
  - Remove consecutive hyphens
  - Trim leading/trailing hyphens
- Check if DDEV project with this name already exists: `ddev list | grep projectname`
- If exists, show error and ask for different name

#### 1.4 Admin Password Reset Option
- Ask: "Do you want to reset an admin user password after import? (y/n)"
- Store response for later

#### 1.5 Confirmation Prompt
- Display summary:
  ```
  Summary:
  - Backup: /path/to/backup.zip
  - Project Directory: /path/to/project
  - Project Name: projectname
  - DDEV URL: https://projectname.ddev.site
  - Reset Admin Password: Yes/No

  Proceed? (y/n)
  ```
- If no, exit cleanly

---

### Phase 2: Project Setup

#### 2.1 Directory Preparation
- Create project directory if it doesn't exist: `mkdir -p /path/to/project`
- Change to project directory: `cd /path/to/project`

#### 2.2 Backup Extraction
- If zip is NOT in project directory: copy it there first
- If zip IS already in project directory: extract directly (no copy needed)
- Extract: `unzip -q backup.zip`
- Verify extraction successful
- Delete the zip copy from project directory (NOT the original)
- **Verify required files exist:**
  - If `wp-config.php` doesn't exist: Abort with error "wp-config.php not found in backup"
  - If `wp-content/mysql.sql` doesn't exist: Abort with error "Database file not found at wp-content/mysql.sql"

#### 2.3 WP Engine Cleanup
Delete these files/directories (use `rm -rf`, ignore if not exist):
```
wp-content/advanced-cache.php
wp-content/object-cache.php
wp-content/mu-plugins/force-strong-passwords/
wp-content/mu-plugins/wpe-cache-plugin/
wp-content/mu-plugins/wpe-update-source-selector/
wp-content/mu-plugins/wpe-wp-sign-on-plugin/
wp-content/mu-plugins/wpengine-common/
wp-content/mu-plugins/mu-plugin.php
wp-content/mu-plugins/slt-force-strong-passwords.php
wp-content/mu-plugins/stop-long-comments.php
wp-content/mu-plugins/wpe-cache-plugin.php
wp-content/mu-plugins/wpe-update-source-selector.php
wp-content/mu-plugins/wpe-wp-sign-on-plugin.php
wp-content/mu-plugins/wpengine-security-auditor.php
```

#### 2.4 wp-config.php Analysis
- Rename: `mv wp-config.php wp-config-backup.php`
- Extract table prefix from wp-config-backup.php:
  - Use grep/sed: `grep '$table_prefix' wp-config-backup.php`
  - Parse the value (e.g., `wp_`, `wp_abc_`, etc.)
  - Store in variable: `TABLE_PREFIX`
  - **If parsing fails or table prefix not found**: Abort with error

##### Constants Extraction Strategy
Use a hybrid approach with blacklist, whitelist, and interactive mode:

**Blacklist (always exclude):**
- WPE-specific: `WPE_APIKEY`, `PWP_NAME`, `WPE_CLUSTER_ID`, `WPE_CLUSTER_TYPE`, `IS_WPE`, `IS_WPE_SNAPSHOT`, `WPE_ISP`, `WPE_BPOD`, `WPE_RO_FILESYSTEM`, `WPE_LARGEFS_BUCKET`, `WPE_SFTP_PORT`, `WPE_SFTP_ENDPOINT`, `WPE_LBMASTER_IP`, `WPE_CDN_DISABLE_ALLOWED`, `WPE_FORCE_SSL_LOGIN`, `WPE_EXTERNAL_URL`, `WPE_WHITELABEL`, `WPE_BETA_TESTER`
- Security keys/salts: `AUTH_KEY`, `SECURE_AUTH_KEY`, `LOGGED_IN_KEY`, `NONCE_KEY`, `AUTH_SALT`, `SECURE_AUTH_SALT`, `LOGGED_IN_SALT`, `NONCE_SALT`
- Database connection: `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_CHARSET`, `DB_COLLATE`
- `WP_CACHE` (WPE-specific caching)
- `FS_METHOD`, `FS_CHMOD_DIR`, `FS_CHMOD_FILE` (filesystem settings)
- `FORCE_SSL_LOGIN`, `WP_TURN_OFF_ADMIN_BAR`, `DISABLE_WP_CRON`, `ALTERNATE_WP_CRON`, `WP_CRON_LOCK_TIMEOUT` (WPE/hosting defaults)
- `WPLANG` (deprecated since WP 4.0)
- `DISABLE_2FA_LOGIN` (we set this ourselves when WP2FA_ENCRYPT_KEY is present)
- `WP_DEBUG`, `WP_DEBUG_LOG`, `WP_DEBUG_DISPLAY`, `AUTOMATIC_UPDATER_DISABLED`, `WP_AUTO_UPDATE_CORE`, `WP_ENVIRONMENT_TYPE` (we set these ourselves or they're redundant)

**Whitelist (auto-preserve):**
- Debug flags: `SCRIPT_DEBUG`, `SAVEQUERIES`
- Performance: `WP_POST_REVISIONS`, `AUTOSAVE_INTERVAL`, `WP_MEMORY_LIMIT`, `WP_MAX_MEMORY_LIMIT`, `EMPTY_TRASH_DAYS`
- Core settings: `DISALLOW_FILE_EDIT`, `DISALLOW_FILE_MODS`
- Security: `WP2FA_ENCRYPT_KEY` (if present, also triggers adding `DISABLE_2FA_LOGIN = true`)

**Interactive mode:**
- For any `define()` statements not in blacklist or whitelist
- Display the constant name and value
- Prompt: "Keep this constant? (y/n)"
- Collect approved constants for inclusion

**Implementation (grep + sed with line preservation):**
```bash
# Extract all define statements
DEFINE_LINES=$(grep -E "^\s*define\s*\(" wp-config-backup.php)

# Loop through each line
while IFS= read -r line; do
    # Extract constant name only (first parameter)
    CONSTANT_NAME=$(echo "$line" | sed -E "s/.*define\s*\(\s*['\"]([^'\"]+)['\"].*/\1/")

    # Check blacklist - skip if found
    if [[ " ${BLACKLIST[@]} " =~ " ${CONSTANT_NAME} " ]]; then
        continue
    fi

    # Check whitelist - preserve entire line if found
    if [[ " ${WHITELIST[@]} " =~ " ${CONSTANT_NAME} " ]]; then
        CONSTANTS_TO_PRESERVE+=("$line")
        continue
    fi

    # Interactive mode - ask user
    echo "Found: $CONSTANT_NAME"
    echo "  $line"
    read -p "Keep this? (y/n) " answer
    [[ "$answer" == "y" ]] && CONSTANTS_TO_PRESERVE+=("$line")
done <<< "$DEFINE_LINES"
```

**Storage:**
- Store all whitelisted + approved constants in array (as complete define() lines)
- Store table prefix
- Track whether WP2FA_ENCRYPT_KEY was found (to add DISABLE_2FA_LOGIN)

#### 2.5 DDEV Configuration
- Run: `ddev config --project-type=wordpress --project-name=projectname`
- This creates `.ddev/config.yaml` and `wp-config-ddev.php`
- DDEV also creates a `wp-config.php`, but we replace it in step 2.6

#### 2.6 wp-config.php Creation
Always replace DDEV's wp-config.php with a custom version (WITHOUT `#ddev-generated`) for imported sites.

- Fetch fresh authentication keys/salts from `https://api.wordpress.org/secret-key/1.1/salt/`
- If fetch fails, abort with error

**Structure:**
```php
<?php
/**
 * wp-config.php for imported WPE site
 * This file is user-managed (not DDEV-generated)
 */

// ======================
// Local Development Defaults
// ======================
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', false);
@ini_set('display_errors', 0);
define('AUTOMATIC_UPDATER_DISABLED', true);
define('WP_ENVIRONMENT_TYPE', 'local');

// ======================
// Authentication Keys and Salts
// ======================
// (fetched from WordPress.org API)
define('AUTH_KEY',         '...');
define('SECURE_AUTH_KEY',  '...');
define('LOGGED_IN_KEY',    '...');
define('NONCE_KEY',        '...');
define('AUTH_SALT',        '...');
define('SECURE_AUTH_SALT', '...');
define('LOGGED_IN_SALT',   '...');
define('NONCE_SALT',       '...');

// ======================
// Table Prefix
// ======================
$table_prefix = '{TABLE_PREFIX}';  // Extracted from wp-config-backup.php

// ======================
// Preserved Constants (from original wp-config.php)
// ======================
// ... whitelisted/approved constants inserted here ...

// If WP2FA_ENCRYPT_KEY was preserved, also add:
// define('DISABLE_2FA_LOGIN', true);

// ======================
// DDEV Database Configuration
// ======================
$ddev_settings = dirname(__FILE__) . '/wp-config-ddev.php';
if (is_readable($ddev_settings) && !defined('DB_USER')) {
    require_once($ddev_settings);
}

// ======================
// Bootstrap WordPress
// ======================
if (!defined('ABSPATH')) {
    define('ABSPATH', __DIR__ . '/');
}
require_once ABSPATH . 'wp-settings.php';
```

**Notes:**
- User manages this file going forward
- Debug settings can be adjusted as needed
- `DISABLE_2FA_LOGIN` is auto-added only when `WP2FA_ENCRYPT_KEY` is present in preserved constants

---

### Phase 3: DDEV & Database

#### 3.1 Start DDEV
- Run: `ddev start`
- Wait for completion

#### 3.2 Database Import
- Import: `ddev import-db --file=wp-content/mysql.sql`
- After successful import, delete the database file: `rm wp-content/mysql.sql`

#### 3.3 Extract Old URL
- Query database for old URL:
  ```bash
  OLD_URL=$(ddev wp option get siteurl --quiet)
  ```
- **If WP-CLI command fails**: Abort with error
- Keep full URL with protocol (e.g., `https://oldsite.wpengine.com`)

#### 3.4 Update Site URLs
- Get new DDEV URL: `NEW_URL="https://projectname.ddev.site"`
- Update WordPress options:
  ```bash
  ddev wp option update home "$NEW_URL"
  ddev wp option update siteurl "$NEW_URL"
  ```

#### 3.5 Search-Replace
- Replace old domain with new domain throughout database:
  ```bash
  ddev wp search-replace "$OLD_URL" "$NEW_URL" --all-tables
  ```

---

### Phase 4: Admin Password Reset (Optional)

#### 4.1 If User Opted for Password Reset
- List admin users:
  ```bash
  ddev wp user list --role=administrator --format=table
  ```
- Prompt user to enter username from the list
- Prompt for new password (with confirmation)
- Reset password:
  ```bash
  ddev wp user update USERNAME --user_pass=NEWPASSWORD
  ```
- Display confirmation message

---

### Phase 5: Completion

#### 5.1 Success Message
```
✓ WP Engine import complete!

Project: projectname
URL: https://projectname.ddev.site
Location: /path/to/project

Useful commands:
  ddev start          - Start the project
  ddev stop           - Stop the project
  ddev wp             - Run WP-CLI commands
  ddev launch         - Open site in browser
  ddev describe       - Show project details

Original wp-config.php saved as: wp-config-backup.php
```

---

## Error Handling

Throughout the script, handle these error cases:
- Missing unzip or curl command
- Invalid zip file path
- Failed to fetch authentication keys from WordPress.org
- Target directory already contains a DDEV project
- Unzip failure
- wp-config.php not found in backup
- Table prefix parsing failure
- DDEV config/start failures
- Database file not found (exit with clear message)
- Database import failure
- WP-CLI command failures (especially old URL extraction)
- Invalid admin username during password reset

Use proper exit codes and clear error messages for each failure point.

---

## Technical Details

### Script Structure
- Shebang: `#!/bin/bash`
- Set error handling: `set -e` (exit on error), but use conditional checks where we want to continue
- Use functions for each major phase
- Use variables for paths, names, URLs
- Color output for better UX (green for success, red for errors, yellow for prompts)

### Path Conversion Function
```bash
convert_windows_path() {
    local path="$1"
    # Check if Windows path (contains : or \)
    if [[ "$path" =~ ^[A-Za-z]: ]]; then
        # Convert C:\ to /mnt/c/
        path=$(echo "$path" | sed 's|\\|/|g' | sed 's|^\([A-Za-z]\):|/mnt/\L\1|')
    fi
    echo "$path"
}
```

### Project Name Sanitization
```bash
sanitize_project_name() {
    local name="$1"
    # Lowercase, replace special chars with hyphens, remove consecutive hyphens
    name=$(echo "$name" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')
    echo "$name"
}
```

---

## Decision Log

### Question Responses
1. **Backup structure**: Files are at the root of the zip
2. **wp-config.php handling**: Rename original to wp-config-backup.php, let DDEV create new one, extract table prefix and add before wp-config-ddev.php include
3. **Search-replace old URL**: Extract from imported database
4. **Search-replace scope**: Just replace old domain with new DDEV URL
5. **Admin password reset**: Always ask during confirmation prompt (optional)
6. **Missing database file**: Exit with error message
7. **Project name validation**: Auto-convert to valid format AND check if already exists
8. **Database table prefix**: Auto-detect from wp-config.php
9. **DDEV configuration**: Use DDEV defaults
10. **Constants to preserve**: Use hybrid approach - blacklist to exclude WPE-specific/sensitive constants, whitelist to auto-preserve common safe constants, interactive mode for anything else. Always create custom wp-config.php (without #ddev-generated) with local dev defaults: WP_DEBUG, WP_DEBUG_LOG, WP_DEBUG_DISPLAY=false, AUTOMATIC_UPDATER_DISABLED, WP_ENVIRONMENT_TYPE='local'. Special handling: if WP2FA_ENCRYPT_KEY is preserved, also add DISABLE_2FA_LOGIN=true.
11. **Zip file after extraction**: Delete the copy from project directory, but leave the original alone
12. **Existing directory**: Ask user if they want to use it anyway (check includes hidden files)
13. **Admin password reset**: List available admin users and let user pick
14. **Table prefix placement**: Add to wp-config.php BEFORE loading wp-config-ddev.php
15. **URL format for search-replace**: Use full URLs with protocol (https://...)
16. **Auto-launch browser**: Don't auto-launch, just show the URL in success message
17. **Missing wp-config.php in backup**: Abort with error
