<img src="https://raw.githubusercontent.com/iambryant/junos-dev-playbook/main/docs/images/junos-devices.jpg" width="50%" alt="Junos Devices" />

# Ansible Playbook: Junos OS

[![CI](https://github.com/iambryant/junos-dev-playbook/actions/workflows/ci.yml/badge.svg)](https://github.com/iambryant/junos-dev-playbook/actions/workflows/ci.yml)

The playbooks in this repository configure my Junos OS infrastructure from the ground up.

## Requirements

This collection uses the [juniper.device](https://galaxy.ansible.com/ui/repo/published/juniper/device/docs/)
collection and requires the following packages on the control node:

  - Python >= 3.12
  - Ansible 2.17 or later
  - Junos py-junos-eznc 2.7.3 or later
  - jxmlease 1.0.1 or later
  - xmltodict 0.13.0 or later
  - jsnapy 1.3.7 or later
  - packaging 25.0 or later

You can install the Python libraries with the `requirements.txt` in this repository using 
`pip3 install -r requirements.txt`.

## Notes

The following notes are oddities/issues I've encountered while developing playbooks for Juniper platforms or issues in
general. I'm hoping if someone encounters a similar issue that they might be able to find a solution here. Please
raise an issue in the repository if you believe any information provided here is outdated or inaccurate, so that it
benefits others:

### VNF CPU Allocation

Looking at Juniper's [official documentation](https://www.juniper.net/documentation/us/en/software/junos/nfx250-getting-started/topics/topic-map/nfx250-ng-overview.html),
I initially thought that running a vSRX VNF's vCPUs across separate physical cores would be good for performance due to
parallelization:

#### Core to CPU Mapping on NFX250

The following tables list the CPU to core mappings for the NFX250 models:

**NFX250-LS1**
|        |      |      |      |      |
| :---   | :--- | :--- | :--- | :--- |
| `Core` | 0    | 1    | 2    | 3    |
| `CPU`  | 0,4  | 1,5  | 2,6  | 3,7  |

**NFX250-S1 and NFX250-S2**
|        |      |      |      |      |      |      |
| :---   | :--- | :--- | :--- | :--- | :--- | :--- |
| `Core` | 0    | 1    | 2    | 3    | 4    | 5    |
| `CPU`  | 0,6  | 1,7  | 2,8  | 3,9  | 4,10 | 5,11 |

However, it would take 15+ minutes to boot. When I configured the vSRX vCPUs on threads on the same physical 
core, it would boot in less than 5 minutes. I assume this was since the vSRX benefits from shared L1/L2 cache.

### Juniper.device collection

Sometimes, when you run a command with the `juniper.device.command` module, you'll get this error:
`Type 'bool' cannot be serialized.`

For example, running this task on a new Juniper device can give you that error:

```yaml
- name: "Check if ACME certificate for domain already exists"
  juniper.device.command:
    commands:
      - "show security pki local-certificate"
```

However, a task like this will run perfectly fine:

```yaml
- name: "Show system uptime"
  juniper.device.command:
    commands:
      - "show system uptime"
```

To avoid this, I've been adding the following to the task:

```yaml
formats: "json"
```

This has prevented any of those errors from occurring. I believe these GitHub issues raised in reference
to the collection are related, but so far, I haven't seen any traction on getting it fixed:

- https://github.com/Juniper/ansible-junos-stdlib/issues/622
- https://github.com/Juniper/ansible-junos-stdlib/issues/808

### VNF Management Interfaces

Juniper's documentation for the interfaces available to a VNF states this:

```text
- To delete a VNF interface, you must stop the VNF, delete the interface, and then restart the VNF.
- After attaching or detaching a virtual function, you must restart the VNF for the changes to take effect.
- eth0 and eth1 are reserved for the default VNF interfaces that are connected to the internal network and the
  out-of-band management network. Therefore, the configurable VNF interface names start from eth2.
- Within a VNF, the interface names can be different, based on guest OS naming conventions. VNF interfaces
  that are configured in the JCP might not appear in the same order within the VNF.
- You must use the target PCI addresses to map to the VNF interfaces that are configured in the JCP and you
  must name them accordingly.
```

Based on my experience with the NFX line, it means this:

- `eth0` is used as the default management interface for the VNF and is accessible through the management interface of
  the NFX itself.
  For example, if your NFX is connected to a management network on `fxp0`, network traffic for `eth0` will be sent
  through that interface. I haven't tested if it's bound to the fxp0 interface specifically, or any interface the NFX
  uses for management, such as a front panel interface or an `irb` interface. I assume it's the former. It is known as
  the `out-of-band` management interface.

- `eth1` maps to an internal subnet on the NFX that is used if you use `request virtual-network-function telnet <vnf-name>`
  or `request virtual-network-function ssh <vnf-name>` to try to access a VNF. From testing, the allocated subnet is
  a `192.0.2.x/24` subnet. It is known as the `internal` management interface. For example, when testing the creation
  of a VNF:

  ```text
  admin@ubuntu-01:~$ ip a show dev ens3
  2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
      link/ether 2c:21:31:5f:c1:20 brd ff:ff:ff:ff:ff:ff
      altname enp0s3
      altname enx2c21315fc120
      inet 192.0.2.100/24 metric 1024 brd 192.0.2.255 scope global dynamic ens3
         valid_lft 2060sec preferred_lft 2060sec
      inet6 fe80::2e21:31ff:fe5f:c120/64 scope link proto kernel_ll 
         valid_lft forever preferred_lft forever
  ```

  and when attempting to run `request virtual-network-functions ssh ubuntu-01`:

  ```text
  admin@nfx-01> request virtual-network-functions ssh ubuntu-01
  The authenticity of host '192.0.2.100 (192.0.2.100)' can't be established.
  ED25519 key fingerprint is SHA256:wid2vI9FAGksAeqaixLl2Afv6M2FpWC8wc1VGjFF2hI.
  This key is not known by any other names.
  Are you sure you want to continue connecting (yes/no/[fingerprint])?
  ```

- `eth2` and onward are interfaces that you can do with as you please. Technically, you can assign the `out-of-band` or
  `internal` designations to them from the edit hierarchy, if you need more management interfaces. Typically, you would
  map `eth2` and onward interfaces to front panel interfaces using either the `vlan` method or `SR-IOV` method.

> [!IMPORTANT]
> Do not assume that `eth0` maps to the first available interface in the VNF, `eth1` maps to the second available
> interface in the VNF, and so on. When testing interfaces inside a VNF, the interfaces were mapped as so:
> ```text
> admin@ubuntu-01:~$ ip a
> 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
>     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
>     inet 127.0.0.1/8 scope host lo
>        valid_lft forever preferred_lft forever
>     inet6 ::1/128 scope host noprefixroute 
>        valid_lft forever preferred_lft forever
> 2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
>     link/ether 2c:21:31:5f:c1:20 brd ff:ff:ff:ff:ff:ff
>     altname enp0s3
>     altname enx2c21315fc120
>     inet 192.0.2.100/24 metric 1024 brd 192.0.2.255 scope global dynamic ens3
>        valid_lft 3578sec preferred_lft 3578sec
>     inet6 fe80::2e21:31ff:fe5f:c120/64 scope link proto kernel_ll 
>        valid_lft forever preferred_lft forever
> 3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
>     link/ether 2c:21:31:5f:c1:21 brd ff:ff:ff:ff:ff:ff
>     altname enp0s4
>     altname enx2c21315fc121
>     inet6 fe80::2e21:31ff:fe5f:c121/64 scope link proto kernel_ll 
>        valid_lft forever preferred_lft forever
> 4: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
>     link/ether 2c:21:31:5f:c1:22 brd ff:ff:ff:ff:ff:ff
>     altname enp0s5
>     altname enx2c21315fc122
>     inet6 fe80::2e21:31ff:fe5f:c122/64 scope link proto kernel_ll 
>        valid_lft forever preferred_lft forever
> ```
> As per the output, we can see that `eth1` was mapped to the first interface, `ens3`. `eth1` being the internal 
> management network, and that `eth0`, the interface for the `out-of-band` management network was not the first
> interface to be mapped. 

## License

MIT

## Acknowledgements

Credit goes to [laurent-jnpr](https://github.com/laurent-jnpr); I used elements from a template in their repository
[VNF-on-Juniper-NFX-with-Ansible](https://github.com/Juniper-SE/VNF-on-Juniper-NFX-with-Ansible) to build the VNF
creation template.
