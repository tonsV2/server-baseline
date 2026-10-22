# Changelog

All notable changes to the `sre.server` collection.

The collection follows [semantic versioning](https://semver.org). A breaking change to a role's
variables or to what it does to a host is a major version, so consumers pinning `v1.x` can take
patches without rereading their group_vars.

## 1.0.1-k3s.1

The first release of the tonsV2 fork: upstream's unreleased fixes after 1.0.1, plus the k3s role.
Fork releases are tagged `v<upstream version>-k3s.<n>`.

### Fixed

- `firewall` inserts its allow rules at the top of `DOCKER-USER` instead of appending them, so a port
  added to `firewall_allowed_ports` or `firewall_allowed_udp_ports` on a later run lands ahead of the
  default drop rather than after it, where it never matched (#4).
- `bootstrap` refreshes the apt cache only when it is more than an hour old, instead of on every run,
  so a repeat run no longer reports its "Update apt cache" task as changed (#7).
- `harden` replaces an existing `/etc/subuid` or `/etc/subgid` entry for `docker_user` instead of
  adding a second one beside it. `useradd` writes an entry for each new user, and wherever its range
  was not `100000:65536` the role left the user with two ranges (#5).
- `harden` merges its Docker daemon settings (`userns-remap`, `icc`, `log-driver`, `log-opts`) into
  `/etc/docker/daemon.json` instead of replacing the file, so settings it does not manage, such as
  `data-root` or `registry-mirrors`, survive a run (#6). The hardening keys still win. Existing
  `log-opts` are kept when the driver was already `json-file`, and dropped otherwise, because
  dockerd will not start with options `json-file` does not know. A file that is not a JSON object
  now stops the role instead of being overwritten.
- `harden` writes its kernel parameters to `/etc/sysctl.d/99-sre-server.conf` instead of
  `/etc/sysctl.conf`, which Ubuntu 26.04 no longer reads at boot, so they were applied once and lost
  on the next reboot: IPv6 came back on, with `rp_filter`, `log_martians` and `suid_dumpable` back
  at their defaults.
- `harden` no longer restarts fail2ban on every run. It ensures the service is enabled and running,
  and restarts it through a handler only when installing it changed something (#8).

### Changed

- The first run after upgrading rewrites `daemon.json` with sorted keys, so Docker restarts once on
  each host even where the settings are unchanged.

### Added

- `sre.server.k3s`, a single node k3s server beside Docker. It filters new connections on the public
  interface, IPv4 and IPv6, to the host's own addresses on the firewall's listed ports, before any
  port is rewritten, which k3s's network policy controller would otherwise get around; lets pod
  traffic past the `DOCKER-USER` drop; keeps k3s's rules out of the restore at boot, which they
  would otherwise make fail; and enforces Pod Security Admission. It is not part of
  `sre.server.baseline`.
- `harden_docker_daemon_config`, the path the Docker daemon settings are merged into. It defaults
  to `/etc/docker/daemon.json` and exists for the tests.
- `tests/docker-daemon.yml`, run by `tests/run.sh`, which applies those tasks to a scratch file and
  checks the result. It is the suite's first test that applies anything.

## 1.0.1

### Fixed

- `harden` runs its AppArmor status probe with `check_mode: false`. The probe is a `command`, which
  ansible skips under `--check`, leaving the assert after it to read a registered result that has no
  `stdout` and fail.

## 1.0.0

First release as an Ansible collection. Everything before this was a plain roles repository that
consumers put on their `roles_path`.

### Added

- `sre.server.baseline`, a role that composes `bootstrap`, `firewall` and `harden` in that order.
- `playbooks/baseline.yml`, runnable as `ansible-playbook sre.server.baseline`.
- Galaxy metadata and an Apache-2.0 licence, so the roles carry a licence and could be published.
- `tests/run.sh`, which builds and installs the collection and syntax-checks the playbooks against
  the installed copy.

### Changed

- The roles are addressed as `sre.server.bootstrap`, `sre.server.firewall` and `sre.server.harden`
  once the collection is installed. Putting `roles/` on `roles_path` and using the bare names still
  works, so a consumer that already does that keeps working untouched.
- `harden` defaults `docker_user` itself instead of borrowing the default `bootstrap` happened to
  leave in scope, so it can be run without `bootstrap`.
- `deploy_dir` defaults to `/opt/deploy`. Nothing in the collection is specific to any one project
  any more, so a consumer that wants a different directory sets it explicitly.
