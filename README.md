# tailscale-initramfs

Run the [tailscale](https://tailscale.com) client in a Debian or Ubuntu
initramfs, to provide access to the Linux system prior to unlocking an encrypted
root filesystem.


## Installation

0. Requires tailscale already be [installed](https://pkgs.tailscale.com/stable/)

1. Install tailscale-initramfs package

```bash
# Add the repository
sudo mkdir -p --mode=0755 /usr/local/share/keyrings
curl -fsSL https://lugoues.github.io/tailscale-initramfs/public.key | sudo tee /usr/local/share/keyrings/tailscale-initramfs-keyring.asc >/dev/null
echo 'deb [signed-by=/usr/local/share/keyrings/tailscale-initramfs-keyring.asc] https://lugoues.github.io/tailscale-initramfs/repo stable main' | sudo tee /etc/apt/sources.list.d/tailscale-initramfs.list >/dev/null

# Install tailscale-initramfs
sudo apt-get update && sudo apt-get install tailscale-initramfs
```

## Configure

Run `setup-initramfs-tailscale` and follow the instructions. It will register a tailscale node with a hostname derived from the host system,
let say the host is named `homeserver`, the tailscale node will be registered as `homeserver-initrd`; that makes it easier to later identify the node in Tailscale panel.

### Tailscale SSH server (highly recommended)

The Tailscale daemon can run a builtin SSH server, if enabled, installing _dropbear_ or _tinyssh_ isn't required to access the node remotely.

To enable it pass `--ssh` option like in: `setup-initramfs-tailscale --ssh`

The main difference of the builtin SSH server to something like _dropbear_ or _tinyssh_ is that the former is only accessible over the tailnet, the node won't respond to local connections unless the client is also connected to the tailscale network.

The setup script generates the SSH host keys in `/etc/tailscale/initramfs/ssh` and they are copied into the initramfs, so the host identity stays the same across reboots. If you ran the setup before this was added, generate them manually and rebuild the initramfs:

```bash
sudo install -m700 -d /etc/tailscale/initramfs/ssh
for typ in rsa ecdsa ed25519; do
  sudo ssh-keygen -q -t "$typ" -N "" -f "/etc/tailscale/initramfs/ssh/ssh_host_${typ}_key"
done
sudo update-initramfs -u -k all
```

## Security Considerations

The *Tailscale node key* and the *SSH host keys* will be stored in plain text inside the initramfs. Even if the root filesystem is encrypted, remember that the initramfs isn't. Someone with physical access to the node could steal the tailscale keys and attempt to log into the tailscale network impersonating the node the keys were created for.

To minimize the attack surface, we can limit the initramfs tailscale node to only accept incoming connections by addding the following [Tailscale ACL](https://login.tailscale.com/admin/acls) and tag clients, servers and initrd nodes accordinglly using the [Tailscale Machines](https://login.tailscale.com/admin/machines) panel.


```json
{
	"tagOwners": {
		"tag:initrd": ["autogroup:admin"],
		"tag:client": ["autogroup:admin"],
		"tag:server": ["autogroup:admin"],
    },

	"acls": [
		{"action": "accept", "src": ["tag:client"], "dst": ["*:*"]},
		{"action": "accept", "src": ["tag:server"], "dst": ["tag:server:*"]},
	],
}
```

Even if the attacker manages to get the node keys, it won't be able to escalate into your tailscale network and all other nodes will be unreacheable.


## NetBird (netbird-initramfs)

The `netbird-initramfs` package does the same with [NetBird](https://netbird.io) instead of Tailscale. It can be installed alongside or instead of `tailscale-initramfs`.

0. Requires netbird already be [installed](https://docs.netbird.io/how-to/installation/linux) (the `netbird` package from pkgs.netbird.io)

1. Install the package: `sudo apt-get install netbird-initramfs` (same repository as above)

2. Register the initramfs peer:

```bash
sudo setup-initramfs-netbird --ssh --setup-key-file node.key
# self-hosted:
sudo setup-initramfs-netbird --ssh --management-url https://netbird.example.net --setup-key-file node.key
```

The peer is named after the host with an `-initrd` suffix (override with `--hostname`). All other arguments are passed to `netbird login`. Without a setup key an SSO URL/QR code is printed instead, but SSO-registered peers expire, so either use a setup key or disable *login expiration* for the peer in the dashboard.

The script registers the peer with a temporary netbird daemon running in a private mount namespace, so the system netbird service and its `/var/lib/netbird` are left untouched. The resulting profile is stored in `/etc/netbird/initramfs/default.json`.

3. Review `/etc/netbird/initramfs/config` and run `sudo update-initramfs -u -k all`.

At boot the client is started with `netbird up --foreground-mode`; extra flags go in `NETBIRD_OPTIONS`. `--ssh` sets `NETBIRD_OPTIONS="--allow-server-ssh --enable-ssh-root"`, enabling NetBird's builtin SSH server (`netbird ssh root@<peer>`); it also needs an access control policy that allows SSH to the peer.

NetBird needs DNS to reach the management service. The DNS servers from DHCP / `ip=` are used, with `FALLBACK_DNS_SERVERS` in the config as a fallback.

Security: the WireGuard private key and the NetBird SSH host key are stored in plain text in the initramfs, as with Tailscale above. Put the initrd peers in their own group and only add policies that allow traffic *from* your admin clients *to* that group, never from it. Note that the netbird binary adds roughly 40 MB (uncompressed) to the initramfs.

## Prior work and big thanks

+ [@darkrain42][gh0] for creating [tailscale-initramfs](https://github.com/darkrain42/tailscale-initramfs), the basis for this entire thing
* [@dangra][gh1] for the foundation of the `setup-initrdfs-init` script he created for [mkinitcpio-tailscale](https://github.com/dangra/mkinitcpio-tailscale)

[gh0]: https://github.com/darkrain42
[gh1]: https://github.com/dangra
