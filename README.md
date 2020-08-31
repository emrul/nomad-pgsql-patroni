# nomad-pgsql-patroni

A simple container running Postgres and Patroni useful for dropping directly into a Hashicorp environment (Nomad + Consul + Vault)

It also contains some helpers for ongoing maintenance

- **awscli**<br />
  So the same container image can be used in backup jobs
- **WAL-G 0.2.15**<br />
  See here for more info - https://github.com/wal-g/wal-g
- **TimescaleDB 1.7.3**<br />
  See here for more info - https://github.com/timescale/timescaledb
- **PostGIS 3.0.2**
  See here for more info - https://postgis.net/
- **pgRouting 3.02**
  See here for more info - https://pgrouting.org/

### Running Postgres 12?

See the [`master`](https://github.com/ccakes/nomad-pgsql-patroni) branch for the latest version.

## Usage
```hcl
# main.tf
resource "nomad_job" "postgres" {
  jobspec = "${file("${path.module}/job.hcl")}"
}

# job.hcl
job "your-task" {
  type = "service"
  dataceners = ["default"]

  vault { policies = ["postgres"] }

  group "your-group" {
    count = 3

    task "db" {
      driver = "docker"

      template {
        data <<EOL
scope: postgres
name: pg-{{env "node.unique.name"}}
namespace: /nomad

restapi:
  listen: 0.0.0.0:{{env "NOMAD_PORT_api"}}
  connect_address: {{env "NOMAD_ADDR_api"}}

consul:
host: consul.example.com
token: {{with secret "consul/creds/postgres"}}{{.Data.token}}{{end}}
register_service: true

# bootstrap config
EOL
      }

      config {
        image = "ccakes/nomad-pgsql-patroni:11.9-1.tsdb_gis"

        port_map {
          pg = 5432
          api = 8008
        }
      }

      resources {
        memory = 1024

        network {
          port "api" {}
          port "pg" {}
        }
      }
    }
  }
}
```

## ISSUES

Postgres runs as the postgres user however that user has been added to the root group. This probably has some security ramifications that I haven't thought of, but it's required for postgres to read TLS keys generated by Vault and written as templates.

[hashicorp/nomad#5020](https://github.com/hashicorp/nomad/issues/5020) is tracking (hopefully) a fix for this.
