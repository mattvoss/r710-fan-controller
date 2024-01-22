# Dell R710/R720 Fan Control Script

> A temperature-based fan speed controller for Dell servers (tested on an R710/R720, should work with most PowerEdges). Supports both local and remote hosts.


- [Requisites](#requisites)
- [Installation / Upgrade](#installation--upgrade)
  - [Docker](#docker)
- [Configuration](#configuration)
- [How it works](#how-it-works)
- [Notes on remote hosts](#notes-on-remote-hosts)
- [Credits](#credits)

---

## Requisites

1. Python 3 is installed.
2. **IPMI Over LAN** is enabled in all used iDRACs (_Login > Network/Security > IPMI Settings_).
   + May not be needed if you're only managing the local machine.
3. `lm-sensors` is installed and configured on the local machine.
   + Example output of `sensors` for a dual CPU system:
        ```text
        coretemp-isa-0000
        Adapter: ISA adapter
        Core 0:       +38.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 1:       +46.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 2:       +40.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 8:       +43.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 9:       +39.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 10:      +39.0°C  (high = +69.0°C, crit = +79.0°C)

        coretemp-isa-0001
        Adapter: ISA adapter
        Core 0:       +29.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 1:       +35.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 2:       +29.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 8:       +34.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 9:       +33.0°C  (high = +69.0°C, crit = +79.0°C)
        Core 10:      +31.0°C  (high = +69.0°C, crit = +79.0°C)
        ```
4. `nvidia-smi` is installed and configured on the local machine.
   + Example output of `nvidia-smi`:
        ```text
        +---------------------------------------------------------------------------------------+
        | NVIDIA-SMI 535.129.03             Driver Version: 535.129.03   CUDA Version: N/A      |
        |-----------------------------------------+----------------------+----------------------+
        | GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
        | Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
        |                                         |                      |               MIG M. |
        |=========================================+======================+======================|
        |   0  Tesla P40                      On  | 00000000:42:00.0 Off |                  Off |
        | N/A   41C    P8              20W / 250W |  12246MiB / 24576MiB |      0%      Default |
        |                                         |                      |                  N/A |
        +-----------------------------------------+----------------------+----------------------+
                                                                                                 
        +---------------------------------------------------------------------------------------+
        | Processes:                                                                            |
        |  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
        |        ID   ID                                                             Usage      |
        |=======================================================================================|
        |    0   N/A  N/A    450829    C+G   vgpu                                      12192MiB |
        +---------------------------------------------------------------------------------------+
        ```
   
## Installation / Upgrade

Clone the repo and run the installation script as root to configure the system or upgrade the already installed controller:

```text
git clone https://github.com/mattvoss/r710-fan-controller.git
cd r710-fan-controller
sudo ./install.sh [<installation path>]
```

The default installation path is `/opt/fan_control` and the service will be installed as `fan-control.service`. If a configuration file already exists, it will be renamed with a `.old` extension.

### Docker

To deploy remote fan management with Docker (`fan_control` running on a separate host and only interacting with remote ones, see [Notes on remote hosts](#notes-on-remote-hosts)), build the image in the repo and bind mount your own YAML config and SSH keys folder:

```bash
docker build -t fan_control .
docker run -d --restart=always --name fan_control -v "./fan_control.yaml:/app/fan_control.yaml:ro" -v "./keys:/app/keys:ro" fan_control
```

Running this tool under a proper orchestrator is advised.

## Configuration

You can tune the controller's settings via the `fan_control.yaml` file in the installation directory.

The file is made of two main sections, `general` and `hosts`. The first one contains global options; the second one, `hosts`, is a list of hosts to manage. Each of them must contain a `temperatures` and a `speeds` lists at a minimum, both of exactly three values. If the `hysteresis` key isn't specified, its value is assumed to be `0`.

Remote hosts must also contain both the `remote_temperature_command` string and the `remote_ipmi_credentials` structure.

| Key | Description |
| --- | --- |
| `general`.`debug` | Toggle debug mode _(print ipmitools commands instead of executing them, enable additional logging)_. |
| `general`.`interval` | How often (in seconds) to read the CPUs' temperatures and adjust the fans' speeds. |
| `hosts`_[n]_.`margin` | How many degrees (in °C) the CPUs' temperature must go below the threshold to trigger slowing the fans down. _Prevents rapid speed changes, a good starting value can be `2`._ |
| `hosts`_[n]_.`curve` | A YAML array of temperature, fan speed percentage values. Each array element should be temperature in celsius and the fan speed desired. e.g. [30, 40] which means at 30&deg; celsius the fan speed will be set to 40%. _See [below](#how-it-works) for details._ An unlimited number of elements can be specified. |
| `hosts`_[n]_.`remote_temperature_command` | **For remote hosts only.** A command that will be executed to obtain the temperatures of this remote system. _See [notes](#notes-on-remote-hosts) for details._ |
| `hosts`_[n]_.`remote_ipmi_credentials`.`host` | **For remote hosts only.** The iDRAC hostname/IP of this remote system. _See [notes](#notes-on-remote-hosts) for details._ |
| `hosts`_[n]_.`remote_ipmi_credentials`.`username` | **For remote hosts only.** The username used to login to this remote system's iDRAC. _See [notes](#notes-on-remote-hosts) for details._ |
| `hosts`_[n]_.`remote_ipmi_credentials`.`password` | **For remote hosts only.** The password used to login to this remote system's iDRAC. _See [notes](#notes-on-remote-hosts) for details._ |

## How it works

Every `general`.`interval` seconds the controller will fetch the temperatures of all the available CPU cores, average them and round the result  and fetch all GPU temperatures, average them and round the result. It will then take the max temperature between CPU and GPU averages to set the fans' speed percentage. A temperature curve can be created to ramp the fans' speed up or down smoothly based upon the temperature observed.

## Notes on remote hosts

This controller can monitor the temperature and change the fan speed of remote hosts too: the only caveat is that you'll need to extract the temperatures via an external command. This could be via SSH, for example. The controller expects such a command to return **a newline-delimited list of numbers parseable as floats**.

**The included example is a good fit for a remote FreeNAS host**: it will connect to it via SSH and extract the temperature of all CPU cores, one per line. This way you'll be able to manage that machine just as well as the local one without applying any hardly trackable modification to the base OS.

## Credits

Major thanks go to [NoLooseEnds's directions](https://github.com/NoLooseEnds/Scripts/tree/master/R710-IPMI-TEMP) for the core commands and [sulaweyo's ruby script](https://github.com/sulaweyo/r710-fan-control) for the idea of automating them.

**Note:** The key difference of this script, other than handling remote hosts, is that it's based on the temperature of the CPUs' cores and not on the ambient temperature sensor on the server's motherboard. The R710 does not expose CPU temperature over IPMI, but other models do; this script should work with them nonetheless.
