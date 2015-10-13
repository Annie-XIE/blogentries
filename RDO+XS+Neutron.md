### Manual of XenServer+RDO+Neutron

This manual gives brief instruction on installing OpenStack 
using RDO under RHEL7/CentOS7.

		Environment:
			XenServer: 6.5
			CentOS: 7.0
			OpenStack: Kilo
			Network: Neutron, ML2 plugin, OVS, VLAN

##### 1. Install XenServer 6.5
Make sure SR is EXT3 (in the installer this is called XenDesktop optimised storage).

##### 2. Install OpenStack VM
OpenStack VM is used for installing OpenStack software. One VM per hypervisor using 
XenServer 6.5 and RHEL7/CentOS7 templates. Please ensure they are HVM guests.

2.1 Create network and interface. Please upload [rdo_xenserver_helper.sh](https://github.com/Annie-XIE/summary-os/blob/master/rdo_xenserver_helper.sh) 
at both Dom0 and DomU.

		create_vif <vm_uuid>

##### 3. Install RDO
3.1 [RDO Quickstart](https://www.rdoproject.org/Quickstart) gives detailed 
installation guide, please follow the instruction step by step.

3.2 `Step 1: Software repositories`. 

*Note:* 

*(a) Remove the postfix `.orig` of `CentOS-XXX.repo.orig` 
in folder `/etc/yum.repos.d` and then try `yum update -y`.*

*(b) You may meet errors while executing yum update, you can ignore these 
errors, some are not needed in our environment.*

3.3 `Step 2: Install Packstack Installer` 

*Note: Packstack is the real one that installs OpenStack service. 
You may also meet package dependency errors during this step, 
you should fix these errors manually*

3.4 `Step 3: Run Packstack to install OpenStack`. Use 
`packstack --answer-file=<ANSWER_FILE>` instead of all-in-one.

`packstack --gen-answer-file=<ANSWER_FILE>` will generate an answer file, 
set neutron related configurations.

    CONFIG_DEFAULT_PASSWORD=<your-password>
    CONFIG_DEBUG_MODE=y
    CONFIG_NEUTRON_INSTALL=y
    CONFIG_NEUTRON_L3_EXT_BRIDGE=br-ex
    CONFIG_NEUTRON_ML2_TYPE_DRIVERS=vlan
    CONFIG_NEUTRON_ML2_TENANT_NETWORK_TYPES=vlan
    CONFIG_NEUTRON_ML2_MECHANISM_DRIVERS=openvswitch
    CONFIG_NEUTRON_ML2_VLAN_RANGES=<physnet1:1000:1050>
    CONFIG_NEUTRON_L2_AGENT=openvswitch
    CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=<physnet1:br-eth1>
    CONFIG_NEUTRON_OVS_BRIDGE_IFACES=<br-eth1:eth1>
    
*Note:*

*(a) Values within <> should be set according to your environment*

*(b) OpenStack is installed and is running at the moment.*

##### 4. Configure OpenStackVM/Hypervisor communications
4.1 Install XenServer PV tools in the OpenStack VM.

4.2 Set up DHCP on HIMN network for OpenStack VM so it can access its hypervisor 
on the static address 169.254.0.1. You can do this on XenCenter or 
do this in Dom0 and DomU with below command.

		Dom0:
		create_himn <vm_uuid>
		DomU:
		active_himn_interface

4.3 Copy Nova and Neutron plugins to XenServer host.

		install_dom0_plugins <dom0_ip>

##### 5. Configure Nova
5.1 Edit /etc/nova/nova.conf, switch compute driver to XenServer. 

    [DEFAULT]
    compute_driver=xenapi.XenAPIDriver
    firewall_driver=nova.virt.firewall.NoopFirewallDriver
    
    [xenserver]
    connection_url=http:169.254.0.1
    connection_username=root
    connection_password=<password>
    vif_driver=nova.virt.xenapi.vif.XenAPIOpenVswitchDriver
    ovs_int_bridge=<integration network bridge>

5.2 Install XenAPI Python XML RPC lightweight bindings

    yum install -y python-pip
    pip install xenapi
    
or
    
    curl https://raw.githubusercontent.com/xapi-project/xen-api/master/scripts/examples/python/XenAPI.py -o /usr/lib/python2.7/site-packages/XenAPI.py

##### 6. Configure Neutron
6.1 Edit confguration itmes in */etc/neutron/rootwrap.conf* to support
using XenServer remotely.

    [xenapi]
    # XenAPI configuration is only required by the L2 agent if it is to
    # target a XenServer/XCP compute host's dom0.
    xenapi_connection_url=http://169.254.0.1
    xenapi_connection_username=root
    xenapi_connection_password=<password>

6.2 Check network config file

This is corresponding to RDO's answer file, if ifcfg-eth1 not exist, create one

		CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=physnet1:br-eth1

		CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-eth1:eth1


##### 7. Launch another neutron-openvswitch-agent for talking with Dom0

For all-in-one installation, typically there should be only one neutron-openvswitch-agent.
Please refer [Deployment Model](https://github.com/Annie-XIE/summary-os/blob/master/deployment-neutron-1.png)

However, XenServer has a seperation of Dom0 and DomU and all instances' VIFs are actually 
managed by Dom0. Their corresponding OVS ports are created in Dom0. Thus, we should manually
start the other ovs agent which is in charge of these ports and is talking to Dom0, 
refer [xenserver_neutron picture](https://github.com/Annie-XIE/summary-os/blob/master/xs-neutron-deployment.png).


7.1 Create another configuration file

    cp /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini.dom0
    
    [ovs]
    integration_bridge = xapi3
    bridge_mappings = physnet1:xapi2
    
    [agent]
    root_helper = neutron-rootwrap-xen-dom0 /etc/neutron/rootwrap.conf
    root_helper_daemon =
    minimize_polling = False
    
    [securitygroup]
    firewall_driver = neutron.agent.firewall.NoopFirewallDriver

7.2 Launch neutron-openvswitch-agent

    /usr/bin/python2 /usr/bin/neutron-openvswitch-agent --config-file /usr/share/neutron/neutron-dist.conf --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini.dom0 --config-dir /etc/neutron/conf.d/neutron-openvswitch-agent --log-file /var/log/neutron/openvswitch-agent.log.dom0 &

##### 8. Restart Nova Services
    for svc in api cert conductor compute scheduler; do \
	    service openstack-nova-$svc restart; \
    done

##### 9. Replace cirros guest with one set up to work for XenServer
    nova image-delete cirros
    wget http://ca.downloads.xensource.com/OpenStack/cirros-0.3.4-x86_64-disk.vhd.tgz
    glance image-create --name cirros --container-format ovf --disk-format vhd --property vm_mode=xen --is-public True --file cirros-0.3.4-x86_64-disk.vhd.tgz

