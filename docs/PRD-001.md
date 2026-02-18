# Plan: NixOS Headless Deployment on Turing Pi 2

## Goal

Deploy NixOS headless on a Turing Pi 2 cluster, starting with the CM4 node (slot 3). RK1 nodes (slots 1 & 2) deferred until NixOS support matures (see thelab-rc-de5). Replicable flake-based configuration, colmena deployment pipeline, and k3s.

## Hardware

| Slot | Module | Storage | Role (target) | Status |
|------|--------|---------|---------------|--------|
| 1 | RK1 | NVMe | k3s server | Deferred — NixOS support experimental |
| 2 | RK1 | NVMe | k3s server | Deferred — NixOS support experimental |
| 3 | CM4 (4GB+, eMMC) | SATA SSD | k3s server | **Phase 1 target** |

Start with CM4 as a single-node k3s server. Add RK1 nodes later for HA (3-node etcd quorum).

## Decisions Made

- **Scope**: CM4 node first. RK1 nodes deferred (nixos-rk3588 archived, boot issues unresolved — see thelab-rc-de5).
- **Deployment tool**: Colmena (tagging, parallel deploys, sops-nix integration).
- **Secrets**: sops-nix with age keys.
- **K3s topology**: Single server (CM4 slot 3) initially. Expand to 3-node HA when RK1 nodes come online.
- **Bootstrap**: Custom pre-baked NixOS image via nixos-generators (SSH keys baked in, no UART needed after first node).
- **Build strategy**: nix-darwin `linux-builder` module for aarch64-linux cross-compilation.
- **Networking**: DHCP reservations on router. No static IPs in Nix config.
- **Security baseline**: Non-root deploy user + nftables firewall. Escalate later.
- **Storage**: Longhorn. CM4 contributes SATA SSD. RK1 nodes will add NVMe later.
- **Updates**: Renovate for automated nixpkgs update PRs.
- **Rollback**: NixOS generations only. etcd snapshots TBD.
- **Repo structure**: Split repos with `thelab-rc-` prefix:
  - `thelab-rc` — journal, learning notes, plans (this repo, renamed)
  - `thelab-infra` — NixOS configs, Colmena, secrets, hardware, image building
  - `thelab-gitops` — Kubernetes manifests, Helm charts, app deployments
- **Pace**: Ship CM4 node fast, iterate.

---

## Phase 1: Workstation Prep (do this first, before touching hardware)

### 1.1 Enable nix-darwin Linux builder

Add to your nix-darwin configuration:

```nix
nix.linux-builder = {
  enable = true;
  systems = [ "aarch64-linux" ];
};
```

Rebuild darwin config. This gives you a local aarch64-linux VM so you can build NixOS configs from your Mac without needing the nodes themselves.

### 1.2 Create repos

Rename this repo and create two new ones:

```bash
# Rename this repo (GitHub + local)
# thelab-rc -> thelab-rc

# Create infra repo
mkdir thelab-infra && cd thelab-infra && git init

# Create gitops repo
mkdir thelab-gitops && cd thelab-gitops && git init
```

**thelab-infra** directory structure:

```
thelab-infra/
  flake.nix
  flake.lock
  hosts/
    common.nix        # Shared: users, SSH, firewall, locale, base packages
    tp-node3.nix      # CM4 slot 3 -- k3s server (first node)
  hardware/
    cm4.nix           # CM4 boot/filesystem config (SATA)
  images/
    base.nix          # nixos-generators config for pre-baked image
  secrets/
    secrets.yaml      # sops-encrypted secrets (k3s token, etc.)
    .sops.yaml        # sops config (age keys per node + your personal key)
  hive.nix            # Colmena hive definition
```

**thelab-gitops** directory structure:

```
thelab-gitops/
  apps/               # Per-app manifests or Helm values
  system/             # Cluster-level resources (namespaces, RBAC, storage)
  README.md
```

RK1 host configs (`tp-node1.nix`, `tp-node2.nix`, `hardware/rk1.nix`) will be added to thelab-infra when RK1 support is ready.

### 1.3 Generate age key for secrets

```bash
nix-shell -p age --run "age-keygen -o ~/.config/sops/age/keys.txt"
```

Save the public key -- it goes in `.sops.yaml`.

---

## Phase 2: Bootstrap CM4 (slot 3)

This is the only node that requires manual UART interaction.

### 2.1 Flash stock NixOS image

1. Download the NixOS AArch64 SD image from Hydra (24.11 stable).
2. Decompress: `unzstd nixos-sd-image-*.zst`
3. Flash via BMC web UI: **Flash Node** tab > select Node 3 > upload `.img` > **Install OS**.
4. Wait ~35 minutes.

### 2.2 Boot and configure via UART

```bash
ssh root@<turing-pi-bmc-ip>          # password: turing
tpi power -n 3 on
# Wait ~60s for boot
tpi uart -n 3 set --cmd "sudo nixos-generate-config"
```

Inject SSH key and create deploy user via UART:

```bash
tpi uart -n 3 set --cmd "mkdir -p /root/.ssh && echo 'ssh-ed25519 AAAA...' > /root/.ssh/authorized_keys"
tpi uart -n 3 set --cmd "sudo nixos-rebuild switch"
```

