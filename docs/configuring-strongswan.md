<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020 Scott Crossen
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up strongSwan

This is an [Ansible](https://www.ansible.com/) role which installs [strongSwan](https://www.strongswan.org/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

strongSwan is an IPsec-based VPN solution. The [container image](https://hub.docker.com/r/strongx509/strongswan) used by this role supports IKEv2 only, with a limited set of plugins.

See the project's [documentation](https://docs.strongswan.org/) to learn what strongSwan does and why it might be useful to you.

## Adjusting the playbook configuration

To enable strongSwan with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# strongswan                                                           #
#                                                                      #
########################################################################

strongswan_enabled: true

########################################################################
#                                                                      #
# /strongswan                                                          #
#                                                                      #
########################################################################
```

### Select the network mode

By default the container runs in its own network, and only traffic from the container itself can pass through its tunnels. This is enough for testing a connection, but site-to-site and road warrior setups need the host (and the networks behind it) to use the tunnels. For those, run the container with host networking:

```yaml
strongswan_container_network: host
```

strongSwan then manages the IPsec policies of the host. The role does not touch the firewall, so open `500/udp` and `4500/udp` yourself, plus ESP (IP protocol 50) if neither peer is behind NAT. To route traffic for the networks behind the host, enable IP forwarding on it as well.

With the default network, the role publishes `500/udp` and `4500/udp` on the host. Peers see Docker's port publishing as NAT, so ESP is encapsulated in UDP on port `4500` and no other ports are needed.

### Configure connections

Connections are configured with `strongswan_swanctl_conf`, which the role installs as [`swanctl.conf`](https://docs.strongswan.org/docs/latest/swanctl/swanctlConf.html). It holds the connections, secrets, pools and authorities.

The example below connects `10.0.1.0/24` behind this host to `10.0.2.0/24` behind the peer, with both sides authenticating with the same pre-shared key. It needs host networking.

```yaml
strongswan_swanctl_conf: |
  connections {
    site-to-site {
      version = 2
      remote_addrs = REMOTE_ADDRESS_HERE
      local {
        auth = psk
        id = LOCAL_ID_HERE
      }
      remote {
        auth = psk
        id = REMOTE_ID_HERE
      }
      children {
        site-to-site {
          local_ts = 10.0.1.0/24
          remote_ts = 10.0.2.0/24
          start_action = start
        }
      }
    }
  }

  secrets {
    ike-site-to-site {
      id = REMOTE_ID_HERE
      secret = "PRE_SHARED_KEY_HERE"
    }
  }
```

To authenticate with certificates instead, mount them into the container under `/etc/swanctl`. Keep them outside of `/strongswan`, which is removed when the service is uninstalled:

```yaml
strongswan_container_additional_volumes_custom:
  - type: bind
    src: /srv/strongswan/x509
    dst: /etc/swanctl/x509
    options: readonly
  - type: bind
    src: /srv/strongswan/private
    dst: /etc/swanctl/private
    options: readonly
```

Configuration snippets mounted at `/etc/swanctl/conf.d` are loaded as well, so `strongswan_swanctl_conf` can be left empty if all of the configuration lives there.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. `strongswan_conf` replaces the whole [`strongswan.conf`](https://docs.strongswan.org/docs/latest/config/strongswanConf.html), so keep its default `start-scripts` and `filelog` sections when overriding it. Without them, the connections are not loaded and nothing is logged.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, strongSwan loads the configured connections and initiates those with `start_action = start`. To see their state, log in to the server with SSH and run `docker exec strongswan swanctl --list-sas` (or how you/your playbook named the container, e.g. `mash-strongswan`).

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu strongswan` (or how you/your playbook named the service, e.g. `mash-strongswan`).
