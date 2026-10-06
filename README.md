# ansible-role-virt

KVM/libvirt host with virt-manager (SPICE console) on openSUSE Leap, and libvirt networks defined in the
inventory (NAT/isolated/open/routed, DNS domain, static DHCP leases, DNS records).

1. zypper patterns `kvm_server` + `kvm_tools` and the SPICE client packages (with recommends, like
   `zypper in --recommends`), all from the Leap oss repo
2. enables/starts the socket-activated modular libvirt daemons (`virtqemud`, `virtnetworkd`, ...)
3. per network: renders the network XML to `/etc/libvirt/ansible-networks/<name>.xml`, defines it with
   `virsh net-define` when the file changed (or the network is missing), starts it and sets autostart

Only openSUSE is implemented. With `install: false` (default) the role does nothing.

## Variables

```yaml
virt:
  install: true                      # default false
  # patterns: [kvm_server, kvm_tools]
  # packages: [...]                  # see defaults/main.yml (virt-manager, spice-gtk, qemu-spice, typelibs, PyYAML)
  extra_packages: []                 # e.g. [cockpit-machines]
  # recommends: true
  networks:
    - name: default
      bridge: virbr0
      forward: nat                   # nat (default) | none (isolated) | open | route
      domain: vm.example.org         # optional: DNS domain of the network (answered by libvirt's dnsmasq only)
      address: 192.168.122.1         # host address on the bridge
      prefix: 24
      dhcp_range: [192.168.122.100, 192.168.122.254]   # optional
      hosts:                         # static DHCP leases; each also gets a DNS record <name>(.<domain>)
        - {name: osl, mac: "52:54:00:ff:a1:22", ip: 192.168.122.61}
      dns_hosts:                     # additional DNS records
        - {ip: 192.168.122.1, names: [vmhost]}
      autostart: true                # default
    - name: isolated                 # no forwarding: guests reach the host (192.168.100.1) only, no internet
      bridge: virbr1
      forward: none
      domain: iso.example.org
      address: 192.168.100.1
      prefix: 24
      dhcp_range: [192.168.100.100, 192.168.100.254]
    - name: old
      state: absent                  # stop + undefine
```

Networks that are not listed are left untouched (e.g. the `default` network created by the package if it is
not managed here). A managed network is defined exactly as listed; changes made with virsh/virt-manager are
overwritten when the inventory changes (the rendered file is the reference, manual changes alone are not
detected).

Checks before anything is changed: unique names and bridges, valid `forward`, IPv4 address, prefix 8-30,
lease IPs and DHCP range inside the subnet, lease MACs/IPs/names unique.

### Changing an active network

A changed network is restarted (`net-destroy` + `net-start`) only if no running guest is attached to it.
Otherwise it is only redefined and a notice is shown; apply it later with
`virsh net-destroy <name> && virsh net-start <name>` (guests on it then need a reboot or interface re-attach).

### Subnet conflicts

libvirt refuses to start a network whose subnet is already used on the host (e.g. testing inside a VM that
itself sits on 192.168.122.0/24 → use another subnet there).

## Users

Members of group `libvirt` manage `qemu:///system` without a password (polkit rule of the libvirt package).
Add the group with ansible-role-user (`users[].groups: [libvirt]`); run this role before ansible-role-user so
the group exists.

## Requirements

- controller: collections `community.general`, `ansible.utils` (Python `netaddr`)
- target: openSUSE Leap (tested: 16.1)

## License

Apache-2.0

Created with the help of AI
