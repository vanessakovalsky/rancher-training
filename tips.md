# Tips et debug courant

## Augmenter le nombre de cpu sur VM multipass:
```
multipass stop rke-master
multipass set local.rke-master.cpus=4
multipass set local.rke-master.memory=8G
multipass set local.rke-master.disk=50G
multipass start rke-master
multipass shell rke-master
sudo systemctl start rke2-server.service
```
