# hcloud-tailnet

Bootstrap and secure a Hetzner or DigitalOcean VPS with Tailscale in minutes.

Connect your computer to Tailscale, then install the dependencies:

```sh
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yaml -p .ansible/collections
```

Copy the example files:

```sh
cp config.example.yaml config.yaml
cp secrets.example.yaml secrets.yaml
chmod 600 secrets.yaml
```

Edit `config.yaml`: set `provider` to `hcloud` or `digitalocean`, then fill in the rest for your server. `server_type`, `server_location`, and `server_image` are provider-specific values, see below.

If you need an SSH key, run `ssh-keygen -t ed25519 -f "$HOME/.ssh/id_ed25519"`. Load a passphrase-protected key with `ssh-add` before running. If the public key is already registered with your provider, set `ssh_key_name` to its name.

Fill in `secrets.yaml` with a [non-ephemeral, one-off Tailscale auth key](https://tailscale.com/docs/features/access-control/auth-keys) and the API token for your chosen provider (see below). Your tailnet must allow SSH to the server.

Then run:

```sh
ansible-playbook -i localhost, setup.yaml
```

To keep more than one server's config around, name the files however you like and pass them in:

```sh
ansible-playbook -i localhost, setup.yaml -e config_file=my-new-config.yaml -e secrets_file=my-new-secrets.yaml
```

The playbook detects your public IPv4 for temporary SSH access. Set `bootstrap_ssh_cidr` in `config.yaml` if it detects the wrong address. If setup stops before closing SSH, fix the problem and rerun. For later runs, use Tailscale with MagicDNS, or pass `-e tailnet_host=100.x.y.z`.

## Hetzner (`provider: hcloud`)

- `server_type`/`server_location`/`server_image`: a [server type](https://www.hetzner.com/cloud), [location](https://docs.hetzner.com/cloud/general/locations/), and image slug (e.g. `cx23`, `nbg1`, `ubuntu-24.04`).
- `hcloud_token` in `secrets.yaml`: a [Hetzner Cloud Read & Write token](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/).

## DigitalOcean (`provider: digitalocean`)

- `server_type`/`server_location`/`server_image`: a [droplet size](https://slugs.do-api.dev/), [region](https://docs.digitalocean.com/platform/regional-availability/), and image slug (e.g. `s-1vcpu-1gb`, `ams3`, `ubuntu-24-04-x64`).
- `digitalocean_token` in `secrets.yaml`: a [DigitalOcean API token](https://docs.digitalocean.com/reference/api/create-personal-access-token/) with read/write scope. Unlike a Hetzner token, this is account-wide, not scoped to one project.
- `do_project_name`: DigitalOcean has no "create in project" option, a Droplet always lands in the Default project first. Set this to move it into a named project after creation, or leave blank to leave it in Default.

## FAQ

**Is this a full IaC tool?** No. It bootstraps one VPS. Reruns reapply the server baseline, but it doesn't maintain an inventory or manage applications.

**Why use a cloud firewall instead of UFW?** [Docker can bypass UFW's usual rules](https://docs.docker.com/engine/network/packet-filtering-firewalls/) when a container port is published. Filtering public traffic before it reaches the VPS, via the [Hetzner firewall](https://docs.hetzner.com/cloud/firewalls/faq/) or [DigitalOcean Cloud Firewall](https://docs.digitalocean.com/products/networking/firewalls/), blocks unsolicited public connections regardless of what's running on the VPS. Access over Tailscale is governed by your tailnet policy.

**Can I switch a server between providers?** No, `provider` picks which API is used to find and create the server; it doesn't migrate anything. Each provider's server, firewall, and SSH key are independent resources named after `server_name`.
