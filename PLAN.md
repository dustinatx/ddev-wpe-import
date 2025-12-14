# Implementation Plan for `ddev wpe-import` Command

## Overview
A Bash script that will be executable as `ddev wpe-import` from anywhere, importing WP Engine backups into new DDEV WordPress projects.

## File Structure
- **Location**: `/home/dustinatx/dev/projects/ddev-wpe-import/wpe-import` (you'll symlink to `.ddev/commands/host/`)
- **Type**: Bash script with executable permissions

---

## Implementation Flow

### Phase 1: Input Collection & Validation

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
- If directory exists and is not empty, ask if they want to use it anyway

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
- Copy zip to project directory if not already there
- Extract: `unzip -q backup.zip`
- Verify extraction successful

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

#### 2.4 wp-config.php Handling
- Rename: `mv wp-config.php wp-config-backup.php`
- Extract table prefix from wp-config-backup.php:
  - Use grep/sed: `grep '$table_prefix' wp-config-backup.php`
  - Parse the value (e.g., `wp_`, `wp_abc_`, etc.)
  - Store in variable: `TABLE_PREFIX`

#### 2.5 DDEV Configuration
- Run: `ddev config --project-type=wordpress --docroot=. --create-docroot=false --project-name=projectname`
- This creates `.ddev/config.yaml`, `wp-config.php`, and `wp-config-ddev.php`

#### 2.6 Modify wp-config.php
- Edit the DDEV-generated `wp-config.php`
- Add the table prefix line BEFORE the `wp-config-ddev.php` include
- Structure:
  ```php
  <?php
  /**
   #ddev-generated: Automatically generated WordPress wp-config.php file.
   ddev manages this file and may delete or overwrite the file unless this comment is removed.
   */

  $table_prefix = 'wp_xyz_';

  if (file_exists(__DIR__ . '/wp-config-ddev.php')) {
      require_once(__DIR__ . '/wp-config-ddev.php');
  }

  // ... rest of DDEV's generated content
  ```

#### 2.7 Delete Backup Zip
- Remove: `rm backup.zip`

---

### Phase 3: DDEV & Database

#### 3.1 Start DDEV
- Run: `ddev start`
- Wait for completion

#### 3.2 Database Import
- Check if `wp-content/mysql.sql` exists
- If NOT found, exit with error: `"Error: Database file not found at wp-content/mysql.sql"`
- If found, import: `ddev import-db --file=wp-content/mysql.sql`

#### 3.3 Extract Old URL
- Query database for old URL:
  ```bash
  OLD_URL=$(ddev wp option get siteurl --quiet)
  ```
- Parse to get just the domain (remove http/https, trailing slashes)

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

#### 5.1 Launch Project
- Run: `ddev launch`
- This opens the site in default browser

#### 5.2 Success Message
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
- Invalid zip file path
- Unzip failure
- DDEV config/start failures
- Database file not found (exit with clear message)
- Database import failure
- WP-CLI command failures
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
10. **Constants to preserve**: Only `$table_prefix` for now
11. **Zip file after extraction**: Delete it automatically
12. **Existing directory**: Ask user if they want to use it anyway
13. **Admin password reset**: List available admin users and let user pick
14. **Table prefix placement**: Add to wp-config.php BEFORE loading wp-config-ddev.php
