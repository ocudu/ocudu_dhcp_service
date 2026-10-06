<!--
SPDX-FileCopyrightText: Copyright (C) 2021-2026 Software Radio Systems Limited
SPDX-FileCopyrightText: Modifications (C) 2026 OpenInfra Foundation Europe. All rights reserved.
SPDX-License-Identifier: BSD-3-Clause-Open-MPI
-->

# OCUDU DHCP Service

[![License](https://img.shields.io/badge/license-BSD--3--Clause--Open--MPI-blue)](https://spdx.org/licenses/BSD-3-Clause-Open-MPI.html)

Dockerised [ISC Kea](https://www.isc.org/kea/) DHCPv4 server for O-RAN radio unit provisioning. Hands out IPv4 addresses and delivers M-Plane bootstrap parameters to O-RUs via DHCP Option 43.

Part of the [OCUDU project](https://ocudu.org), governed under the Linux Foundation.

## Overview

This service is a single-purpose DHCPv4 server for an O-RAN fronthaul network. It does three things:

1. **Leases IPv4 addresses** to devices on a managed subnet from a configurable dynamic pool, plus a static reservation that always maps a known MAC address to a fixed IP.
2. **Provides standard DHCP configuration options,**  including the default gateway, DNS server, domain name, and NTP server, to clients on the subnet.
3. **Bootstraps O-RAN radio units (O-RUs)** by recognising their DHCP Option 60 (Vendor Class Identifier) and replying with a matching Option 43 payload, encoded as O-RAN TLV hex. The server supports two O-RU variants: current RU2 units (Option 60 `o-ran-ru2/`) receive the controller's IP address, FQDN, and call-home transport; legacy units (Option 60 `o-ran-ru/`) receive the controller's IP address and FQDN only (no call-home transport). Either way, this is what lets a freshly powered O-RU discover and connect back to its M-Plane management controller.

The server is ISC Kea's `kea-dhcp4` daemon running in an Alpine container. All behaviour is driven by a single environment file (`dhcp_server.env`) that is read when the container starts; a shell entrypoint turns those variables into Kea's JSON config and the Option 43 hex payloads before launching the daemon. The lease store is Kea's flat-file memfile backend — there is no database.


## Quick start

```bash
docker compose -f docker-compose.dhcp.yml up
```

Build only (no run):

```bash
docker compose -f docker-compose.dhcp.yml build
```

The container uses `network_mode: host` because DHCP requires direct access to the host's network interfaces for broadcast traffic.

## How it works

At container start, `entrypoint.sh` runs through these steps:

1. Sources `/etc/kea/dhcp_server.env` (configuration variables)
2. Sources `generate_option43_hex.sh` (computes O-RAN TLV payloads)
3. Writes `/etc/kea/subnets4.json` (subnet, pool, DHCP options, reservations)
4. Writes `/etc/kea/kea-dhcp4.conf` (main Kea config referencing the subnet file)
5. `exec`s `kea-dhcp4` (replaces the shell with the DHCP daemon)

Kea then listens for DHCPDISCOVER packets. When an O-RU identifies itself via Option 60 (Vendor Class Identifier), Kea responds with the matching Option 43 payload:

| Client class | Option 60 match | Spec |
|---|---|---|
| `ORAN-RU2` | starts with `o-ran-ru2/` | Current O-RAN WG4 |
| `ORAN-RU-LEGACY` | starts with `o-ran-ru/` | Legacy |

## DHCP Option 43

O-RAN uses Option 43 to pass M-Plane bootstrap parameters to radio units as TLV (Type-Length-Value) encoded hex. `generate_option43_hex.sh` builds these payloads.

### RU2 payload (current spec)

| Tag | Hex | Length | Content |
|---|---|---|---|
| Controller IP | `0x81` | 4 bytes | IPv4 address of the M-Plane controller |
| Controller FQDN | `0x82` | variable | ASCII hostname of the M-Plane controller |
| Call-home transport | `0x86` | 1 byte | `0x00` = SSH, `0x01` = TLS |

### Legacy payload

| Tag | Hex | Length | Content |
|---|---|---|---|
| Controller IP | `0x01` | 4 bytes | IPv4 address of the M-Plane controller |
| Controller FQDN | `0x02` | variable | ASCII hostname of the M-Plane controller |

## Configuration

All configuration lives in `ocudu_dhcp/server/dhcp_server.env`. This file is baked into the image at build time (copied in by the Dockerfile) and read by `entrypoint.sh` at container start, which generates the Kea config from it. Because the defaults are baked in and the compose file wires up no runtime override (no bind mount or `environment:` block), changing these values means editing `dhcp_server.env` and rebuilding the image.

### Network and leases

| Variable | Default | Description |
|---|---|---|
| `ETH_4` | `*` | Interface(s) Kea listens on (`*` = all) |
| `RENEW_TIMER_4` | `1000` | T1 renewal timer (seconds) |
| `REBIND_TIMER_4` | `2000` | T2 rebind timer (seconds) |
| `VALID_LIFETIME_4` | `4000` | Lease valid lifetime (seconds) |

### Subnet

| Variable | Default | Description |
|---|---|---|
| `SUBNET_IP_4` | `192.168.50.0/24` | Managed subnet (CIDR) |
| `POOL_4` | `192.168.50.10-192.168.50.99` | Dynamic address pool (90 IPs) |
| `ROUTER_IP_4` | `0.0.0.0` | Default gateway sent to clients |
| `DNS_IP_4` | `192.168.50.53` | DNS server |
| `DOMAIN_NAME_4` | `oran.lab` | Domain name |
| `NTP_SERVERS_4` | `192.168.50.123` | NTP server |

### Static reservation

| Variable | Default | Description |
|---|---|---|
| `MAC_ADDRESS_RESERVATION_4` | `aa:bb:cc:dd:ee:01` | Reserved MAC address |
| `IP_ADDRESS_RESERVATION_4` | `192.168.50.11` | Reserved IP address |

### O-RAN Option 43

| Variable | Default | Description |
|---|---|---|
| `RU_CONTROLLER_IP_ADDRESS` | `192.255.255.255` | M-Plane controller IPv4 |
| `RU_CONTROLLER_FQDN` | `testcontroller.oran.example` | M-Plane controller hostname |
| `CALLHOME_SSH_OR_TLS` | `0x00` | Call-home transport (`0x00` = SSH, `0x01` = TLS) |

These are lab defaults. For a real deployment, override at minimum: `SUBNET_IP_4`, `POOL_4`, `ROUTER_IP_4`, `RU_CONTROLLER_IP_ADDRESS`, and `RU_CONTROLLER_FQDN`.

## Repository layout

```
docker-compose.dhcp.yml              Compose file (build + run)
ocudu_dhcp/server/
├── Dockerfile                       Alpine 3.22 + ISC Kea 3.0
├── entrypoint.sh                    Config generator and Kea launcher
├── generate_option43_hex.sh         O-RAN TLV payload encoder
└── dhcp_server.env                  Default environment variables
.gitlab-ci.yml                       CI pipeline (SPDX headers, YAML lint)
.gitlab/main.tf                      Terraform GitLab project settings
```

## Docker image

Base: Alpine 3.22. Kea 3.0 packages from [ISC's Cloudsmith repository](https://cloudsmith.io/~isc/repos/):

- `isc-kea-dhcp4` — DHCPv4 daemon
- `isc-kea-hooks` — open-source hook libraries
- `isc-kea-mysql` — MySQL hook library
- `isc-kea-pgsql` — PostgreSQL hook library

Lease database uses Kea's memfile backend (`kea-leases4.csv`), persisted via the `kea4-var` volume at `/var/lib/kea`.

## CI pipeline

The `.gitlab-ci.yml` pulls shared CI definitions from the main [ocudu/ocudu](https://gitlab.com/ocudu/ocudu) repository.

| Stage | Job | Trigger | What it checks |
|---|---|---|---|
| `ci` | `headers` | Every pipeline | REUSE/SPDX header compliance |
| `static` | `yaml lint` | MR (when YAML changed) | yamllint |
| `build` | — | — | Stage declared, no jobs yet |

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).

## License

BSD 3-Clause Open MPI variant — see [LICENSE](./LICENSE).

Portions of this software may implement 3GPP specifications, which may be subject to additional licensing requirements.
