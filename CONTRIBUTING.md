<!--
SPDX-FileCopyrightText: Copyright (C) 2026 OCUDU contributors
SPDX-License-Identifier: BSD-3-Clause-Open-MPI
-->

# Contributing guidelines

Welcome! We're glad you're interested in contributing to the OCUDU DHCP Service.

The project accepts contributions via GitLab merge requests. For broader OCUDU contribution guidance, see the [Developer Guide](https://docs.ocudu.org/dev_guide/).

By participating, you are expected to follow our [Code of Conduct](./CODE_OF_CONDUCT.md).

## Getting started

1. Fork the repository on GitLab.
2. Create a feature branch from `main`.
3. Make your changes.
4. Ensure all SPDX headers are correct (the CI checks this automatically).
5. Open a merge request against `main`.

## What we look for in MRs

- Descriptive commit messages following [conventional commits](https://www.conventionalcommits.org/) style (e.g. `fix:`, `docs:`, `feat:`).
- One logical change per MR.
- Shell scripts pass [shellcheck](https://www.shellcheck.net/) without warnings.
- YAML passes yamllint (CI runs this on MR).
- SPDX/REUSE headers present on every file. New files use `SPDX-FileCopyrightText: Copyright (C) <year> OCUDU contributors`; files derived from existing SRS-authored code keep the original Software Radio Systems Limited line and add the OCUDU contributors line. If your organization is contributing copyrightable work for the first time, add it to [`CONTRIBUTORS.md`](./CONTRIBUTORS.md) (alphabetical).

## Development setup

The repo is small — four functional files and a compose config. To test locally:

```bash
# Build the image
docker compose -f docker-compose.dhcp.yml build

# Run
docker compose -f docker-compose.dhcp.yml up
```

No additional tooling is required. If you want to lint locally before pushing:

```bash
# Shell lint
shellcheck ocudu_dhcp/server/entrypoint.sh ocudu_dhcp/server/generate_option43_hex.sh

# YAML lint
yamllint -c .yamllint .
```

## Reporting issues

Use [GitLab issues](https://gitlab.com/nordixinfra-group/ocudu/ocudu_elements/ocudu_oran_apps/ocudu_dhcp_service/-/issues) for bug reports and feature requests.

## Licensing

This project is licensed under BSD-3-Clause-Open-MPI. By submitting a merge request you agree that your contributions will be licensed under the same terms.

Any contribution requiring a patent license beyond what is already required under relevant 3GPP standards must be disclosed with the contribution. Contributions requiring additional license requirements must be approved by the TSC prior to acceptance.
