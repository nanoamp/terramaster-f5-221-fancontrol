# Fan control for TerraMaster F5-221 on Linux

This is based on an [F4-220 fan control script](https://github.com/cinzas/terramaster-fancontrol), which is itself a port of the [Xpenology fancontrol script by Eudean](https://github.com/cinzas/terramaster-fancontrol) for OMV/Debian.

You can choose between reading the temperature from the disks, with _smartctl_, or reading it from the NAS motherboard (which uses an IT8613E chipset) using _lmsensors_.

Note that if you use disk sensing, you will not be able to spindown the disks, as smartctl wakes them to read the temperature. If reading from the board temperature sensor you can still spindown disks.

## Prerequisites

1. You will need a compiled it87 module to read the board temperature sensor, such as [this one](https://github.com/frankcrawford/it87.git). It's recommended to use the `dkms-install.sh` script so that the module has more chance of surviving linux kernel upgrades. If installing manually, ensure you `modprobe it87`.

2. You will need lmsensors and/or smartctl to do the measurements.
    ```
    apt install smartmontools lm-sensors
    sensors detect
    ```
    Say yes to all questions and make sure `coretemp` and `it87` modules are added to `/etc/modules`.

## Installation

1. **Download or clone the repo**

    ```git clone https://github.com/nanoamp/terramaster-f5-221-fancontrol.git```

2. **Prepare for smartctl** (optional)

    If you plan to use to use smartctl, you must create a directory containing all the drive ids that you want to monitor. By default this is `/srv/fancontrol/disks`. You can change this by searching and replacing this path in `fancontrol.cpp`.
    For example, to monitor `/dev/sda`, `/dev/sdb`, and `/dev/sdc`:

    ```
    mkdir -p /srv/fancontrol/disks
    cd /srv/fancontrol/disks
    touch sda sdb sdc
    ```

3. **Build with GCC**
   - Build using the docker image for ease of use:

       ```
       docker pull gcc
       docker run --rm -v "$PWD":/usr/src/myapp -w /usr/src/myapp gcc gcc -o fancontrol fancontrol.cpp
       ```
   - Alternatively, compile using a local installation of gcc:

       ```
       gcc -o fancontrol fancontrol.cpp
       ```

4. **Copy the compiled binary**

    ```
    sudo cp fancontrol /usr/local/bin/
    ```
    - Note that by installing lmsensors previously, there'll be a /usr/sbin/fancontrol script installed which uses /etc/fancontrol and pwmconfig to set itself up. Check that the correct command is picked up by default with `which fancontrol`. If not, make sure to use the explicit /usr/local/bin path prefix when calling. 

## Running

- Arguments are described with `fancontrol -h`
- To run fancontrol with console debug, using disk temperature, and a setpoint of 37C, use `fancontrol 1 37 `
- To run fancontrol without console debugging, using board sensor, and a setpoint of 40C, use `fancontrol 0 40 1`

You may need to use `sudo`, depending on your setup.

## Systemd

Use one of the provided systemd services to automate running fancontrol on startup.

If using smartctl, first edit the `fancontrol-smartctl.service` file to specify the drives to monitor in the `ExecStartPre` lines.

To use smartctl to monitor disk temperatures:

```
sudo cp fancontrol-smartctl.service /etc/systemd/system/   # copy the service
sudo systemctl enable fancontrol-smartctl                  # enable the service
sudo systemctl start fancontrol-smartctl                   # start the service
```

To use lmsensors to monitor board temperature:

```
sudo cp fancontrol-sensors.service /etc/systemd/system/    # copy the service
sudo systemctl enable fancontrol-sensors                   # enable the service
sudo systemctl start fancontrol-sensors                    # start the service
```
