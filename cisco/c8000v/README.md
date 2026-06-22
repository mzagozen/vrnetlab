# Cisco Catalyst 8000V Edge Software

This is the vrnetlab docker image for Cisco Catalyst 8000V Edge Software, or
'c8000v' for short.

The Catalyst 8000v platform is a successor to the CSR 1000v and supports two
operating modes:

- **Autonomous mode** - Standard IOS-XE routing platform
- **Controller mode** - SD-WAN managed mode (Viptela)

The build process automatically produces both variants from a single qcow2 image.

In addition, the image can be booted in a **ZTP mode** (`MODE=ztp`) that performs
day-zero, DHCP-based Zero Touch Provisioning instead of applying a bootstrap
config. This is meant for *testing* a ZTP server/workflow against c8000v -- see
[Zero Touch Provisioning (ZTP) mode](#zero-touch-provisioning-ztp-mode) below.

On installation of Catalyst 8000v the user is presented with the choice of
output, which can be over serial console, a video console or through automatic
detection of one or the other. Empirical studies show that the automatic
detection is far from infallible and so we force the use of the serial console
by feeding the VM an .iso image that contains a small bootstrap configuration
that sets the output to serial console. This means we have to boot up the VM
once to feed it this configuration and then restart it for the changes to take
effect. Naturally we want to do this in the build process as to avoid having to
restart the router once for every time we run the docker image. Unfortunately
docker doesn't allow us to run docker build with `--privileged` so there is no
KVM acceleration making this process excruciatingly slow were it to be performed
in the docker build phase. Instead we build a basic image using docker build,
which essentially just assembles the required files, then run it with
`--privileged` to start up the VM and feed it the .iso image. After we are done
we shut down the VM and commit this new state into the final docker image. This
is unorthodox but works and saves us a lot of time.

**Note:** This installation process is not performed for controller and ztp
mode. Both variants are built from the same serial-enabled qcow2, and skipping
the install leaves the disk pristine. For ztp that is essential: the config the
install writes disarms ZTP, so `launch.py --install` refuses to run with
`MODE=ztp`.

## Building the docker image

Put the .qcow2 file in this directory and run `make docker-image` and you should
be good to go. The build process automatically produces both autonomous and controller
mode variant images from a single qcow2 file.

**Note:** Controller mode requires a serial-enabled image (e.g.,
`c8000v-universalk9_16G_serial.17.12.05a.qcow2`).

The resulting images are called `vr-c8000v:VERSION` and `vr-c8000v:controller-VERSION`.
You can tag them with something else if you want, like `my-repo.example.com/vr-c8000v`
and then push to your repo. The tag is the same as the version of the Catalyst 8000v
image, so if you have c8000v-universalk9.16.04.01.qcow2 your final docker images will be
called `vr-c8000v:16.04.01` and `vr-c8000v:controller-16.04.01`

It's been tested to boot and respond to SSH with:

- 16.03.01a (c8000v-universalk9.16.03.01a.qcow2)
- 16.04.01 (c8000v-universalk9.16.04.01.qcow2)
- 17.11.01a (c8000v-universalk9_16G_serial.17.11.01a.qcow2)
- 17.16.01a (c8000v_universalk9_8g_seria.qcow2) - Autonomous and controller modes tested

## Usage

```bash
docker run -d --privileged --name my-c8000v-router vr-c8000v
```

## Zero Touch Provisioning (ZTP) mode

Setting `MODE=ztp` boots the router "day zero" so that it runs IOS-XE
DHCP-based Zero Touch Provisioning itself, instead of having vrnetlab feed it a
bootstrap config. Use this when you want to **test a ZTP server/workflow** with
c8000v.

In this mode the launcher:

- does **not** generate or mount the CVAC config ISO (`iosxe_config.txt`), and
- refuses to start if a user `startup-config.cfg` is present,

because IOS-XE only enters ZTP when it boots with an **empty startup-config**.
The router then DHCPs on its interfaces and uses DHCP option 67 (or option 150
for TFTP) to fetch the provisioning config (a Python script executed in Guest
Shell, or a plain IOS config). The node is marked "up" as soon as it obtains a
DHCP lease on any interface or reaches an interactive prompt.

### Requirements

  **ztp image tag** (`vrnetlab/cisco_c8000v:ztp-VERSION`), which `make
  docker-image` builds with no install step and `MODE=ztp`.
- **DHCP server** The router must reach a real DHCP/ZTP server, so
  run it with `CLAB_MGMT_PASSTHROUGH=true` and put a DHCP/ZTP server on the same
  management network.

### Environment variables

| Variable               | Default      | Description                                                                 |
| ---------------------- | ------------ | --------------------------------------------------------------------------- |
| `MODE`                 | per image tag | `ztp` on the ztp-tagged build: boot day-zero for DHCP-based ZTP. Also `autonomous`, `controller`. |
| `ZTP_BOOT_TIMEOUT`     | `600`        | Backstop: seconds of console idle after which the node is marked up if no DHCP lease / prompt was seen (best-effort, not a precise deadline). |
| `CLAB_MGMT_PASSTHROUGH`| `false`      | Set `true` so the router's mgmt port is bridged to the clab mgmt network.    |

### Example (containerlab)

```yaml
name: c8000v-ztp
mgmt:
  network: clab-mgmt
  ipv4-subnet: 172.20.20.0/24
topology:
  nodes:
    ztp-server:                 # dnsmasq: DHCP + TFTP/HTTP on the mgmt net
      kind: linux
      image: your/dnsmasq-http:latest
      mgmt-ipv4: 172.20.20.10
      binds:
        - ztp/:/srv/ztp/
    c8000v:
      kind: cisco_c8000v
      type: ztp                               # the kind supplies MODE from this
      image: vr-c8000v:controller-17.16.01a   # pristine, serial-enabled image
      env:
        CLAB_MGMT_PASSTHROUGH: "true"
```

dnsmasq on `ztp-server` serves option 67 (or 150 for the TFTP form):

```
dhcp-range=172.20.20.100,172.20.20.200,12h
# HTTP form (no option 150 needed):
dhcp-option=67,"http://172.20.20.10:8000/ztp.py"
# --- or TFTP form ---
# dhcp-option=150,172.20.20.10
# dhcp-boot=/ztp.py
enable-tftp
tftp-root=/srv/ztp
```

`ztp/ztp.py` runs in Guest Shell (recognized by the `.py` extension) and applies
 the day-one config:

```python
import cli
cli.configurep([
    "hostname C8000V-ZTP-OK",
    "platform console serial",
    "license boot level network-premier addon dna-premier",
    "username vrnetlab privilege 15 password VR-netlab9",
    "line vty 0 4", "login local", "transport input ssh",
])
cli.cli("write memory")
```

After a successul startup the launcher logs *"ZTP: device up (...)"*.

> **Note:** the exact ZTP console banners vary by IOS-XE release, so the
> launcher's progress markers are best-effort. As a fallback, `ZTP_BOOT_TIMEOUT`
> always brings the node up so you can inspect it.

## Interface mapping

IOS XE 16.03.01 and 16.04.01 does only support 10 interfaces, GigabitEthernet1 is always configured
as a management interface and then we can only use 9 interfaces for traffic. If you configure vrnetlab
to use more then 10 the interfaces will be mapped like the table below.

The following images have been verified to NOT exhibit this behavior

- c8000v-universalk9.03.16.02.S.155-3.S2-ext.qcow2
- c8000v-universalk9.03.17.02.S.156-1.S2-std.qcow2

| vr-c8000v | vr-xcon |
| :-------: | :-----: |
|    Gi2    |   10    |
|    Gi3    |    1    |
|    Gi4    |    2    |
|    Gi5    |    3    |
|    Gi6    |    4    |
|    Gi7    |    5    |
|    Gi8    |    6    |
|    Gi9    |    7    |
|   Gi10    |    8    |
|   Gi11    |    9    |

## System requirements

CPU: 1 core

RAM: 4GB

Disk: <500MB

## License handling

You can feed a license file into c8000v by putting a text file containing the
license in this directory next to your .qcow2 image. Name the license file the
same as your .qcow2 file but append ".license", e.g. if you have
"c8000v-universalk9.16.04.01.qcow2" you would name the license file
"c8000v-universalk9.16.04.01.qcow2.license".

The license is bound to a specific UDI and usually expires within a given time.
To make sure that everything works out smoothly we configure the clock to
a specific date during the installation process. This is because the license
only has an expiration date not a start date.

The license unlocks feature and throughput. The default throughput for C8000v is
20Mbit/s which is perfectly for basic management and testing.

## Known issues

If during the image boot process (not during the install process) you notice messages like:

```
% Failed to initialize nvram
% Failed to initialize backup nvram
```

Then the image will boot, but SSH might not work. You still can use telnet to access the running VM. For instance:

```bash
telnet <container name> 5000
```
