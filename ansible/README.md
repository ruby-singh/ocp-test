# RHEL Tools Ansible Role

This Ansible role downloads, packages, and deploys various tools for RHEL/AlmaLinux systems in air-gapped environments.

## Overview

The role performs the following operations:
1. **Download** - Downloads tools and dependencies from the internet
2. **Create Tar** - Packages all downloads into a combined tarball
3. **Copy Tar** - Copies the tarball to target nodes
4. **Extract** - Extracts the tarball on control node for staging
5. **Test** - Validates all operations completed successfully
6. **Install** - Installs tools on target nodes (optional)

## Tools Included

- **Container Tools**: podman, skopeo, buildah
- **Web Server**: httpd, httpd-tools
- **Network Tools**: firewalld, dnsmasq, bind-utils, telnet
- **Development Tools**: git, wget, curl, vim, openssl
- **Utilities**: jq, helm, haproxy, rsync

## Directory Structure

```
ansible/
├── playbooks/
│   ├── rhel-tools-playbook.yml        # Main playbook
│   └── test-tar-operations.yml        # Standalone test playbook
├── rhel-tools-role/
│   ├── tasks/
│   │   ├── main.yml                   # Main task orchestration
│   │   └── sub_tasks/
│   │       ├── create_tar_file.yml    # Download and create tarball
│   │       ├── copy_tar.yml           # Copy tarball to remote
│   │       ├── extract_tar.yml        # Extract files
│   │       ├── test_tar_operations.yml # Test validation
│   │       └── install_tools.yml      # Install tools (WIP)
│   ├── vars/
│   │   └── main.yml                   # Variables and download URLs
│   └── ...
└── build/                             # Generated during execution
    ├── tools_download/                # Downloaded files
    ├── tools_extract/                 # Extracted files
    └── tools_download.tar.gz          # Combined tarball
```

## Prerequisites

- Ansible installed on control node
- Internet access on control node (for downloads)
- Required tools: `curl`, `tar`, `rpm2cpio`, `cpio`, `sha256sum`

## Commands

### 1. Run Complete Workflow (All Tasks)

```bash
ansible-playbook playbooks/rhel-tools-playbook.yml -v
```

Runs all tasks: download → create tar → copy → extract

---

### 2. Test Downloaded and Extracted Tools

```bash
ansible-playbook playbooks/test-installed-tools.yml
```

**Independent test** that checks what tools are downloaded and extracted in the build folder. Shows:
- ✓ Downloaded files with size and version
- ✗ Missing downloads with error messages
- ✓ Extracted directories
- ✗ Missing extracts with error messages
- Summary with tool versions

---

## Test Coverage

The test suite (test-installed-tools.yml) validates:

### ✅ Downloaded Files
- Checks all downloaded files in `build/tools_download/`
- Reports file size in MB
- Extracts and displays version numbers
- Lists missing downloads with error messages

### ✅ Extracted Directories
- Verifies extracted directories in `build/tools_extract/`
- Checks bin/, httpd/, git/, curl/, vim/, openssl/, etc.
- Lists missing extracts with error messages

### ✅ Summary Report
- Total count of downloaded vs missing files
- Total count of extracted vs missing directories
- Complete version information for all tools (Podman 5.3.1, Httpd 2.4.65, etc.)

---

## Variables

Key variables defined in `rhel-tools-role/vars/main.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `project_root` | `{{ playbook_dir \| dirname }}` | Project root directory |
| `build_dir` | `{{ project_root }}/build` | Build output directory |
| `local_download_dir` | `{{ build_dir }}/tools_download` | Downloaded files location |
| `local_extract_dir` | `{{ build_dir }}/tools_extract` | Extracted files location |
| `local_combined_tar` | `{{ build_dir }}/tools_download.tar.gz` | Combined tarball path |
| `remote_tar_file` | `/tmp/tools_download.tar.gz` | Remote tarball location |

Download URLs for each tool are also defined in this file.

---

## Updating Tool Versions

To update tool versions, edit `rhel-tools-role/vars/main.yml`:

```yaml
podman_download_url: "https://github.com/containers/podman/releases/download/v5.3.1/podman-remote-static-linux_amd64.tar.gz"
httpd_download_url: "https://dlcdn.apache.org/httpd/httpd-2.4.65.tar.gz"
# ... etc
```

After updating, run the full playbook to download new versions.

---

## Troubleshooting

### Build folder not found
**Error:** `Extraction directory does not exist at /ansible/build/tools_extract`

**Solution:** Run the full playbook first:
```bash
ansible-playbook playbooks/rhel-tools-playbook.yml -v
```

### Tests failing
**Error:** `Some critical files/directories were not extracted properly`

**Solution:** Ensure all tasks completed before testing:
1. Check build directory exists: `ls ansible/build/`
2. Check downloads: `ls ansible/build/tools_download/`
3. Check tarball: `ls ansible/build/tools_download.tar.gz`
4. Re-run full playbook if any are missing

### Download failures
**Error:** Download timeout or connection error

**Solution:**
- Check internet connectivity
- Verify download URLs in `vars/main.yml` are still valid
- Increase timeout value in download tasks

### RPM directories empty
**Warning:** `WARNING: httpd-tools directory exists but appears to be empty`

**Explanation:** This is a warning, not an error. The extraction tasks may not have run, or `rpm2cpio` failed. Check if `rpm2cpio` and `cpio` are installed.

---

## Output Artifacts

After successful execution, you'll have:

```
ansible/build/
├── tools_download/              # Individual downloaded files
│   ├── podman.tar.gz
│   ├── httpd.tar.gz
│   ├── jq
│   ├── *.rpm
│   └── ...
├── tools_extract/               # Extracted source files
│   ├── bin/
│   │   ├── bin/podman-remote-static-linux_amd64
│   │   ├── jq
│   │   └── helm
│   ├── httpd/
│   ├── git/
│   └── ...
└── tools_download.tar.gz        # Combined tarball (~126 MB)
```

---

## Next Steps

1. **Transfer tarball to air-gapped environment:**
   ```bash
   scp ansible/build/tools_download.tar.gz user@airgapped-host:/tmp/
   ```

2. **Extract on target system:**
   ```bash
   tar -xzf /tmp/tools_download.tar.gz -C /opt/tools/
   ```

3. **Install tools** (work in progress - see `install_tools.yml`)

---

## License

SPDX-License-Identifier: MIT-0

---

## Support

For issues or questions, check:
- Task files in `rhel-tools-role/tasks/sub_tasks/`
- Variable definitions in `rhel-tools-role/vars/main.yml`
- Test output with `-vv` flag for detailed information
