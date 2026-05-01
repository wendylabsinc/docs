# WendyOS Networking Overview

WendyOS provides a self-contained networking stack designed for USB-tethered connectivity between a device (Raspberry Pi, Jetson, or other supported hardware) and a host computer. When you plug a WendyOS device into a host via USB-C, the device presents a composite USB gadget that includes both a network interface and a serial console. No drivers, no pairing, no manual configuration — the device advertises itself over mDNS and the wendy-agent can discover and connect to it immediately.

## Architecture

```
WendyOS device                          Host computer
───────────────────────────────         ───────────────────────────────
  usb0 (NCM gadget interface)  ←USB→    enxXXXXXXXXXXXX (CDC NCM)
  IPv4: DHCP client (or shared)         IPv4: 10.42.0.1/24 (NM shared)
  IPv6: link-local (SLAAC)              IPv6: link-local (SLAAC)
  mDNS: _wendyos._udp (Avahi)           mDNS discovery (avahi-browse/dns-sd)
  /dev/ttyGS0 (ACM serial)     ←USB→    /dev/ttyACM0 (serial console)
```

## Components

| Component | Role | Source |
|---|---|---|
| `gadget-setup.sh` | Configures the USB gadget in configfs at boot | `recipes-core/gadget-setup/` |
| NetworkManager | Manages `usb0` IP configuration | `recipes-connectivity/networkmanager/` |
| dnsmasq | Optional on-device DHCP server for `usb0` | `recipes-support/gadget-network-config/` |
| radvd | IPv6 Router Advertisement daemon on `usb0` | `recipes-connectivity/radvd/` |
| Avahi | mDNS service broadcasting (`_wendyos._udp`) | `recipes-connectivity/avahi/` |
| `generate-hostname.sh` | Sets a stable device hostname before Avahi starts | `recipes-connectivity/avahi/` |
| `update-mdns-uuid.sh` | Injects device UUID and name into the Avahi service file | `recipes-core/wendyos-identity/` |

## Topics

- [USB NCM Gadget](./usb-ncm.md) — How the USB gadget is constructed and initialized
- [MAC Address Generation](./mac-addresses.md) — Deterministic MAC generation and IPv6 link-local formation
- [IPv4 DHCP](./ipv4-dhcp.md) — DHCP server configuration for `usb0`
- [IPv6 Router Advertisements](./ipv6-ra.md) — radvd configuration and SLAAC on `usb0`
- [mDNS / Avahi](./mdns.md) — Service broadcasting, hostname generation, and host-side discovery
- [Troubleshooting](./troubleshooting.md) — Diagnosing common networking problems

## Key Addresses

| What | Value |
|---|---|
| Device `usb0` IPv4 (DHCP client mode) | Assigned by host (typically `10.42.0.2`–`10.42.0.5`) |
| Host IPv4 when using NetworkManager shared | `10.42.0.1/24` |
| dnsmasq pool (on-device DHCP server mode) | `10.42.0.2`–`10.42.0.5` |
| IPv6 | Link-local only (`fe80::/64`, derived from MAC via EUI-64) |
| mDNS service | `_wendyos._udp`, port `50051` |
| mDNS hostname | `wendyos-<device-name>.local` or `wendyos-<short-uuid>.local` |

## Platform Notes

WendyOS runs on multiple hardware platforms. Networking behaviour is identical across them, but the underlying USB controller differs:

| Platform | USB Controller | USB Version |
|---|---|---|
| Raspberry Pi 5 | `dwc2` | USB 2.0 |
| Jetson AGX Orin / Orin Nano | `tegra-xudc` | USB 3.2 |
| Other Linux (generic) | Detected at runtime | USB 2.0 fallback |

The QEMU virtual machine (`qemuarm64-wendyos`) does not use USB NCM gadget networking. It emulates a standard virtio network device instead.
