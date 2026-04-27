# AWX Golden Config Rendering

This AWX project renders the IOS-XE access-switch golden config from NetBox-native data plus NetBox `config_context`.

Current status:

- project sync in AWX: working
- render job in AWX: working
- runtime secret injection in AWX: working
- direct deployment to switch: scaffolded

## Project Layout

- `playbooks/render_iosxe_access_switch_golden_config.yml`
- `playbooks/deploy_iosxe_access_switch_golden_config.yml`
- `roles/netbox_golden_config_render/`
- `roles/iosxe_golden_config_deploy/`
- `inventories/localhost.yml`
- `collections/requirements.yml`

## Required AWX Variables

Provide these as AWX credential-backed environment variables or extra vars:

- `netbox_api_url`
- `netbox_api_token`
- `golden_config_target_device`

Required runtime secret environment variables:

- `GOLDEN_CONFIG_MASTER_KEY`
- `GOLDEN_CONFIG_TACACS_SECRET`
- `GOLDEN_CONFIG_RADIUS_KEY`
- `GOLDEN_CONFIG_LOCAL_ADMIN_SECRET`
- `GOLDEN_CONFIG_ENABLE_SECRET`
- `GOLDEN_CONFIG_SNMPV3_AUTH_PASSWORD`
- `GOLDEN_CONFIG_SNMPV3_PRIV_PASSWORD`
- `GOLDEN_CONFIG_SMART_LICENSE_IDTOKEN`

Optional variables:

- `golden_config_template_path`
- `golden_config_render_output_dir`
- `golden_config_render_context_output_dir`
- `golden_config_write_render_context`

## Execution Flow

1. Query NetBox for the target device
2. Retrieve the device interfaces, site VLANs, primary management IP, and management prefix
3. Read the default gateway from NetBox IPAM on the tagged management prefix
4. Build a VLAN role map from native VLAN objects using the `golden_vlan_role` custom field
5. Resolve logical VLAN role references stored in `config_context`
6. Build the final render payload expected by the Jinja template
7. Override NetBox placeholder secrets with AWX-injected runtime secrets
8. Fail closed if any required secret is still unresolved
9. Render the final configuration to a file

Current source-interface behavior:

- The template uses `source_interface.name` for TACACS, RADIUS, DNS lookup,
  HTTP client, SSH, logging, SNMP trap-source, and NTP source commands.
- That value is currently derived in render logic from the NetBox interface
  tagged with `golden_port_profile = oob_management`.
- For the pilot switch, this resolves to `GigabitEthernet1/0/1`.
- If you later want those commands to source from in-band `Vlan2`, change the
  render payload assembly in `roles/netbox_golden_config_render/tasks/main.yml`.

## NetBox IPAM Assumptions

The role expects:

- exactly one management prefix per site tagged `management-subnet`
- that prefix to have a custom field named `default_gateway`

This keeps the render workflow independent of `netaddr` or execution-environment-specific IP math libraries.

## Security Notes

- Do not store live secrets in NetBox `config_context`.
- Do not store live secrets in project files.
- The render role now uses `no_log: true` for secret-sensitive NetBox and runtime-secret tasks.
- NetBox placeholder secrets such as `__FROM_AAP__` are expected to be replaced by AWX runtime credential injection before deployment use.

## Expected AWX Job Template

Recommended job template name:

- `Render IOS-XE Access Switch Golden Config`

Recommended inventory:

- `localhost`

Recommended AWX credentials attached to the job template:

- `NetBox Golden Config API`
- `Golden Config Runtime Secrets`

For the deployment job template, also attach:

- a Cisco network credential for the switch login

Recommended extra vars:

```yaml
golden_config_target_device: par-acc-sw-01
netbox_api_url: http://172.17.152.181
```

Do not store `netbox_api_token` in plain project files.

## Current Validation

Latest verified state:

- AWX render job completes successfully
- uplink member rendering resolves correctly
- rendered payload contains runtime-injected secrets instead of `__FROM_AAP__`

Current temporary limitation:

- the attached runtime secret credential is using dummy values and must be replaced with real values before any deployment to a switch

## Deployment Workflow

Deployment playbook:

- `playbooks/deploy_iosxe_access_switch_golden_config.yml`

Deployment flow:

1. Render the config on `localhost`
2. Confirm the rendered file exists
3. Connect to the target IOS-XE switch with `network_cli`
4. Back up the current running config
5. Push the rendered golden config
6. Save the config
7. Run basic post-check commands and store the output

Required collections for deployment:

- `ansible.netcommon`
- `cisco.ios`

Recommended deployment inventory pattern:

- inventory group: `iosxe_access_switches`
- host vars should provide the target switch connectivity expected by your AWX network credential
