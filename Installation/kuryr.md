# kuryr Installation
### Run on Master node
- Change kubernetes api-server configuration
I using kubeadm to build kubernetes
```
/etc/kubernetes/manifests/kube-apiserver.yaml

    - --insecure-bind-address=0.0.0.0
    - --insecure-port=8080 
```
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
### Run on Controller node
- Create project, user, role, and environment files
```
openstack project create --description 'Kubernetes' k8s
openstack user create --project k8s --password k8spassword k8suser
openstack role add --user k8suser --project k8s admin
openstack role add --user k8suser --project services admin

cat << EOF >> k8src
unset OS_SERVICE_TOKEN
    export OS_USERNAME=k8suser
    export OS_PASSWORD='k8spassword'
    export OS_AUTH_URL=http://OPENSTACK_API:5000/v3
    export PS1='[\u@\h \W(kubernetes)]\$ '
    
export OS_PROJECT_NAME=k8s
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
EOF
```
- Create Internal network and External network if you havent create it before
```
source k8src

openstack network create internal
openstack subnet create --network internal --subnet-range 192.168.1.0/24 --gateway 192.168.1.1 --allocation-pool start=192.168.1.100,end=192.168.1.199 --dns-nameserver 8.8.8.8 internal

neutron net-create external --provider:network_type flat --provider:physical_network extnet --shared --router:external
neutron subnet-create external EXTERNAL_NET/24 --name external --gateway EXTERNAL_GATEWAY --disable-dhcp --allocation-pool start=START_IP,end=LAST_IP --dns-nameserver 8.8.8.8
```
- Create Pod Network
```
source k8src

openstack network create pod
openstack subnet create --network pod --no-dhcp \
    --gateway 10.1.255.254 \
    --subnet-range 10.1.0.0/16 \
    pod_subnet
```
- Create Service Network
```
source k8src

openstack network create services
openstack subnet create --network services --no-dhcp \
    --gateway 10.2.255.254 \
    --ip-version 4 \
    --allocation-pool start=10.2.128.1,end=10.2.255.253 \
    --subnet-range 10.2.0.0/16 \
    service_subnet
```
- Create router and add all network
```
source k8src

openstack router create kuryr-kubernetes
openstack router set --external-gateway external kuryr-kubernetes
openstack router add subnet kuryr-kubernetes internal

openstack port create --network pod --fixed-ip ip-address=10.1.255.254 pod_subnet_router
openstack port create --network services --fixed-ip ip-address=10.2.255.254 service_subnet_router

openstack router add port kuryr-kubernetes pod_subnet_router
openstack router add port kuryr-kubernetes service_subnet_router
```
- Create security group for kuryr
```
source k8src

openstack security group create service_pod_access_sg
openstack security group rule create --remote-ip 192.168.1.0/24 --ethertype IPv4 --protocol tcp service_pod_access_sg
openstack security group rule create --remote-ip 10.2.0.0/16 --ethertype IPv4 --protocol tcp service_pod_access_sg
openstack security group rule create --remote-ip 10.1.0.0/16 --ethertype IPv4 --protocol tcp service_pod_access_sg
```
- List component needed
```
source k8src 

openstack network list
openstack security group list
openstack subnet list
openstack project list
```
### Run on Master node
- Edit kuryr.conf based on data created on controller node
```
cd ~
crudini --set /etc/kuryr/kuryr.conf DEFAULT use_stderr true
crudini --set /etc/kuryr/kuryr.conf DEFAULT lock_path /var/kuryr-lock
crudini --set /etc/kuryr/kuryr.conf DEFAULT bindir /usr/local/libexec/kuryr
crudini --set /etc/kuryr/kuryr.conf kubernetes api_root http://KUBERNETES-API:8080
crudini --set /etc/kuryr/kuryr.conf neutron auth_url http://OPENSTACK-API:5000/v3
crudini --set /etc/kuryr/kuryr.conf neutron username k8suser
crudini --set /etc/kuryr/kuryr.conf neutron user_domain_name Default
crudini --set /etc/kuryr/kuryr.conf neutron password k8spassword
crudini --set /etc/kuryr/kuryr.conf neutron project_name k8s
crudini --set /etc/kuryr/kuryr.conf neutron project_domain_name Default
crudini --set /etc/kuryr/kuryr.conf neutron auth_type password
crudini --set /etc/kuryr/kuryr.conf neutron_defaults ovs_bridge br-int
crudini --set /etc/kuryr/kuryr.conf neutron_defaults external_svc_net ID_NETWORK_EXTERNAL
crudini --set /etc/kuryr/kuryr.conf neutron_defaults pod_security_groups ID_POD_SECURITY_GROUP
crudini --set /etc/kuryr/kuryr.conf neutron_defaults external_svc_subnet ID_SUBNET_NETWORK_EXTERNAL
crudini --set /etc/kuryr/kuryr.conf neutron_defaults service_subnet ID_SUBNET_NETWORK_SERVICE
crudini --set /etc/kuryr/kuryr.conf neutron_defaults pod_subnet ID_SUBNET_NETWORK_POD
crudini --set /etc/kuryr/kuryr.conf neutron_defaults project ID_KUBERNETES_PROJECT
```
- Downgrade openstacksdk (I have error on latest version)
```
pip install openstacksdk==0.31.0 --upgrade
```
- Create kuryr-kubernetes service
```
cat << EOF >> /etc/systemd/system/kuryr-kubernetes.service
[Unit]
Description = kuryr-kubernetes.service

[Service]
ExecReload = /bin/kill -HUP $MAINPID
TimeoutStopSec = 300
KillMode = process
ExecStart = /usr/local/bin/kuryr-k8s-controller --config-file /etc/kuryr/kuryr.conf
User = root

[Install]
WantedBy = multi-user.target
EOF

sudo systemctl daemon-reload
systemctl enable kuryr-kubernetes.service
systemctl start kuryr-kubernetes.service
systemctl status kuryr-kubernetes.service
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