### 2.3 SSH in and verify

```bash
ssh root@<node3-ip>
nixos-version
```

From here, all further config is over SSH. No more UART.

---

## Phase 3: Flake + Colmena Setup

### 3.1 Write flake.nix

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
    sops-nix.url = "github:Mic92/sops-nix";
    nixos-generators = {
      url = "github:nix-community/nixos-generators";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, sops-nix, nixos-generators }: {
    nixosConfigurations = {
      tp-node3 = nixpkgs.lib.nixosSystem {
        system = "aarch64-linux";
        modules = [
          sops-nix.nixosModules.sops
          ./hardware/cm4.nix
          ./hosts/common.nix
          ./hosts/tp-node3.nix
        ];
      };
      # tp-node1 and tp-node2 (RK1) will be added when NixOS RK1 support is ready
    };

    # Pre-baked image for CM4 nodes
    packages.aarch64-linux.cm4-image = nixos-generators.nixosGenerate {
      system = "aarch64-linux";
      modules = [ ./hardware/cm4.nix ./images/base.nix ];
      format = "sd-aarch64";
    };
  };
}
```

### 3.2 Write hosts/common.nix

Shared config across all CM4 nodes:

```nix
{ pkgs, ... }: {
  # --- Users ---
  users.users.deploy = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAA... your-key-here"
    ];
  };
  security.sudo.extraRules = [{
    users = [ "deploy" ];
    commands = [{ command = "ALL"; options = [ "NOPASSWD" ]; }];
  }];
  users.users.root.hashedPassword = "!";  # Disable root login

  # --- SSH ---
  services.openssh = {
    enable = true;
    settings = {
      PasswordAuthentication = false;
      PermitRootLogin = "no";
    };
  };

  # --- Firewall ---
  networking.firewall = {
    enable = true;
    allowedTCPPorts = [ 22 6443 ];  # SSH + k3s API (add 2379/2380 for etcd when HA)
    allowedTCPPortRanges = [{ from = 10250; to = 10252; }];  # kubelet
  };

  # --- Base packages ---
  environment.systemPackages = with pkgs; [
    vim
    htop
    curl
    git
  ];

  # --- Networking ---
  networking.useDHCP = true;

  # --- Nix settings ---
  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  system.stateVersion = "24.11";
}
```

### 3.3 Write hardware/cm4.nix

```nix
{ pkgs, ... }: {
  boot = {
    kernelPackages = pkgs.linuxKernel.packages.linux_rpi4;
    initrd.availableKernelModules = [ "xhci_pci" "usbhid" "usb_storage" ];
    loader = {
      grub.enable = false;
      generic-extlinux-compatible.enable = true;
    };
  };
  fileSystems."/" = {
    device = "/dev/disk/by-label/NIXOS_SD";
    fsType = "ext4";
    options = [ "noatime" ];
  };
}
```

### 3.4 Write host-specific configs

CM4 node 3 runs as a single-node k3s server initially. RK1 nodes will join later for HA.

**hosts/tp-node3.nix** (k3s server — single node for now):

```nix
{ ... }: {
  networking.hostName = "tp-node3";
  services.k3s = {
    enable = true;
    role = "server";
    clusterInit = true;
    extraFlags = "--disable traefik";  # Use your own ingress later
  };
  # sops secret for k3s token will be added here
}
```

RK1 host configs (`tp-node1.nix`, `tp-node2.nix`) will be added when NixOS RK1 support is ready. They will join as servers with `serverAddr` pointing to node 3.

### 3.5 Write hive.nix (Colmena)

```nix
{
  meta = {
    nixpkgs = import <nixpkgs> { system = "aarch64-linux"; };
  };

  defaults = { ... }: {
    deployment.targetUser = "deploy";
    deployment.tags = [ "k3s-server" ];
  };

  tp-node3 = { ... }: {
    deployment.targetHost = "<tp-node3-ip>";
    deployment.tags = [ "cm4" ];
    imports = [ ./hardware/cm4.nix ./hosts/common.nix ./hosts/tp-node3.nix ];
  };

  # RK1 nodes will be added here when ready
}
```

Deploy:

```bash
colmena apply --on tp-node3        # Single node
colmena apply                      # Everything
```

---

## Phase 4: Custom Pre-baked Image

Eliminate UART for future nodes (RK1 when ready, or additional CM4s).

### 4.1 Write images/base.nix

```nix
{ ... }: {
  # Minimal bootstrap image -- just enough to accept a colmena deploy
  users.users.deploy = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAA... your-key-here"
    ];
  };
  security.sudo.extraRules = [{
    users = [ "deploy" ];
    commands = [{ command = "ALL"; options = [ "NOPASSWD" ]; }];
  }];
  services.openssh = {
    enable = true;
    settings.PasswordAuthentication = false;
  };
  networking.useDHCP = true;
  system.stateVersion = "24.11";
}
```

### 4.2 Build the image

```bash
nix build .#packages.aarch64-linux.cm4-image
```

This uses your nix-darwin linux-builder to cross-compile. The output is an `.img` you flash via BMC. It boots with SSH ready and your key authorized -- flash, power on, run `colmena apply`.

---

## Phase 5: sops-nix Secrets

### 5.1 Configure .sops.yaml

```yaml
keys:
  - &admin age1xxxxxxxxx...     # Your personal age key
  - &node3 age1xxxxxxxxx...     # Node 3's age key (generated on node)
