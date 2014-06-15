---
layout: post
title: "Two Node OpenStack Neutron Configurations with GRE and Open
vSwitch"
date: 2014-01-30 14:44:48 +0530
comments: true
categories:
- OpenStack
- Open vSwitch
---

Set-up info
-----------

These configurations are from an OpenStack RDO Havana test setup on
Fedora 20.

- Neutron configuration files (and iptables rules) for a two node setup:

  - Controller node: Nova, Keystone, Cinder, Glance, Neutron (using Open
    vSwitch plugin and GRE tunneling).
  - Compute node: Nova (nova-compute), Neutron (openvswitch-agent)

Controller node Neutron configurations
--------------------------------------

### 1. neutron.conf

    $ cat /etc/neutron/neutron.conf | grep -v ^$ | grep -v ^#
    [DEFAULT]
    core_plugin =neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2
    rpc_backend = neutron.openstack.common.rpc.impl_qpid
    control_exchange = neutron
    qpid_hostname = 192.169.142.49
    auth_strategy = keystone
    allow_overlapping_ips = True
    dhcp_lease_duration = 120
    allow_bulk = True
    qpid_port = 5672
    qpid_heartbeat = 60
    qpid_protocol = tcp
    qpid_tcp_nodelay = True
    qpid_reconnect_limit=0
    qpid_reconnect_interval_max=0
    qpid_reconnect_timeout=0
    qpid_reconnect=True
    qpid_reconnect_interval_min=0
    qpid_reconnect_interval=0
    debug = False
    verbose = False
    [quotas]
    [agent]
    [keystone_authtoken]
    admin_tenant_name = services
    admin_user = neutron
    admin_password = fedora
    auth_host = 192.169.142.49
    auth_port = 35357
    auth_protocol = http
    auth_uri=http://192.169.142.49:5000/
    [database]
    [service_providers]
    [AGENT]
    root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf

### 2. (OVS) plugin.ini

    $ cat /etc/neutron/plugin.ini | grep -v ^$ | grep -v ^#
    [ovs]
    tenant_network_type = gre
    tunnel_id_ranges = 1:1000
    enable_tunneling = True
    integration_bridge = br-int
    tunnel_bridge = br-tun
    local_ip = 192.169.142.49
    [agent]
    [securitygroup]
    [DATABASE]
    sql_connection = mysql://neutron:fedora@node1-controller/ovs_neutron
    sql_max_retries=10
    reconnect_interval=2
    sql_idle_timeout=3600
    [SECURITYGROUP]
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

### 3. dhcp_agent.ini

    $ cat /etc/neutron/dhcp_agent.ini | grep -v ^$ | grep -v ^#
    [DEFAULT]
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    handle_internal_only_routers = TRUE
    external_network_bridge = br-ex
    use_namespaces = True
    dnsmasq_config_file = /etc/neutron/dnsmasq.conf

### 4. l3_agent.ini

    $ cat /etc/neutron/dhcp_agent.ini | grep -v ^$ | grep -v ^#
    [DEFAULT]
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    handle_internal_only_routers = TRUE
    external_network_bridge = br-ex
    use_namespaces = True
    dnsmasq_config_file = /etc/neutron/dnsmasq.conf

### 5. dnsmasq.conf

This logs dnsmasq output to a file, instead of journalctl):

    $ cat /etc/neutron/dnsmasq.conf | grep -v ^$ | grep -v ^#
    log-facility = /var/log/neutron/dnsmasq.log
    log-dhcp

### 6. api-paste.ini

    $ cat /etc/neutron/api-paste.ini | grep -v ^$ | grep -v ^#
    [composite:neutron]
    use = egg:Paste#urlmap
    /: neutronversions
    /v2.0: neutronapi_v2_0
    [composite:neutronapi_v2_0]
    use = call:neutron.auth:pipeline_factory
    noauth = extensions neutronapiapp_v2_0
    keystone = authtoken keystonecontext extensions neutronapiapp_v2_0
    [filter:keystonecontext]
    paste.filter_factory = neutron.auth:NeutronKeystoneContext.factory
    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
    admin_user=neutron
    auth_port=35357
    admin_password=fedora
    auth_protocol=http
    auth_uri=http://192.169.142.49:5000/
    admin_tenant_name=services
    auth_host = 192.169.142.49
    [filter:extensions]
    paste.filter_factory = neutron.api.extensions:plugin_aware_extension_middleware_factory
    [app:neutronversions]
    paste.app_factory = neutron.api.versions:Versions.factory
    [app:neutronapiapp_v2_0]
    paste.app_factory = neutron.api.v2.router:APIRouter.factory

