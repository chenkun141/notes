## vm配置网络 ##
### 安装虚拟机  ###
### 虚拟机安装linux ###
### 配置虚拟机网路 ###
NAT设置 IP地址需要和子网IP在同一网段

### 配置linux网络适配器 ###

- 1.配置  

	vi /etc/sysconfig/network-scripts/ifcfg-eth0 

	//编辑文件信息
	DEVICE=eth1
	TYPE=Ethernet
	ONBOOT=yes
	NM_CONTROLLED=yes
	IPADDR=192.168.80.11
	PREFIX=24
	NETMASK=255.255.255.0
	GATEWAY=192.168.80.1
	DNS1=114.114.114.114
	DNS2=8.8.8.8  

- 2.重新网络

	  service network restart

- 3. 主机和虚拟机相互ping地址,互相ping通,网络就可以了

      