# Refresh router ARP with Ansible

For Linux servers with Python 3, iproute2 (`ip` with JSON support), and iputils
`ping`. The login account needs working SSH authentication and passwordless sudo
to root. Disabling password prompts does not grant sudo privileges.

1. Add servers under `arp_targets.hosts` in `inventory/hosts.yml`.
2. Set `arp_interface` to a list in inventory (per host if necessary), or override it below.
3. Run from this directory so Ansible finds `ansible.cfg`:

```sh
ansible-playbook clear-router-arp.yml -e '{"arp_interface":["ens192","ens224"]}' --check
ansible-playbook clear-router-arp.yml -e '{"arp_interface":["ens192","ens224"]}'
```

For example, configure a single interface with `arp_interface: [ens192]` or
multiple interfaces with `arp_interface: [ens192, ens224]`. An undefined, null,
or empty list skips that host before SSH, discovery, or ARP changes. An empty
string also counts as unset; a nonempty scalar string is rejected. The inventory
template defaults to `[]`. Duplicate names are processed once. Each interface
completes discovery and recovery before the next starts; recovery timeouts apply
per interface. A failure stops further interfaces on that host.

If interface validation fails, check the reported value and type. Ansible's
`-e 'arp_interface=["enp0s5"]'` key=value syntax passes a string, even though
it looks like a list. Use `-e '{"arp_interface":["enp0s5"]}'` instead; extra
variables override the inventory value. Validation uses sequence, string, and
mapping tests instead of comparing Python type names.

Use `--limit server01` to target a single inventory host. Check mode performs
read-only discovery and displays the target without clearing ARP.

The router address is the subnet's network address plus one, calculated from the
live interface address and prefix. Multiple addresses in the same subnet are
supported; multiple subnets on one interface, /31 and /32 prefixes, a locally owned router address,
and static/NOARP router entries are rejected. IPv6 NDP is outside this play's scope.

Run when the replacement router is ready to answer ARP. The remote asynchronous
job clears only the derived router's dynamic neighbour entry on the selected
interface, then sends traffic until a MAC is learned. ICMP echo replies are not
required. It waits up to `arp_recovery_timeout` (default 120 seconds). Ansible
waits up to `arp_connection_timeout` (default 180 seconds) for working SSH and
checks the job result. This verifies local ARP resolution and controller access;
it does not test every routed application or establish that a MAC belongs to the
intended replacement device.

If SSH drops during job-status polling, the play reconnects and resumes polling
for up to three sessions per interface. Each connection wait uses
`arp_connection_timeout`. A ping process timeout is retried within the ARP
recovery window. Job failures or unverified completion fail the host explicitly.

The controller must initially reach each server to launch the job. An SSH loss
after launch does not stop the remote operation. If the launch acknowledgement
itself is lost, the play reconnects and reports the unverified job rather than
claiming success; rerunning is safe. A network outage that outlasts the configured
timeouts requires restoring connectivity and rerunning. For a cutover that will
prevent initial SSH access, use an independent management path.

Host key checking is disabled as requested. Set `remote_user` under `[defaults]`
in `ansible.cfg` to select the SSH login account. The play does not determine or
override the remote username. No SSH or sudo password prompts are enabled.

## Validation

Run these from the project directory with Ansible and ansible-lint installed:

```sh
ansible-playbook --syntax-check clear-router-arp.yml
ansible-lint --offline clear-router-arp.yml tasks/*.yml
```

Live cutover behaviour still requires validation against your servers and network.

## References

- [Ansible asynchronous actions](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_async.html)
- [Ansible wait_for_connection](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/wait_for_connection_module.html)
- [Ansible configuration](https://docs.ansible.com/projects/ansible/latest/reference_appendices/config.html)
- [iproute2 neighbour operations](https://www.man7.org/linux/man-pages/man8/ip-neighbour.8.html)
