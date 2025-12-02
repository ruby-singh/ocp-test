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
cd /Users/ruby/Desktop/Code/ocp-test/ansible && ansible-playbook playbooks/rhel-tools-playbook.yml -v
```

Runs all tasks: download → create tar → copy → extract → test

---

### 2. Run Only Tests (Using Tags)

```bash
cd /Users/ruby/Desktop/Code/ocp-test/ansible && ansible-playbook playbooks/rhel-tools-playbook.yml --tags test -v
```

Runs only test validation tasks. **Note:** Requires build folder and files to already exist from running the full playbook first.

---

### 3. Run Standalone Test Playbook

```bash
cd /Users/ruby/Desktop/Code/ocp-test/ansible && ansible-playbook playbooks/test-tar-operations.yml -v
```

Alternative way to run only tests with inline variables.

---

## Test Coverage

The test suite validates:

### ✅ Test 1-2: Tar File Creation & Copy
- Combined tar file exists on control node
- Tar file copied to `/tmp/tools_download.tar.gz`
- File sizes match between local and remote

### ✅ Test 3-4: Extraction Validation
- Extraction directory exists at `build/tools_extract/`
- Critical binaries extracted (podman, jq, helm)
- Source directories extracted (httpd, git, curl, etc.)

### ✅ Test 5: Tar Integrity
- Tar file can be read without corruption
- Archive contains files (not empty)

### ✅ Test 6: Downloaded Files Validation
- All files are valid gzip archives or binaries
- Files are not corrupt (proper file types)
- Files meet minimum size requirements

### ✅ Test 7: Extracted Files Integrity
- All source directories present
- RPM extraction directories exist
- Warns if RPM directories are empty

### ✅ Test 8: Version Detection
- Extracts version numbers from URLs
- Displays versions for all tools
- Validates version format

### ✅ Test 9: Archive Contents
- Counts files in each tarball
- Verifies tarballs contain expected data

### ✅ Test 10: Checksums
- Generates SHA256 checksums for verification
- Useful for supply chain validation

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
