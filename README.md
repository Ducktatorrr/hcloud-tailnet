# hcloud-tailnet

Bootstraps one Hetzner Cloud VPS with Tailscale, then closes public SSH after checking private access. Reruns reapply this baseline; the tool has no fleet inventory or application management.

Connect your computer to Tailscale, then install the dependencies:

```sh
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yaml -p .ansible/collections
```

Copy the example files and edit `config.yaml` for your server, an available type/location, and matching SSH key paths:

```sh
cp config.example.yaml config.yaml
cp secrets.example.yaml secrets.yaml
chmod 600 secrets.yaml
```

If you need an SSH key, run `ssh-keygen -t ed25519 -f "$HOME/.ssh/id_ed25519"`. Load a passphrase-protected key with `ssh-add` before running. If the public key is already in your Hetzner project, set `hetzner_ssh_key_name` to its name.

Fill in `secrets.yaml` with a [Hetzner Cloud Read & Write token](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/) and a [non-ephemeral, one-off Tailscale auth key](https://tailscale.com/docs/features/access-control/auth-keys). Your tailnet must allow SSH to the server.

Then run:

```sh
ansible-playbook -i localhost, setup.yaml
```

The playbook detects your public IPv4 for temporary SSH access. Set `bootstrap_ssh_cidr` in `config.yaml` if it detects the wrong address. If setup stops before closing SSH, fix the problem and rerun. For later runs, use Tailscale with MagicDNS, or pass `-e tailnet_host=100.x.y.z`.

## FAQ

**Why use a Hetzner firewall instead of UFW?** [Docker can bypass UFW's usual rules](https://docs.docker.com/engine/network/packet-filtering-firewalls/) when a container port is published. The [Hetzner firewall](https://docs.hetzner.com/cloud/firewalls/faq/) filters public traffic before it reaches the VPS; with no inbound rules, unsolicited public connections are blocked. Access over Tailscale is governed by your tailnet policy.
