# AWX Validation Pipeline — Architecture

## Overview

This framework provides automated **pre-change and post-change validation** for
multi-vendor network environments managed through AWX/Ansible Tower. It captures
device state before and after changes, compares the snapshots, flags unexpected
drift, and produces audit-ready reports.

## Supported Platforms

| Platform | Connection | Managed By | Validation Type |
|---|---|---|---|
| **Arista EOS** | network_cli (eAPI) | Network Team | Full read/write |
| **Cisco ACI** | REST API (APIC) | Network Team | Full read/write |
| **Cisco Catalyst** (IOS/IOS-XE) | network_cli | Network Team | Full read/write |
| **VMware NSX-T** | REST API | Other Team | **Read-only visibility** |

## Architecture

```
AWX Workflow Template
├── Node 1: Pre-Validation (Job Template)
│   ├── validate_common   → create dirs, record metadata
│   ├── validate_arista   → show commands + structured facts
│   ├── validate_cisco_aci → APIC REST API queries
│   ├── validate_cisco_catalyst → show commands + structured facts
│   └── validate_nsx      → NSX-T REST API queries (read-only)
│
├── Node 2: Approval Gate / Change Execution
│   └── AWX Approval Node or external change playbook
│
└── Node 3: Post-Validation & Report (Job Template)
    ├── Same collection roles with validation_phase=post
    ├── validate_common/compare → pre vs post diff per host
    └── Report generation → HTML + JSON audit reports
```

## Workflow Phases

### Phase 1 — Pre-Validation
Captures baseline state across all platforms:
- CLI show commands (text output)
- Structured facts via Ansible resource modules (parsed JSON)
- REST API state from controllers (ACI APIC, NSX Manager)

Outputs: `snapshots/pre_<hostname>.json` per device

### Phase 2 — Execute Change
Either:
- **AWX Approval Node** — operator makes changes manually, then approves
- **Automated** — `change_playbook` extra var points to the change playbook

### Phase 3 — Post-Validation
Same collection as Phase 1, saved as `snapshots/post_<hostname>.json`

### Phase 4 — Compare & Report
- Loads pre and post snapshots per device
- Compares each check key (e.g., `bgp_summary`, `interfaces_status`)
- Marks each as `CHANGED` or `UNCHANGED`
- If `fail_on_drift: true` and changes fall outside `expected_changes`, the workflow **fails**
- Generates HTML and JSON reports for audit/security review

## Directory Structure

```
.
├── ansible.cfg                     # Project-level Ansible config
├── collections/
│   └── requirements.yml            # Required collections
├── inventories/
│   └── production/
│       ├── hosts.yml               # Device inventory
│       └── group_vars/
│           ├── all/main.yml        # Global settings
│           ├── arista/main.yml     # Arista commands & connection
│           ├── cisco_aci/main.yml  # ACI API endpoints
│           ├── cisco_catalyst/main.yml  # Catalyst commands
│           └── nsx/main.yml        # NSX API endpoints
├── roles/
│   ├── validate_common/            # Shared setup + comparison
│   ├── validate_arista/            # Arista EOS collection
│   ├── validate_cisco_aci/         # ACI APIC collection
│   ├── validate_cisco_catalyst/    # Catalyst IOS collection
│   └── validate_nsx/               # NSX-T collection (read-only)
├── playbooks/
│   ├── pre_validate.yml            # Standalone pre-validation
│   ├── post_validate.yml           # Standalone post-validation + report
│   ├── full_validation_workflow.yml # All-in-one orchestration
│   └── examples/                   # Sample change playbooks & manifests
├── templates/
│   └── validation_report.html.j2   # HTML audit report template
├── awx/
│   ├── workflow_template.yml       # AWX workflow/job template defs
│   └── setup_awx.yml              # Playbook to configure AWX objects
└── docs/
    └── ARCHITECTURE.md             # This file
```

## Usage

### CLI — Full Workflow

