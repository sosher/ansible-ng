# Ansible Learning Lab

This will be the staging ground for bot learning and applying Ansible. It will evolve as I gain experience and broaden new ways to use the tool. From time to time I'll challenge myself with mini-quizzes or reviews to confirm the concepts stick.

## Living document

As I add new playbooks, roles, and experiments, I'll capture the lessons learned, patterns that work well, and any pitfalls worth revisiting. Nothing here is final—it's a sandbox meant to grow with my skill set.

## Immediate ideas

- Track what I learn from each automation experiment.
- Capture reusable snippets so future playbooks start from a stronger baseline.
- Note any supporting infrastructure (inventory layout, testing approach, etc.) that makes trying ideas faster.

## Current scaffolding

- `inventories/hosts.ini` – defines an **inventory**, which is the list of machines Ansible can target. The only entry is `localhost` because I run Ansible on my trusted workstation and let the `community.general.lxd_container` module talk straight to the IncusOS API using my existing client certificate.
- `playbooks/create_trixie.yml` – the first **playbook**, i.e., a YAML file that declares a desired state. It ensures a Debian Trixie container exists and installs `duf`, `bat`, `git`, and `fzf` inside it.
- `playbooks/bootstrap_arch.yml` – bootstraps an Arch host by installing base packages, enabling the Chaotic-AUR repo, and installing `yay` (prebuilt) so later playbooks can install AUR packages.

## Playbook summary

These are intentionally small, single-purpose playbooks.

- `playbooks/create_trixie.yml` (hosts: `incus`)
   - Ensures an Incus container named `trixie-lab` is running from the `debian/trixie` image.
   - Installs a small toolset inside the container (defaults: `duf`, `bat`, `git`, `fzf`) by running `incus exec ... apt-get`.
   - Run: `ansible-playbook -i inventories/hosts.ini playbooks/create_trixie.yml`

- `playbooks/bootstrap_arch.yml` (hosts: `arch`)
   - Installs base packages needed for AUR builds (`arch_base_packages` from group vars).
   - Enables the Chaotic-AUR repository and installs `yay` from the repo (prebuilt).
   - Installs any `arch_chaotic_packages` from group vars (e.g., `pacseek`) via `pacman`.
   - If the repo install fails, falls back to cloning/building `yay` from AUR.
   - Run (Vault is usually required): `ansible-playbook -i inventories/hosts.ini playbooks/bootstrap_arch.yml`

- `playbooks/chaotic_aur.yml` (hosts: `arch`)
   - Adds the Chaotic-AUR signing key + installs keyring/mirrorlist packages.
   - Adds the `[chaotic-aur]` block to `/etc/pacman.conf` and refreshes the package DB.
   - Note: this is now included in `playbooks/bootstrap_arch.yml` (kept for standalone use).
   - Run: `ansible-playbook -i inventories/hosts.ini playbooks/chaotic_aur.yml`

- `playbooks/install_hyprland_min.yml` (hosts: `arch`)
   - Installs a minimal Hyprland package set (`arch_hyprland_packages` from group vars).
   - Creates `~/.config/hypr/` and writes a minimal `hyprland.conf` only if it does not already exist.
   - Run: `ansible-playbook -i inventories/hosts.ini playbooks/install_hyprland_min.yml`

## How to run the Trixie playbook

1. Make sure the `community.general` collection is installed (`ansible-galaxy collection install community.general`), since the `community.general.lxd_container` module comes from there.
2. Make sure your workstation already trusts `IncusOS` (e.g., `incus remote add IncusOS https://IncusOS:8443 --accept-certificate` and `incus remote switch IncusOS`). The module uses the default client cert/key under `~/.config/lxc/`.
3. Adjust the variables near the top of `playbooks/create_trixie.yml` if your Incus remote name or API URL differ (`trixie_incus_remote`, `trixie_incus_url`, etc.).
4. From the repo root run:
   ```bash
   ansible-playbook -i inventories/hosts.ini playbooks/create_trixie.yml
   ```
   The first task uses the `community.general.lxd_container` module (see the Context7 Ansible docs) to pull the `debian/trixie` alias and launch/start the container on Incus. The next two tasks call `incus exec` to inspect which packages are missing and install only what’s required.
Once I'm comfortable running this end-to-end manually, the next iteration will be to split container creation and package management into roles and start covering them with Molecule tests.

## How to bootstrap an Arch laptop

1. Edit `inventories/hosts.ini` and replace `ansible_host=CHANGE_ME` for `mucus`.
2. Ensure you can SSH to the laptop as `sosher` and that `sudo` works.
3. Install the required collection (once):
   ```bash
   ansible-galaxy collection install community.general
   ```
4. Create an Ansible Vault file to hold the sudo password (recommended instead of `-K`). For the whole `arch` group, put it at:
   ```bash
   ansible-vault create inventories/group_vars/arch/vault.yml
   ```
   Put this inside (either name works; `ansible_become_pass` is the common one):
   ```yaml
   ansible_become_pass: "<your sudo password on mucus>"
   ```
   (If the password differs per machine, use `inventories/host_vars/mucus/vault.yml` instead.)
5. Run (you will be prompted for the Vault password):
   ```bash
   ansible-playbook -i inventories/hosts.ini playbooks/bootstrap_arch.yml --ask-vault-pass
   ```

Option: to avoid typing the Vault password every run, create a local file `./.vault_pass` containing only the Vault password, then run:
```bash
ansible-playbook -i inventories/hosts.ini playbooks/bootstrap_arch.yml --vault-password-file ./.vault_pass
```

If you have multiple vault passwords, use vault identities (examples):
- Prompted: `--vault-id arch@prompt`
- From file: `--vault-id arch@./.vault_pass`

If you see an error about being "Unable to use multiprocessing" and `/dev/shm`, verify `/dev/shm` exists and is writable (it is normally a tmpfs mounted with mode `1777`).

## How to add the Chaotic-AUR repository (Arch)

This playbook only enables the repo (signing key, keyring/mirrorlist packages, and `/etc/pacman.conf` entry).

If you’re bootstrapping a new Arch host, `playbooks/bootstrap_arch.yml` now enables Chaotic-AUR as part of the bootstrap.

```bash
ansible-playbook -i inventories/hosts.ini playbooks/chaotic_aur.yml --ask-vault-pass
```

### Why the Arch playbook looks longer than the four manual commands

`makepkg` now emits two artifacts (`yay-<version>.pkg.tar.zst` and `yay-debug-<version>.pkg.tar.zst`). The debug tarball does **not** contain `/usr/bin/yay`, so the playbook explicitly filters it out before attempting to install anything. In addition, `pacman -Qip` sprinkles ANSI color codes into its output when the command is run non-interactively, which breaks naive regex parsing of the package metadata. To keep automation reliable, the playbook now derives the package name straight from the archive filename instead of scraping `pacman -Qip`. Those two guardrails are why there’s a bit more YAML than the four shell commands listed on the Arch Wiki, but they make the run idempotent and self-healing when Ansible executes unattended.
