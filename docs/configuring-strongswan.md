<!--
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up strongSwan

This is an [Ansible](https://www.ansible.com/) role which installs [strongSwan](https://www.strongswan.org/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

strongSwan is an IPsec-based VPN solution, implementing the IKEv1 and IKEv2 key exchange protocols.

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

### Configure connections

To enable the service it is also required to configure at least one connection. The role installs the contents of `strongswan_swanctl_conf` as [`swanctl.conf`](https://docs.strongswan.org/docs/latest/swanctl/swanctlConf.html), which holds the connections, secrets, pools and authorities.

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

Certificates and keys can be mounted into the container under `/etc/swanctl` (e.g. `/etc/swanctl/x509`, `/etc/swanctl/private`) with the `strongswan_container_additional_volumes_custom` variable.

### Configure networking

By default the container runs in its own network and publishes the IKE (`500/udp`) and NAT traversal (`4500/udp`) ports on the host. IPsec tunnels are terminated inside the container, so only the container itself sees the traffic flowing through them. Peers see the published ports as a NAT, so IKE and ESP continue over `4500/udp` (UDP encapsulation).

To let strongSwan manage the IPsec policies of the host instead, so that the host (and networks it routes for) can use the tunnels, run the container with host networking. The firewall has to be adjusted by yourself in this case.

```yaml
strongswan_container_network: host
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using `strongswan_conf` (for `strongswan.conf`) and `strongswan_environment_variables_additional_variables` variables

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, strongSwan becomes available and loads the configured connections. To see the state of the connections, log in to the server with SSH and run `docker exec strongswan swanctl --list-sas` (or how you/your playbook named the container, e.g. `mash-strongswan`).

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu strongswan` (or how you/your playbook named the service, e.g. `mash-strongswan`).
