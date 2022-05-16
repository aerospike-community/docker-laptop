# Manual usage instructions

## Installation

### Pre-requisites

Install the required tools (homebrew, qemu emulator, wget, docker cli command and docker-compose command):

```
which brew; [ $? -ne 0 ] && /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install wget
brew install qemu
brew install docker
brew link --overwrite docker
brew install docker-compose
brew link --overwrite docker-compose
```

### Configure

```
cd ~/.kubelab
vi etc/qemu-settings.sh
### modify RAM and CPU lines to suit your needs and save the file
```

### Create virtual disk and resize it

Change the `50` in the below command to how many GB you want the VM to be able to use

```
cd ~/.kubelab
./bin/qcow-make.sh
./bin/qcow-resize.sh 50
```

## Operating

### Start

```
~/.kubelab/bin/start.sh
```

If this is your first time starting a VM, either execute the below `source` commands, or simply Quit your terminal and reopen it. Future invocations of the start script do not require this any more.

If using zsh: `source ~/.zshrc`

If using bash: `source ~/.bashrc`

### Status

```
~/.kubelab/bin/status.sh
```

### Stop

```
~/.kubelab/bin/stop.sh
```

### Kill (in case stop doesn't work)

```
~/.kubelab/bin/stop.sh kill
```

## Other actions

### Update docker server version in the VM

```
~/.kubelab/bin/upgrade.sh docker
```

### Update all packages in the VM

```
~/.kubelab/bin/upgrade.sh all
```

### Attach to the VM host console (debugging only)

```
~/.kubelab/bin/console
```

### Remove image (reset VM) and recreate it

```
cd ~/.kubelab
./bin/stop.sh
./bin/qcow-del.sh
./bin/qcow-make.sh
./bin/qcow-resize.sh 50
```

### Get new scripts version when it becomes available

This operation is safe and you will not loose any data.

```
cd ~
~/.kubelab/bin/stop.sh
~/.kubelab/bin/stop.sh kill
rm -f kubelab-scripts.tgz
wget https://kubelab.s3.eu-west-1.amazonaws.com/kubelab-scripts.tgz
tar -zxvf kubelab-scripts.tgz
```

### Upgrade and rebase

Below oneliner will remove your virtual disk and recreate. This will cause a reset of docker to factory defaults.

The command will create a disk, resize it, start a VM, upgrade all packages, stop the VM and merge the upgraded version back to the `root` image.

From this point any resets will go back to this stored stage as opposed to the one that was originally downloaded.

```
cd ~/.kubelab/bin
./qcow-del.sh ; ./qcow-make.sh && ./qcow-resize.sh 50 && ./start.sh && sleep 5 && ./upgrade.sh all && sleep 5 && ./stop.sh && sleep 5 && ./qcow-commit.sh
```