creation_rules:
  - path_regex: secrets/secrets\.yaml$
    key_groups:
      - age:
        - *admin
        - *node3
```

Add RK1 node keys when they come online.

### 5.2 Create secrets

```bash
sops secrets/secrets.yaml
```

Add the k3s token:

```yaml
k3s_token: "your-generated-token-here"
```

### 5.3 Reference in node configs

```nix
# In each tp-nodeN.nix
sops.secrets.k3s_token = {
  sopsFile = ../secrets/secrets.yaml;
};
services.k3s.tokenFile = config.sops.secrets.k3s_token.path;
```

---

## Phase 6: Longhorn Storage (after k3s is running)

1. Mount SATA SSD on CM4 node via NixOS `fileSystems` config.
2. Install Longhorn via Helm or kubectl manifest.
3. Configure Longhorn to use `/var/lib/longhorn` (NixOS bind-mounts the SATA drive there).
4. Single-node for now; replication increases when RK1 nodes join with NVMe.

---

## Phase 7: Renovate for Updates

1. Add a `renovate.json` to the infra repo.
2. Configure it to track `nixpkgs` input in `flake.nix`.
3. Renovate opens PRs when nixpkgs updates. You review, merge, then `colmena apply`.

---

## Future Phases (not in scope yet)

- [ ] **RK1 nodes**: Add when mainline NixOS RK1 support is viable (track thelab-rc-de5). Add hardware/rk1.nix, tp-node1.nix, tp-node2.nix. Expand to 3-node HA k3s.
- [ ] **Disk encryption**: LUKS on data partitions with network-bound unlock.
- [ ] **Monitoring**: Prometheus + Grafana as NixOS services.
- [ ] **Overlay network**: Tailscale or WireGuard for cross-network access.
- [ ] **etcd backups**: Periodic snapshots once workloads are real.
- [ ] **CIS hardening**: Audit logging, restricted kernel params, read-only root.

---

## Known Gotchas

| Issue | Mitigation |
|---|---|
| CM4 Slot 3 has reported NixOS boot issues | If slot 3 fails, try an RK1 node as initial bootstrap instead |
| Flashing can take up to 35 min | Don't interrupt. Verify via BMC UI progress indicator. |
| NixOS image filename version can be inaccurate | Verify with `nixos-version` after boot |
| nix-darwin linux-builder may need manual start | Run `sudo launchctl start org.nixos.linux-builder` if builds fail |
| Cross-compilation can be slow | First build is slow; subsequent builds use cache. Consider Cachix. |
| RK1 NixOS support is experimental | Deferred. See thelab-rc-de5 for research and options. |
| BMC mDNS may not resolve | Use IP address directly; set DHCP reservation |
| Longhorn on eMMC is space-constrained | Monitor disk usage; prefer nodes with dedicated drives for volume-heavy workloads |

---

## Execution Order (bias toward shipping)

```
Week 1:  Phase 1 (workstation prep + repos) + Phase 2 (bootstrap CM4 slot 3 via UART)
         Create repos, get CM4 booting NixOS and accessible via SSH.

Week 2:  Phase 3 (flake + colmena) + Phase 5 (sops-nix)
         Push first colmena deploy to CM4. Secrets encrypted. Single-node k3s running.

Week 3:  Phase 4 (custom image) + Phase 6 (Longhorn)
         Build pre-baked CM4 image, mount SATA storage, deploy Longhorn.

Week 4:  Phase 7 (Renovate) + gitops repo setup
         Set up automated updates. Begin populating thelab-gitops with k8s manifests.
```

---

## References

- [Run NixOS on the Turing Pi (full walkthrough)](https://woutswinkels.github.io/posts/Run-NixOS-on-the-Turing-Pi/)
- [NixOS Discourse: Turing Pi 2 + CM4](https://discourse.nixos.org/t/turing-pi-2-cm4/27402)
- [Turing Pi Docs: Flashing OS (CM4)](https://docs.turingpi.com/docs/raspberry-pi-cm4-flashing-os)
- [Turing Pi Docs: Flashing OS (RK1)](https://docs.turingpi.com/docs/turing-rk1-flashing-os)
- [NixOS Headless Profile](https://nlewo.github.io/nixos-manual-sphinx/configuration/profiles/headless.xml.html)
- [nixos-rk3588 (RK1 support)](https://github.com/ryan4yin/nixos-rk3588)
- [Colmena](https://github.com/zhaofengli/colmena)
- [sops-nix](https://github.com/Mic92/sops-nix)
- [nixos-generators](https://github.com/nix-community/nixos-generators)
- [nix-darwin linux-builder](https://nixos.org/manual/nixpkgs/stable/#sec-darwin-builder)
- [Longhorn](https://longhorn.io/)
- [Renovate](https://docs.renovatebot.com/)
