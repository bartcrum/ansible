# AWX Validation Pipeline

Automated **pre-change / post-change network validation** for multi-vendor environments managed through AWX.

Captures device state before and after changes, compares snapshots, flags unexpected drift, and produces audit-ready reports.

## Supported Platforms

| Platform | Connection | Validation Type |
|---|---|---|
| Arista EOS | network_cli / eAPI | Full (+ optional AVD mode) |
| Cisco ACI | REST API (APIC) | Full |
| Cisco Catalyst (IOS/IOS-XE) | network_cli | Full |
| VMware NSX-T | REST API | Read-only visibility |

## Quick Start

```bash
# 1. Install required collections
ansible-galaxy collection install -r collections/requirements.yml

# 2. Update inventory with your devices
#    Edit inventories/production/hosts.yml

# 3. Run pre-validation
ansible-playbook playbooks/pre_validate.yml \
  -e change_ticket=CHG0012345

# 4. Make your changes

# 5. Run post-validation
ansible-playbook playbooks/post_validate.yml \
  -e change_ticket=CHG0012345 \
  -e validation_output_dir=/tmp/network_validation/<pre_timestamp>
```

## AWX Setup

```bash
ansible-playbook awx/setup_awx.yml \
  -e awx_host=https://awx.example.com \
  -e awx_token=your_oauth_token \
  -e awx_scm_url=https://git.example.com/netops/awx-validation-pipeline.git
```

This creates the Project, Job Templates, Workflow Template, custom credential types, and surveys in AWX.

## How It Works

```
Pre-Validate  ──►  Execute Change  ──►  Post-Validate & Report
  (baseline)      (manual/automated)      (compare + audit)
```

1. **Pre-Validate** — snapshots device state (show commands, resource modules, REST APIs)
2. **Execute Change** — AWX approval gate or automated change playbook
3. **Post-Validate** — captures new state, diffs against baseline, generates HTML + JSON reports

If `fail_on_drift: true` and changes fall outside `expected_changes`, the workflow **fails** — giving you a hard gate for change control.

## Documentation

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for detailed architecture, usage examples, customization, and security considerations.

## Required Collections

- `ansible.netcommon` >= 5.0.0
- `arista.eos` >= 6.0.0
- `cisco.ios` >= 5.0.0
- `cisco.aci` >= 2.7.0
- `arista.avd` >= 5.0.0 (optional, for AVD validation mode)
- `awx.awx` >= 22.0.0 (for AWX setup playbook)
