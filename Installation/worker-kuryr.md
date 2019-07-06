# Kuryr worker
- Install package to build kuryr
```
sudo apt install gcc libffi-dev python-dev libssl-dev python-pip virtualenv crudini -y
```
- Build kuryr
```
mkdir kuryr-k8s-controller
cd kuryr-k8s-controller

git clone https://git.openstack.org/openstack/kuryr-kubernetes -b stable/stein
pip install -e kuryr-kubernetes
```
- Generate configuration
```
cd kuryr-kubernetes
./tools/generate_config_file_samples.sh

mkdir -p /etc/kuryr/
cp etc/kuryr.conf.sample /etc/kuryr/kuryr.conf
```
- Edit kuryr configuration
```
crudini --set /etc/kuryr/kuryr.conf DEFAULT use_stderr true
crudini --set /etc/kuryr/kuryr.conf DEFAULT lock_path /var/kuryr-lock
crudini --set /etc/kuryr/kuryr.conf DEFAULT bindir /usr/local/libexec/kuryr
crudini --set /etc/kuryr/kuryr.conf kubernetes api_root http://KUBERNETES-API:8080
```
- Create CNI
```
mkdir -p /opt/cni/bin
ln -s $(which kuryr-cni) /opt/cni/bin/
sudo mkdir -p /etc/cni/net.d/
cat << EOF >> /etc/cni/net.d/10-kuryr.conf
{
    "cniVersion": "0.3.0",
    "name": "kuryr",
    "type": "kuryr-cni",
    "kuryr_conf": "/etc/kuryr/kuryr.conf",
    "debug": true
}
EOF
```
- Install python requirement package
```
sudo pip install 'oslo.privsep>=1.20.0' 'os-vif>=1.5.0'
```
- Restart kubelet
```
sudo systemctl daemon-reload
systemctl restart kubelet.service
systemctl status kubelet.service
```
- Create kuryr-daemon service
```
cat << EOF >> /etc/systemd/system/kuryr-daemon.service
[Unit]
Description = kuryr-daemon.service

[Service]
Group = root
ExecReload = /bin/kill -HUP $MAINPID
TimeoutStopSec = 300
KillMode = process
ExecStart = /usr/local/bin/kuryr-daemon --config-file /etc/kuryr/kuryr.conf
User = root

[Install]
WantedBy = multi-user.target
EOF

sudo systemctl daemon-reload
systemctl enable kuryr-daemon.service
systemctl start kuryr-daemon.service
systemctl status kuryr-daemon.service
```
