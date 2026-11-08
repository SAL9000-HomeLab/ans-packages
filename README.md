# ans-packages

Installs and removes software packages on Linux and Windows hosts with one role, `ans_packages`. Each package has its
own variable, set on the inventory, group or host, and its own task file that handles the OS differences. It is meant
to be included from [ans-defos] as part of the default OS configuration, and [site.yml](site.yml) runs it on its own.

Only the packages whose variable is set are touched.

| Package | Variable | Tag | Windows | RHEL/Rocky 9, 10 | Ubuntu 24.04 |
| --- | --- | --- | --- | --- | --- |
| PowerShell 7 | `ans_packages_powershell` | `powershell` | MSI | RPM | .deb |

## Requirements

| Requirement | Version |
| --- | --- |
| ansible-core | >= 2.16 (tested on 2.19 and 2.21) |
| Python | >= 3.10 |

- Linux targets: RHEL/Rocky 9 and 10, Ubuntu 24.04, reached over SSH with `become`.
- Windows targets: Windows Server 2025 and Windows 11, reached over PSRP as an administrator. The controller needs the
  Python packages in [requirements.txt](requirements.txt).
- x86_64 (x64) only, for now.
- The hosts download from GitHub (or the mirror in `ans_packages_powershell_base_url`), and RHEL/Rocky hosts also
  from packages.microsoft.com for the signing key.
- Collections: `ansible.windows`, from [requirements.yml](requirements.yml).

The role gathers the `min` fact subset itself when facts were not gathered, so it works under plays with
`gather_facts: false`.

## Variables

Defaults are in [roles/ans_packages/defaults/main.yml](roles/ans_packages/defaults/main.yml).

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_packages_powershell` | `""` | `present`, `absent`, or empty to leave PowerShell 7 alone. |
| `ans_packages_powershell_version` | `7.6.9` | Release to install (7.6 is the current LTS). |
| `ans_packages_powershell_base_url` | GitHub releases | Where the installers and `hashes.sha256` come from; `v<version>` is appended. |
| `ans_packages_powershell_rpm_key_url` | Microsoft's key | RHEL/Rocky: the key the RPM is signed with. |
| `ans_packages_powershell_rpm_key_fingerprint` | `BC52...29CF` | RHEL/Rocky: the fingerprint that key must have. |
| `ans_packages_powershell_msi_properties` | see defaults | Windows: MSI properties. Microsoft Update is off. |
| `ans_packages_download_dir_linux` | `/var/tmp` | Where installers are downloaded before they are installed and deleted. |
| `ans_packages_download_dir_windows` | `C:\Windows\Temp` | The same, on Windows. |

Because each package has its own variable, a host can override one package without repeating the rest. Example, in
group vars or the AWX inventory:

```yaml
# all hosts
ans_packages_powershell: present

# one host
ans_packages_powershell: absent
```

## Behavior

- **PowerShell, present:** installs `ans_packages_powershell_version` when PowerShell 7 is missing or older than it,
  and leaves a newer version alone, so raising the version upgrades hosts and nothing is downgraded. The role
  downloads the release's `hashes.sha256`, downloads the installer, checks its SHA256, installs it and deletes it.
  - Windows: the x64 MSI, with PowerShell on the `PATH`, the event log manifest registered, and Microsoft Update,
    PowerShell remoting and the Explorer menu entries off. Installing over an older 7.x upgrades it in place.
    Windows PowerShell 5.1 and preview builds are not touched.
  - RHEL/Rocky: the release's RHEL RPM, through `dnf`, which installs its dependencies and checks the RPM's
    signature. Before that the role installs `gnupg2` (which a minimal Rocky 10 may lack) and imports Microsoft's
    signing key from `ans_packages_powershell_rpm_key_url`, which must have the fingerprint in
    `ans_packages_powershell_rpm_key_fingerprint`.
  - Ubuntu: the release's .deb, through `apt`, which installs its dependencies.
- **PowerShell, absent:** removes the MSI on Windows, or the `powershell` package on Linux.
- **Package repositories:** the role does not add the Microsoft package repository, so the version only changes
  when `ans_packages_powershell_version` does. A host that already has PowerShell from that repository keeps it, and
  upgrades from it continue.

## Adding a package

1. Add `ans_packages_<name>: ""` (and any settings it needs) to `defaults/main.yml` with comments.
2. Add `tasks/<name>.yml`, which handles each OS, splitting into `tasks/<name>_<os>.yml` files when it helps.
3. In `tasks/main.yml`, add the variable to the check list and an `include_tasks` with its own tag.
4. Add a row to the tables here and in [ans-defos], and a changelog line.

## Inventory

[site.yml](site.yml) runs the role on the `linux` group (with `become`) and the `windows` group (over PSRP), as in
the placeholder [inventory/hosts.yml](inventory/hosts.yml). Put real hosts in the AWX inventory, or in a gitignored
`local/` copy.

## Development and CI

The workflows and lint configs are the shared ones from `ans-template`:

- **Ansible CI** (`.github/workflows/ansible-ci.yml`), on every pull request: `yamllint`,
  `ansible-playbook --syntax-check site.yml`, `ansible-lint`.
- **Linting Validation** (`.github/workflows/ci.yml`), on pull requests: markdownlint, linkspector, yamllint.

Run the same checks locally:

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install "ansible-core>=2.16" "ansible-lint>=6" "yamllint>=1.30" -r requirements.txt
ansible-galaxy collection install -r requirements.yml
yamllint -f parsable .
ansible-playbook -i inventory/hosts.yml --syntax-check site.yml
ansible-lint .
npx markdownlint-cli2 "**/*.md" "#.venv"
```

Add a line under `## [Unreleased]` in [CHANGELOG.md](CHANGELOG.md) with each change. Pushing a `vX.Y.Z` tag
publishes that version's section as a GitHub release.

## License

MIT, see [LICENSE](LICENSE).

[ans-defos]: https://github.com/SAL9000-HomeLab/ans-defos
