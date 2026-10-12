# nixos

Installs NixOS from the live installer, then manages the host declaratively:
admin user with an SSH key and passwordless sudo, root password, SSH, firewall,
mDNS, and podman containers through `virtualisation.oci-containers`.

The role writes `/etc/nixos/configuration.nix` and `/etc/nixos/containers.nix`
(it owns them; `hardware-configuration.nix` comes from `nixos-generate-config`),
root-only files in `/etc/nixos/secrets/`, and runs `nixos-rebuild switch` when
one of them changes.

## How it runs

1. Logs in as `nixos_admin_user` with the SSH key, or as root with
   `nixos_root_password` when the key doesn't work yet. NixOS has no Python by
   default, so it uses the system `python3` or fetches one with `nix-build`.
2. If `/` is a tmpfs (live installer): erases `nixos_install_disk`, creates a
   UEFI boot partition and an ext4 root, writes the configuration, runs
   `nixos-install`, reboots and logs back in as the admin user. This needs
   `nixos_install_wipe: true` and doesn't run in check mode.
3. Writes the configuration and applies it.

## Inputs

| Variable | Meaning |
| --- | --- |
| `nixos_ssh_public_key` | Admin user's authorized key (secret input) |
| `nixos_ssh_private_key` | Matching private key, written to `nixos_ssh_private_key_file` on the controller (secret input) |
| `nixos_root_password` | Root password (secret input) |
| `nixos_secret_files` | `name: content` files written to `/etc/nixos/secrets/<name>`, mode 0600, e.g. container `environmentFiles` (secret input) |
| `nixos_install_disk` | Disk erased by the install, preferably a `/dev/disk/by-id/` path |
| `nixos_install_wipe` | Must be `true` for the install to run |
| `nixos_containers` | `virtualisation.oci-containers.containers`, as YAML |
| `nixos_container_dirs` | Host directories created for the containers |
| `nixos_firewall_tcp_ports`, `nixos_firewall_udp_ports` | Open ports |

See `defaults/main.yml` for the others.

`nixos_containers` is converted to Nix as is: strings, numbers, booleans,
lists and mappings, so any `oci-containers` option works. Never put secrets in
it, because `/etc/nixos` and the Nix store are world-readable. Use
`environmentFiles` that point to `nixos_secret_files`.