### 7. metadata_agent.ini

    $ cat /etc/neutron/metadata_agent.ini | grep -v ^$ | grep -v ^#
    [DEFAULT]
    auth_url = http://192.169.142.49:35357/v2.0/
    auth_region = regionOne
    admin_tenant_name = services
    admin_user = neutron
    admin_password = fedora
    nova_metadata_ip = 192.168.142.49
    nova_metadata_port = 8775
    metadata_proxy_shared_secret = fedora


Compute node Neutron configurations
-----------------------------------

### 1. neutron.conf

    $ cat /etc/neutron/neutron.conf | grep -v ^$ | grep -v ^#
    [DEFAULT]
    core_plugin =neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2
    rpc_backend = neutron.openstack.common.rpc.impl_qpid
    qpid_hostname = 192.169.142.49
    auth_strategy = keystone
    allow_overlapping_ips = True
    qpid_port = 5672
    debug = True
    verbose = True
    [quotas]
    [agent]
    [keystone_authtoken]
    admin_tenant_name = services
    admin_user = neutron
    admin_password = fedora
    auth_host = 192.169.142.49
    [database]
    [service_providers]
    [AGENT]
    root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf

### 2. (OVS) plugin.ini

    $ cat plugin.ini | grep -v ^$ | grep -v ^#
    [ovs]
    tenant_network_type = gre
    tunnel_id_ranges = 1:1000
    enable_tunneling = True
    integration_bridge = br-int
    tunnel_bridge = br-tun
    local_ip = 192.169.142.57
    [DATABASE]
    sql_connection = mysql://neutron:fedora@node1-controller/ovs_neutron
    [SECURITYGROUP]
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    [agent]
    [securitygroup]

### 3. metadata_agent.ini

    $ cat metadata_agent.ini | grep -v ^$ | grep -v ^#
    [DEFAULT]
    auth_url = http://localhost:5000/v2.0
    auth_region = RegionOne
    admin_tenant_name = %SERVICE_TENANT_NAME%
    admin_user = %SERVICE_USER%
    admin_password = %SERVICE_PASSWORD%


iptables on Controller node
---------------------------

    $ cat /etc/sysconfig/iptables
    *filter
    :INPUT ACCEPT [0:0]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    -A INPUT -p icmp -j ACCEPT
    -A INPUT -i lo -j ACCEPT
    -A INPUT -p tcp -m multiport --dports 3260 -m comment --comment "001 cinder incoming" -j ACCEPT 
    -A INPUT -p tcp -m multiport --dports 80 -m comment --comment "001 horizon incoming" -j ACCEPT 
    -A INPUT -p tcp -m multiport --dports 9292 -m comment --comment "001 glance incoming" -j ACCEPT 
    -A INPUT -p tcp -m multiport --dports 5000,35357 -m comment --comment "001 keystone incoming" -j ACCEPT 
    -A INPUT -p tcp -m multiport --dports 3306 -m comment --comment "001 mariadb incoming" -j ACCEPT 
    -A INPUT -p tcp -m multiport --dports 6080 -m comment --comment "001 novncproxy incoming" -j ACCEPT 
    -A INPUT -p tcp -m multiport --dports 8770:8780 -m comment --comment "001 novaapi incoming" -j ACCEPT 
    -A INPUT -p tcp -m multiport --dports 9696 -m comment --comment "001 neutron incoming" -j ACCEPT 
    -A INPUT -p tcp -m multiport --dports 5672 -m comment --comment "001 qpid incoming" -j ACCEPT 
    -A INPUT -p tcp -m multiport --dports 8700 -m comment --comment "001 metadata incoming" -j ACCEPT 
    -A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
    -A INPUT -m state --state NEW -m tcp -p tcp --dport 5900:5999 -j ACCEPT
    -A INPUT -j REJECT --reject-with icmp-host-prohibited
    -A INPUT -p gre -j ACCEPT 
    -A OUTPUT -p gre -j ACCEPT
    -A FORWARD -j REJECT --reject-with icmp-host-prohibited
    COMMIT

iptables on Compute node
------------------------

    $ cat /etc/sysconfig/iptables
    *filter
    :INPUT ACCEPT [0:0]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    -A INPUT -p icmp -j ACCEPT
    -A INPUT -i lo -j ACCEPT
    -A INPUT -m state --state NEW -m tcp -p tcp --dport 5900:5999 -j ACCEPT
    -A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
    -A INPUT -p gre -j ACCEPT 
    -A INPUT -j REJECT --reject-with icmp-host-prohibited
    -A OUTPUT -p gre -j ACCEPT
    -A FORWARD -j REJECT --reject-with icmp-host-prohibited
    COMMIT

