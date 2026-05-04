# fix-cloud-init-dns

Fixes a bug in cloud-init's ENI renderer on Proxmox where `dns-nameservers` and `dns-search` directives are placed under the loopback interface (`lo`) instead of the physical interface, causing VMs to boot without functional DNS resolution.

## The problem

When deploying VMs from a Proxmox template with cloud-init enabled, Proxmox generates a `network-config` file in version 1 format and passes it to cloud-init via a seed ISO mounted at `/dev/sr0`. Cloud-init processes this file using the ENI renderer and writes the result to `/etc/network/interfaces.d/50-cloud-init`.

Due to a bug in cloud-init's ENI renderer, the `nameserver` block is associated with the loopback interface instead of the physical interface:

```
auto lo
iface lo inet loopback
    dns-nameservers 192.168.100.5    # <-- wrong: should not be here
    dns-search lan                   # <-- wrong: should not be here

auto eth0
iface eth0 inet static
    address 192.168.100.9/24
    gateway 192.168.100.1
    # <-- correct: dns directives should be here
```

Since `ifupdown` ignores `dns-nameservers` and `dns-search` when they appear under the loopback interface, `resolvconf` is never called, `/etc/resolv.conf` remains empty, and the VM boots without DNS resolution.

## The solution

A shell script corrects the generated interfaces file by moving the DNS directives from `lo` to the physical interface. A systemd service invokes the script at the right moment — after `cloud-init-local.service` writes the network file and before `networking.service` brings up the interfaces.

The script can also be invoked directly at package install time to fix already-running VMs without requiring a reboot.

## Affected environment

- **Hypervisor:** Proxmox VE
- **Guest OS:** Debian 13 (Trixie)
- **Network management:** `ifupdown` with ENI renderer
- **cloud-init datasource:** NoCloud (seed via `/dev/sr0`)

## Requirements

- `resolvconf` — required for `ifupdown` to process `dns-nameservers` directives
- `ifupdown`
- `cloud-init`

## Installation

### Option A — Install the .deb package (recommended)

Download the latest release from the [releases page](https://github.com/ezequielandrush/fix-cloud-init-dns/releases) and install it:

```bash
sudo dpkg -i fix-cloud-init-dns.deb
```

`apt` will handle the `resolvconf` dependency automatically if it is not already installed:

```bash
sudo apt install ./fix-cloud-init-dns.deb
```

### Option B — Manual installation

**1. Install resolvconf:**

```bash
sudo apt install resolvconf
```

**2. Copy the script:**

```bash
sudo cp etc/network/if-pre-up.d/fix-cloud-init-dns /etc/network/if-pre-up.d/fix-cloud-init-dns
sudo chmod +x /etc/network/if-pre-up.d/fix-cloud-init-dns
```

**3. Copy the systemd service:**

```bash
sudo cp etc/systemd/system/fix-cloud-init-dns.service /etc/systemd/system/fix-cloud-init-dns.service
```

**4. Enable the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable fix-cloud-init-dns.service
```

## Including in a Proxmox template

Perform the installation steps above on the base VM before converting it to a template. Before shutting down, clean the cloud-init state so cloned VMs boot as fresh instances:

```bash
# Clear bash history
sudo truncate -s 0 /root/.bash_history
sudo truncate -s 0 /home/<user>/.bash_history
history -c

# Clear logs
sudo find /var/log -type f -exec truncate -s 0 {} \;
sudo journalctl --rotate
sudo journalctl --vacuum-size=1M

# Clean cloud-init state
sudo cloud-init clean --logs --seed

sudo shutdown -h now
```

Then convert the VM to a template from the Proxmox GUI.

## Verification

After a reboot, verify that:

**The interfaces file has DNS in the correct place:**

```bash
cat /etc/network/interfaces.d/50-cloud-init
```

Expected output:

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    dns-nameservers 192.168.100.5
    dns-search lan
    address 192.168.100.9/24
    gateway 192.168.100.1
```

**The service ran successfully:**

```bash
sudo systemctl status fix-cloud-init-dns.service
# Should show: Active: active (exited) ... status=0/SUCCESS
```

**DNS resolution works:**

```bash
cat /etc/resolv.conf
ping -c 2 8.8.8.8
```

## Known limitations

- **Multiple physical interfaces:** the script applies the DNS directives only to the first physical interface found in the network file. This is sufficient because `resolvconf` updates `/etc/resolv.conf` globally when any interface comes up with DNS configured.
- **Proxmox network-config format:** this fix assumes Proxmox continues generating `network-config` in version 1 format. If Proxmox switches to version 2 in the future, the original bug may disappear and this workaround would become unnecessary — though harmless, as the script would simply find no DNS under `lo` and exit cleanly.

## Contributing

Bug reports and pull requests are welcome at https://github.com/ezequielandrush/fix-cloud-init-dns.

## License

MIT
