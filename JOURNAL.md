# Homelab Journey

## Day 1 - July 8th, 2025
- I am starting down the rabit hole that is NixOS. Why? As a recovering perfectionist it really itches that need to have a reproducible environment that I can always return back to where I feel safe.  Just tear it all down and start again.

## Day 2 - July 9th, 2025
- GPT stands for `GUID Partition Table`.  It tells the OS where it can find a particular partion on the disk. It lives along side the partitions it references.
- When installing NixOS from the minimal iso, it is required that you manually partition your disk.

## Day 3 - February 14th, 2026
- Added `nix.linux-builder` to my nix-darwin config. This runs a NixOS virtual machine (QEMU) on macOS so I can build `aarch64-linux` packages without a separate Linux machine. The config itself was simple — just 5 lines in `modules/darwin/builders.nix`.
- The interesting part was debugging the first boot. Three things tripped us up:

### Nix flakes can't see untracked files
- `darwin-rebuild switch` failed with "path does not exist" for `builders.nix`. Nix flake evaluation only sees files that git knows about. In a dirty tree, new files must be `git add`ed before Nix can find them. This is a flake sandboxing behavior — it evaluates against the git index, not the working directory.

### The VM takes time to boot
- After a successful rebuild, `ssh-keyscan` kept returning "broken pipe" on port 31022. Turns out the QEMU process was running but NixOS inside the VM was still booting. On first run it has to create the store image (~1GB) and boot a full Linux kernel. Takes 1-2 minutes.
- Useful debugging flow: `ps aux | grep qemu` (is QEMU alive?) → `sudo lsof -nP -iTCP:31022` (is the port open?) → `nc -w 5 localhost 31022 < /dev/null` (is SSH responding?). Each command narrows the problem.

### SSH key permissions are root-only by design
- `nix store ping --store ssh-ng://linux-builder` failed with "host key verification failed". The real error was `Load key "/etc/nix/builder_ed25519": Permission denied` — the SSH private key is `600 root:nixbld`.
- This is intentional. The nix daemon runs as root and handles all builder connections. Your user never SSHs to the builder directly. The right smoke test is just: `nix build --system aarch64-linux nixpkgs#hello`.

### Commands I learned
- `sudo launchctl list | grep linux-builder` — check if a macOS service is running and its PID
- `sudo launchctl print system/org.nixos.linux-builder` — detailed service info (args, env, working dir)
- `sudo lsof -nP -iTCP:31022` — show what process is listening on a specific TCP port. `-n` skips DNS, `-P` skips port name lookup
- `nc -w 5 localhost 31022 < /dev/null` — raw TCP connection test with 5s timeout. Shows the SSH banner if the service is up, without needing SSH keys
- `file result/bin/hello` — identify binary type. Confirmed the output was `ELF 64-bit ... ARM aarch64 ... GNU/Linux`
