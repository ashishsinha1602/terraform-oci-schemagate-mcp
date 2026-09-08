# terraform-oci-schemagate-mcp

Terraform module and OCI Resource Manager stack for
[schemagate](https://github.com/ashishsinha1602/schemagate) — identity-scoped
schema selection for text-to-SQL.

It stands up an Always-Free-eligible VM running `python -m schemagate.mcp_server`
over streamable HTTP against an Autonomous Database it creates, or one you
already have, and points any MCP client at it.

[![Deploy to Oracle Cloud](https://oci-resourcemanager-plugin.plugins.oci.oraclecloud.com/latest/deploy-to-oracle-cloud.svg)](https://cloud.oracle.com/resourcemanager/stacks/create?zipUrl=https://github.com/ashishsinha1602/schemagate/releases/latest/download/schemagate-oci-stack.zip)

## What schemagate does

Given a question and a caller identity it returns the handful of tables and views
the model needs, with every object that caller may not read removed *before*
ranking — so a restricted table never reaches the prompt at all.

That ordering is the point. Schema selection happens before the query runs, which
is before row-level security can act. If selection is not identity-aware, the
model is handed a table the caller cannot read, writes correct SQL, RLS filters
every row, and the user sees "no records found" and believes it.

Certified live on Oracle AI Database 26ai.

## Usage

```hcl
module "schemagate" {
  source  = "ashishsinha1602/schemagate-mcp/oci"
  version = "0.1.5"

  tenancy_ocid     = var.tenancy_ocid
  compartment_ocid = var.compartment_ocid
  region           = "us-phoenix-1"

  create_adb         = true
  adb_admin_password = var.adb_admin_password
  ssh_public_key     = file("~/.ssh/id_rsa.pub")

  # Descriptions written by OCI Generative AI on first boot, through the
  # instance principal. No API key, and no prompt leaves your tenancy.
  catalog_provider = "oci"
  catalog_model    = "google.gemini-2.5-pro"

  allowed_cidr = "203.0.113.0/24" # narrow this
}

output "mcp_url" {
  value = module.schemagate.mcp_url
}
```

Then point any MCP client at the `mcp_url` output:

```json
{"mcpServers": {"schemagate": {"url": "http://<public-ip>:8765/mcp"}}}
```

## Inputs

| name | description | default |
|---|---|---|
| `tenancy_ocid` | Tenancy OCID | — |
| `compartment_ocid` | Compartment for the instance, network and database | — |
| `region` | OCI region | — |
| `create_adb` | Create an Always Free Autonomous Database (23ai) | `true` |
| `adb_admin_password` | ADMIN password when `create_adb` is true | `""` |
| `database_url` | SQLAlchemy URL of your own database when `create_adb` is false | `""` |
| `catalog_provider` | `oci` to catalogue with OCI Generative AI on first boot, or `none` | `oci` |
| `catalog_model` | OCI Generative AI model id | `google.gemini-2.5-pro` |
| `ssh_public_key` | SSH public key for the `opc` user | — |
| `instance_shape` | Compute shape | `VM.Standard.E2.1.Micro` |
| `allowed_cidr` | Who may reach the MCP endpoint | `0.0.0.0/0` |
| `mcp_port` | MCP port | `8765` |

## Outputs

| name | description |
|---|---|
| `mcp_url` | Point Claude Desktop, Cursor or any MCP client here |
| `ssh` | SSH command for the instance |
| `catalogued_with` | Which model wrote the table descriptions |

## Cataloguing, keylessly

With `catalog_provider = "oci"` the stack creates a dynamic group and a policy
scoped to this one instance, so it can call OCI Generative AI through its
instance principal. Descriptions are written with no API key stored anywhere,
and the prompts — schema metadata only, never rows — stay in your tenancy.

Measured blind across six schemas, descriptions take business-language recall
from about 56% on identifiers alone to about 92%. Re-run it any time:

```bash
sudo -u opc OCI_CLI_AUTH=instance_principal /opt/catalog-once.sh
```

## Not using Terraform?

`pip install --user 'schemagate[oracle,oci]'` in OCI Cloud Shell catalogues an
existing Autonomous Database in about a minute with no VM at all. See
[`oci/README.md`](https://github.com/ashishsinha1602/schemagate/blob/main/oci/README.md).

## Security

The MCP endpoint has no authentication of its own — identity comes from the
`principal` each call passes. Narrow `allowed_cidr` before pointing anything
real at it, and destroy the stack to remove everything it created.

## License

Apache-2.0. Copyright Ashish Sinha.
