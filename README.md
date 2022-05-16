# Docker on Mac M1 with full x86_64 support

The below manual explains how to deploy docker with x86_64 support on Mac M1 processor. The system uses full CPU emulation.

On an Intel Mac, the system automatically uses Apple's Hypervisor instead, providing good virtualisation speed.

## Note

These instructions and any files referenced within come with no warranty. Use at your own risk.

## Usage

### Install KubeLab Virtual Machine

KubeLab is a pre-built Linux virtual machine with docker, and qemu wrapper scripts to allow for easy deployment.

```
cd ~
rm -f kubelab-qemu.tgz kubelab-scripts.tgz
curl https://kubelab.s3.eu-west-1.amazonaws.com/kubelab-qemu.tgz --output kubelab-qemu.tgz
tar -zxvf kubelab-qemu.tgz
curl https://kubelab.s3.eu-west-1.amazonaws.com/kubelab-scripts.tgz --output kubelab-scripts.tgz
tar -zxvf kubelab-scripts.tgz
cp -a ~/.kubelab/KubeLab-Launcher.app /Applications/
```

### Getting started

If you have `Automator` installed on your MacOS (this should be available by default), you can start KubeLab menu by running `KubeLab-Launcher` application as you normally would any other app.

Alternatively, you can start the menu by running the below script:

```
~/.kubelab/bin/main.sh
```

### Menu - Next steps

1. Create a data disk
2. Resize the data disk
3. Adjust CPU/RAM
4. Start KubeLab - note that after the first-time start, you may need to restart the Terminal app before docker becomes available (`Cmd+q`)
5. Stop KubeLab

### Menu - Other actions

* Download new scripts - download latest menu/helper scripts and install them
* Delete data disk - reset the VM to factory defaults - does not remove settings, such as CPU/RAM
* Commit data disk to root volume - make the data disk, with current docker containers, the new factory-default
* Delete disk, Start, Upgrade, Commit - destructive action for your containers; runs an upgrade on the VM and commits it to new factory-default
* Advanced settings - adjust advanced settings for the VM
* Kill KubeLab - for when "Stop KubeLab" doesn't work
* (debug) VM Console - attach to the VM serial console for debugging purposes
* (debug) VM Console Log - display the console log of the VM (`cat ~/.kubelab/log/vm.log`)

## Running Aerospike

Follow [these instructions](https://docs.aerospike.com/server/operations/install/docker-desktop#procedure) to setup and deploy Aerospike.

## Manual non-menu usage instructions

It is possible to use the KubeLab setup without the menu script. For full instructions, see [here](manual-use.md)

## Troubleshooting

To submit a bug report, please open a GitHub issue with a description of the problem, also providing `~/.kubelab/log/vm.log` and `~/.kubelab/log/vmnet.log`
