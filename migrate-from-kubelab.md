# How to migrate from kubelab

1. Install `docker-laptop`
2. Ensure `kubelab` is stopped
3. Execute the following script to migrate the data volumes:

```
~/.kubelab/bin/qcow-data-commit.sh
rm -f ~/.kubelab/images/data/rw.qcow2
mv ~/.kubelab/images/data/ro.qcow2 ~/.docker-laptop/images/data/amd64/rw.qcow2
```

4. Execute the following script to remove kubelab:

```
rm -rf ~/.kubelab
sudo rm -rf /Applications/KubeLab-Launcher.app
sudo launchctl unload "/Library/LaunchDaemons/io.github.virtualsquare.vde-2.vde_switch.plist"
sudo launchctl unload "/Library/LaunchDaemons/io.github.lima-vm.vde_vmnet.plist"
sudo launchctl unload "/Library/LaunchDaemons/io.github.virtualsquare.vde-2.vde_switch.plist"
sudo rm -f "/Library/LaunchDaemons/io.github.virtualsquare.vde-2.vde_switch.plist"
sudo rm -f "/Library/LaunchDaemons/io.github.lima-vm.vde_vmnet.plist"
sudo rm -rf /opt/vde
```