```bash
# Install collections
ansible-galaxy collection install -r collections/requirements.yml

# Run the full workflow (with manual pause for changes)
ansible-playbook playbooks/full_validation_workflow.yml \
  -i inventories/production/hosts.yml \
  -e change_ticket=CHG0012345

# Or with an automated change playbook
ansible-playbook playbooks/full_validation_workflow.yml \
  -i inventories/production/hosts.yml \
  -e change_ticket=CHG0012345 \
  -e change_playbook=path/to/change.yml \
  -e "expected_changes=['bgp_summary','route_summary']"
```

### CLI — Separate Pre/Post

```bash
# Step 1: Pre-validate
ansible-playbook playbooks/pre_validate.yml \
  -i inventories/production/hosts.yml \
  -e change_ticket=CHG0012345

# Step 2: Make your changes (manual or automated)

# Step 3: Post-validate (use the same output dir from pre)
ansible-playbook playbooks/post_validate.yml \
  -i inventories/production/hosts.yml \
  -e change_ticket=CHG0012345 \
  -e validation_output_dir=/tmp/network_validation/<timestamp>
```

### AWX — Setup

```bash
# Configure AWX with job templates, workflow, and credential types
ansible-playbook awx/setup_awx.yml \
  -e awx_host=https://awx.example.com \
  -e awx_token=your_oauth_token \
  -e awx_scm_url=https://git.example.com/netops/network-validation.git
```

### AWX — Running the Workflow

1. Navigate to **Templates → Network Change Validation Workflow**
2. Click **Launch**
3. Fill in the survey:
   - **Change Ticket**: Your ITSM ticket number (e.g., `CHG0012345`)
   - **Fail on Unexpected Drift**: `true` (recommended)
   - **Expected Changes**: comma-separated check keys you expect to change
4. Pre-validation runs automatically
5. **Approval Node** pauses the workflow — make your changes, then approve
6. Post-validation runs, compares, and generates the report
7. Reports are available in the job artifacts

## Drift Detection

The `fail_on_drift` mechanism provides a hard gate for change control:

```yaml
# In extra_vars or survey:
fail_on_drift: true
expected_changes:
  - bgp_summary      # We expect BGP to change
  - route_summary     # We expect routes to change

# If post-validation detects changes OUTSIDE this list → WORKFLOW FAILS
# This forces review before the change is considered complete
```

## Report Output

### HTML Report
- Professional, print-friendly layout for audit review
- Summary cards (hosts validated, checks passed/changed)
- Per-host expandable sections with diff details
- Timestamped with change ticket reference

### JSON Report
- Machine-readable for integration with ITSM/CMDB tools
- Complete pre/post snapshots and diff data
- Can be ingested by ServiceNow, Splunk, or ELK

## Customization

### Adding New Validation Checks

Edit the platform group_vars to add commands:

```yaml
# inventories/production/group_vars/arista/main.yml
arista_validation_commands:
  - command: "show ip bgp summary"
    key: "bgp_summary"
    description: "BGP neighbor state"
  # Add your new check:
  - command: "show vxlan vtep"
    key: "vxlan_vteps"
    description: "VXLAN tunnel endpoints"
```

### Adding a New Platform

1. Create a new role: `roles/validate_<platform>/`
2. Add group_vars: `inventories/production/group_vars/<platform>/main.yml`
3. Add hosts to inventory under a new group
4. Add a play section to each playbook targeting the new group

### Targeting Specific Hosts

Use `--limit` to scope validation:

```bash
ansible-playbook playbooks/pre_validate.yml --limit arista
ansible-playbook playbooks/pre_validate.yml --limit cat-edge-rtr-01
```

## Security Considerations

- **Credentials**: Never store in plain text. Use AWX credential types or Ansible Vault.
- **NSX Read-Only**: NSX role only performs GET requests — no write operations.
- **Report Access**: Reports may contain sensitive network state. Store in access-controlled locations.
- **ACL Auditing**: Catalyst validation captures `show ip access-lists` for security review.
- **Fault Visibility**: ACI validation captures critical faults for awareness.
