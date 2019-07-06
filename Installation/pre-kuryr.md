# Pre-kuryr
### Run on Kubernetes nodes
- Install neutron-openvswitch-agent in all kubernetes nodes
```
add-apt-repository cloud-archive:stein -y
apt install neutron-openvswitch-agent -y
```
- Allow amqp and mariadb from all kubernetes nodes in Controller nodes
```
iptables -I INPUT 1 -s IP_KUBERNETES_NODES/32 -p tcp -m multiport --dports 5671,5672 -m comment --comment "001 amqp incoming amqp_IP_KUBERNETES_NODES" -j ACCEPT
iptables -I INPUT 1 -s IP_KUBERNETES_NODES/32 -p tcp -m multiport --dports 3306 -m comment --comment "001 mariadb incoming mariadb_IP_KUBERNETES_NODES" -j ACCEPT
```
- Allow neutron tunneling from all kubernetes nodes in network and compute nodes
```
iptables -I INPUT 1 -s IP_KUBERNETES_NODES/32 -p udp -m multiport --dports 4789 -m comment --comment "001 neutron tunnel port incoming neutron_tunnel_IP_KUBERNETES_NODES" -j ACCEPT
```
- In one of compute node, copy neutron configuration into all kubernetes nodes
```
for instance in kubernetes_node; do
	egrep -v ^'(#|$)' /etc/neutron/neutron.conf | ssh ${instance} tee /etc/neutron/neutron.conf
	egrep -v ^'(#|$)' /etc/neutron/plugins/ml2/openvswitch_agent.ini | ssh ${instance} tee /etc/neutron/plugins/ml2/openvswitch_agent.ini
	egrep -v ^'(#|$)' /etc/neutron/l3_agent.ini | ssh ${instance} tee /etc/neutron/l3_agent.ini
	egrep -v ^'(#|$)' /etc/neutron/dhcp_agent.ini | ssh ${instance} tee /etc/neutron/dhcp_agent.ini
	egrep -v ^'(#|$)' /etc/neutron/metadata_agent.ini | ssh ${instance} tee /etc/neutron/metadata_agent.ini
done
```
- Change `local-ip` neutron in all kubernetes nodes
```
nano /etc/neutron/plugins/ml2/openvswitch_agent.ini
```
- Start neutron agent
```
systemctl enable neutron-openvswitch-agent 
systemctl start neutron-openvswitch-agent
```

