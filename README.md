# Ansible Role: alloy

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-alloy)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-alloy)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-alloy)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-alloy/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-alloy/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-alloy/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-alloy/actions/workflows/main.yml?query=branch%3Amain)

Install Grafana Alloy from Grafana's stable package repositories and manage its
telemetry configuration.

## Purpose

Install Grafana Alloy and run it as a systemd service. The default configuration
exposes Alloy's own metrics locally. Set `alloy_config` to a complete native
Alloy configuration to collect and forward metrics, logs or traces.

## Requirements

- The managed host uses systemd and needs access to apt.grafana.com or
  rpm.grafana.com for package installation.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: ansible.posix
    version: '>=2.0.0'
  - name: containers.podman
    version: '>=1.7.0'
```

## Role Variables

### `alloy_backup`

Type: `bool`. Required: `false`.

Back up managed configuration files before replacing them.

Default:

```yaml
alloy_backup: true
```

### `alloy_config`

Type: `str`. Required: `false`.

Complete configuration file in native Alloy syntax.

Default:

```yaml
alloy_config: prometheus.exporter.self "local" {}
```

### `alloy_reporting_enabled`

Type: `bool`. Required: `false`.

Send Alloy's anonymous usage statistics to Grafana Labs.

Default:

```yaml
alloy_reporting_enabled: false
```

## Managed Files

- `/etc/alloy/config.alloy`
- `/etc/systemd/system/alloy.service.d/10-ansible.conf`

## Check Mode

The install entry point supports check mode on new and provisioned hosts. When
package indexes or the Grafana repository are pending, the dry run reports the
required changes and defers package resolution. Use check mode for the full role
after initial provisioning.

## Service Behavior

The role validates the candidate file with `alloy validate` before installing it
and reloads Alloy when it changes. A change to the systemd drop-in reloads the
unit definition and restarts Alloy to apply the new sandbox. Reapplying the same
configuration leaves a running service undisturbed. The service starts on boot.

## Security Notes

- The role disables anonymous usage reporting with `--disable-reporting`. Set
  `alloy_reporting_enabled: true` to opt in. Changing this setting restarts
  Alloy.
- Backups of replaced pipeline files can contain credentials from earlier
  configurations. Set `alloy_backup: false` to disable new backups.
- Root owns the configuration file and the alloy group can read it. Store
  credentials supplied through inventory in Ansible Vault.
- Alloy runs as the alloy user with a read-only system filesystem. Place
  persistent pipeline data under `/var/lib/alloy`, its writable state directory.
- Alloy has private temporary directories and a private device tree. It cannot
  gain new privileges and can open only Unix, IPv4 and IPv6 sockets.

## Operational Notes

- The role uses the package's default configuration path and generally available
  components. The default HTTP endpoint is 127.0.0.1:12345.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Leap | latest | [jomrr/molecule-opensuse-leap:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-leap) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Install Alloy for a separately managed service

Install the package and its account. The caller manages the service.

```yaml
- name: Install Grafana Alloy
  hosts: telemetry
  gather_facts: false
  tasks:
    - name: Install the Alloy package
      ansible.builtin.import_role:
        name: jomrr.alloy
        tasks_from: install
```

### Run Alloy locally

Check readiness with `curl http://127.0.0.1:12345/-/ready`.

```yaml
- name: Run Grafana Alloy
  hosts: telemetry
  gather_facts: false
  roles:
    - role: jomrr.alloy
```

### Forward Alloy metrics to Prometheus

Replace the example URL with a Prometheus-compatible remote write endpoint.

```yaml
- name: Forward collector metrics
  hosts: telemetry
  gather_facts: false
  roles:
    - role: jomrr.alloy
      alloy_config: |
        prometheus.exporter.self "local" {}

        prometheus.scrape "self" {
          targets    = prometheus.exporter.self.local.targets
          forward_to = [prometheus.remote_write.monitoring.receiver]
        }

        prometheus.remote_write "monitoring" {
          endpoint {
            url = "https://prometheus.example.org/api/v1/write"
          }
        }
```

## References

- [Grafana Alloy usage statistics](https://grafana.com/docs/alloy/latest/data-collection/)
- [Install Grafana Alloy on Linux](https://grafana.com/docs/alloy/latest/set-up/install/linux/)
- [Alloy configuration syntax](https://grafana.com/docs/alloy/latest/get-started/configuration-syntax/)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
