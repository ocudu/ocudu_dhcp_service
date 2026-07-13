<!--
SPDX-FileCopyrightText: Copyright (C) 2021-2026 Software Radio Systems Limited
SPDX-FileCopyrightText: Modifications (C) 2026 OpenInfra Foundation Europe. All rights reserved.
SPDX-License-Identifier: BSD-3-Clause-Open-MPI
-->

# OCUDU DHCP Service

A Docker container running [ISC Kea](https://www.isc.org/kea/) DHCPv4 on Alpine
Linux. It hands out IP addresses to O-RAN radio units and tells them where their
M-Plane controller lives, using DHCP Option 43.

## Quick start

Build and run:

```sh
docker compose -f docker-compose.dhcp.yml up
```

Build only:

```sh
docker compose -f docker-compose.dhcp.yml build
```

The container runs with `network_mode: host` because DHCP needs direct access to
the host's network interfaces for broadcast traffic. No port mapping required.

## How it works

`entrypoint.sh` runs at container start and does five things:

1. Loads `/etc/kea/dhcp_server.env` (all the configuration variables)
2. Runs `generate_option43_hex.sh` (computes the O-RAN TLV payloads)
3. Writes `etc/kea/subnets4.json` (subnet definition, address pool, DHCP options, reservations)
4. Writes `etc/kea/kea-dhcp4.conf` (main Kea config, references the subnet file)
5. `exec`s into `kea-dhcp4` (replaces the shell process with the DHCP daemon)

Steps 3 and 4 use relative paths (`etc/kea/...` not `/etc/kea/...`). This works
because no `WORKDIR` is set in the Dockerfile, so CWD defaults to `/`. The
relative path resolves to the absolute one by coincidence of the container's
default working directory.

The generated Kea config defines two client classes. When an O-RU sends a
DHCPDISCOVER, Kea checks its Vendor Class Identifier (Option 60) and responds
with the matching Option 43 payload:

| Client class | Option 60 prefix | Which spec |
|---|---|---|
| `ORAN-RU2` | `o-ran-ru2/` | Current O-RAN WG4 |
| `ORAN-RU-LEGACY` | `o-ran-ru/` | Legacy |

## DHCP Option 43: O-RAN vendor information

Option 43 is the "vendor-specific information" field in DHCP. O-RAN uses it to
pass M-Plane bootstrap parameters to the radio unit as TLV (Type-Length-Value)
encoded hex. `generate_option43_hex.sh` builds these payloads from environment
variables.

### RU2 payload (current spec)

| Tag | Hex | Length | What it carries |
|---|---|---|---|
| Controller IP | `0x81` | 4 bytes (fixed) | IPv4 address of the M-Plane controller |
| Controller FQDN | `0x82` | Variable | ASCII hostname of the M-Plane controller |
| Call-home transport | `0x86` | 1 byte | `0x00` = NETCONF/SSH, `0x01` = NETCONF/TLS |

### Legacy payload (older firmware)

| Tag | Hex | Length | What it carries |
|---|---|---|---|
| Controller IP | `0x01` | 4 bytes (fixed) | IPv4 address of the M-Plane controller |
| Controller FQDN | `0x02` | Variable | ASCII hostname of the M-Plane controller |

Which payload gets sent depends on what the O-RU puts in its Option 60 field
during DHCPDISCOVER. If it starts with `o-ran-ru2/`, Kea sends the RU2 payload.
If it starts with `o-ran-ru/`, it gets the legacy one.

## Environment variables

Defined in `ocudu_dhcp/server/dhcp_server.env`. Baked into the image at build
time (there is no `env_file:` in the compose file, so you cannot override these
at runtime without rebuilding).

### Network and lease settings

| Variable | Default | What it does |
|---|---|---|
| `ETH_4` | `*` | Interface(s) Kea listens on (`*` = all) |
| `RENEW_TIMER_4` | `1000` | DHCP T1 renewal timer (seconds) |
| `REBIND_TIMER_4` | `2000` | DHCP T2 rebind timer (seconds) |
| `VALID_LIFETIME_4` | `4000` | Lease valid lifetime (seconds) |

### Subnet settings

| Variable | Default | What it does |
|---|---|---|
| `SUBNET_IP_4` | `192.168.50.0/24` | Managed subnet (CIDR) |
| `POOL_4` | `192.168.50.10-192.168.50.99` | Dynamic address pool (90 IPs) |
| `ROUTER_IP_4` | `0.0.0.0` | Default gateway sent to clients |
| `DNS_IP_4` | `192.168.50.53` | DNS server sent to clients |
| `DOMAIN_NAME_4` | `oran.lab` | Domain name sent to clients |
| `NTP_SERVERS_4` | `192.168.50.123` | NTP server sent to clients |

### Static reservation

| Variable | Default | What it does |
|---|---|---|
| `MAC_ADDRESS_RESERVATION_4` | `aa:bb:cc:dd:ee:01` | MAC address for the static lease |
| `IP_ADDRESS_RESERVATION_4` | `192.168.50.11` | IP address for the static lease |

### O-RAN Option 43 settings

| Variable | Default | What it does |
|---|---|---|
| `RU_CONTROLLER_IP_ADDRESS` | `192.255.255.255` | M-Plane controller IPv4 address |
| `RU_CONTROLLER_FQDN` | `testcontroller.oran.example` | M-Plane controller hostname |
| `CALLHOME_SSH_OR_TLS` | `0x00` | Call-home transport (`0x00`=SSH, `0x01`=TLS) |

These are lab defaults. For a real deployment, override at minimum `SUBNET_IP_4`,
`POOL_4`, `ROUTER_IP_4`, `RU_CONTROLLER_IP_ADDRESS`, and `RU_CONTROLLER_FQDN`.

## Repository layout

```
docker-compose.dhcp.yml              Compose file (builds and runs the container)
ocudu_dhcp/server/
  ├── Dockerfile                     Alpine 3.22 + ISC Kea 3.0
  ├── entrypoint.sh                  Config generator and Kea launcher
  ├── generate_option43_hex.sh       O-RAN TLV payload encoder
  └── dhcp_server.env                Default environment variables
.gitlab-ci.yml                       CI pipeline (header checks, YAML lint)
.gitlab/main.tf                      Terraform GitLab project settings
```

## Docker image

Base image is Alpine 3.22. Kea 3.0 packages come from
[ISC's Cloudsmith repo](https://cloudsmith.io/~isc/repos/):

- `isc-kea-dhcp4` (the DHCPv4 daemon)
- `isc-kea-hooks` (open-source hook libraries)
- `isc-kea-mysql` (MySQL hook library)
- `isc-kea-pgsql` (PostgreSQL hook library)

The lease database uses Kea's memfile backend (`kea-leases4.csv`), persisted
across container restarts via the `kea4-var` Docker volume mounted at
`/var/lib/kea`.

Build-time dependencies (`curl`, `bash`) are removed after Kea installation.
The running image only has `ash` (Alpine's default shell).

## CI pipeline

`.gitlab-ci.yml` pulls shared CI definitions from the main `ocudu/ocudu`
repository. Locally it defines:

| Stage | Job | Trigger | What it checks |
|---|---|---|---|
| `ci` | `headers` | Every pipeline | REUSE/SPDX header compliance |
| `static` | `yaml lint` | MR only (when YAML changed) | yamllint against `.yamllint` rules |
| `build` | (none) | — | Stage declared, no jobs defined here |

## License

BSD 3-Clause Open MPI variant. See [LICENSE](./LICENSE).

Portions of this software may implement 3GPP specifications, which may be
subject to additional licensing requirements.
