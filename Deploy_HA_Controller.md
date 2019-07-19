# Hướng dẫn deploy High Availability cho Controller Node


## 1. Mục tiêu LAB
- Dựng hệ thống Openstack phiên bản Rocky và HA thành phần Controller Node
 
## 2. Mô hình
![img](image/HA_Rocky.jpg)

## 3. Chuẩn bị môi trường  
- OS: Centos 7.5 64bit
- Thêm hostname các máy vào /etc/hosts
- Đặt IP
- Đặt hostname
- Tắt Selinux, Firewall
- Đặt Timezone


## 4. Cài đặt
- Lưu ý :
  - Tất cả câu lệnh đều thực hiện với quyền `ROOT`  

### 1. Chuẩn bị môi trường cơ bản
- Bước 1 - All Controller Node + Computer Node
  * sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
  * sudo systemctl disable NetworkManager
  * sudo systemctl stop NetworkManager
  * sudo systemctl enable network
  * sudo systemctl start network
  * systemctl mask NetworkManager
  * yum remove postfix NetworkManager NetworkManager-libnm -y
  * yum -y install vim wget net-tools git chrony
  * sudo systemctl disable firewalld
  * sudo systemctl stop firewalld
  * timedatectl set-timezone Asia/Ho_Chi_Minh
  * systemctl enable chronyd.service
  * systemctl restart chronyd.service
  * reboot
- Bước 2 - All Controller Node: Tạo repo MariaDB
  * vi /etc/yum.repos.d/mariadb.repo
```
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2.9/centos7-amd64/
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

- Bước 3 - All Controller Node: Cài đặt các gói phần mềm như Mariadb, rabbitmq, pacemaker, haproxy, openstack
 * yum -y install mariadb-server mariadb-client galera rsync socat jemalloc centos-release-openstack-rocky
 * yum -y install centos-release-openstack-rocky
 * yum -y update
 * yum -y upgrade
 * yum install pacemaker corosync haproxy pcs fence-agents-all resource-agents psmisc policycoreutils-python -y
 * yum install rabbitmq-server memcached python-memcached wget net-tools openstack-utils chrony openstack-keystone httpd mod_wsgi openstack-glance openstack-nova-api openstack-nova-conductor   openstack-nova-console openstack-nova-novncproxy   openstack-nova-scheduler openstack-nova-placement-api openstack-neutron openstack-neutron-ml2   openstack-neutron-linuxbridge ebtables  openstack-dashboard python-openstackclient openstack-selinux openstack-cinder targetcli python-keystone -y
 * reboot
 
### 2. Cài đặt Galera MariaDB
- Bước 1: All Controller Node: Đặt mật khẩu
 * systemctl start mariadb
 * mysql_secure_installation
 * systemctl stop mariadb
- Bước 2: Controller 01: Điền thông số cluster vào file server.cnf.
 * vim /etc/my.cnf.d/server.cnf
```
[mysqld]
collation-server = utf8_general_ci
character-set-server = utf8
[galera]
wsrep_on = ON
binlog_format = ROW
default-storage-engine = innodb
innodb_autoinc_lock_mode = 2
innodb_locks_unsafe_for_binlog = 1
query_cache_size = 0
query_cache_type = 0
bind-address = 172.20.80.161
datadir = /var/lib/mysql
innodb_log_file_size = 100M
innodb_file_per_table =
innodb_flush_log_at_trx_commit = 2
wsrep_provider = /usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address = gcomm://172.20.80.161,172.20.80.162,172.20.80.163
wsrep_cluster_name = galera_clusterha
wsrep_node_address = 172.20.80.161
wsrep_node_name = controller01
wsrep_sst_method = rsync
max_connections = 10240
skip-name-resolve =
```
- Bước 3: Lặp lại với 2 Node còn lại. Lưu ý thay `bind-address` , `wsrep_node_address`, `wsrep_node_name` bằng thông tin của host
- Bước 4: Controller 01: Khởi động Cluster
 * galera_new_cluster
- Bước 5: 2 Node Controller còn lại
 * systemctl start mariadb
- Bước 6: Kiểm tra trạng thái cluster. Truy cập DB
 * SHOW GLOBAL STATUS LIKE 'wsrep_%';

### 3. Cài đặt Memcached
- Bước 1: All Controller Node: Start dịch vụ
 * sed -i 's/OPTIONS="-l 127.0.0.1,::1/OPTIONS="/g' /etc/sysconfig/memcached
 * systemctl enable memcached.service
 * systemctl restart memcached.service

### 4. Cài đặt Cluster RabbitMQ
- Bước 1: Controller 01: Sửa listen service
```
cat > /etc/rabbitmq/rabbitmq-env.conf << EOF
NODE_IP_ADDRESS=172.20.80.161
EOF
```
- Bước 2: Lặp lại với 2 Node còn lại. Lưu ý thay `NODE_IP_ADDRESS` bằng IP của host
- Bước 3: Controller 01: Tạo file cookie và chuyển file cookie sang 2 Node còn lại
 * systemctl restart rabbitmq-server
 * systemctl stop rabbitmq-server
 * scp -p /var/lib/rabbitmq/.erlang.cookie controller02:/var/lib/rabbitmq
 * scp -p /var/lib/rabbitmq/.erlang.cookie controller03:/var/lib/rabbitmq
- Bước 4: Controller02 & Controller03: Phân lại quyền cho file cookie
 * chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
 * chmod 400 /var/lib/rabbitmq/.erlang.cookie
- Bước 5: All Controller Node: Start dịch vụ
 * systemctl enable rabbitmq-server.service
 * systemctl restart rabbitmq-server.service
- Bước 6: Controller 01 
 * rabbitmqctl start_app
- Bước 7: Controller 02
 * rabbitmqctl stop_app
 * rabbitmqctl join_cluster rabbit@controller01
 * rabbitmqctl start_app
- Bước 8: Controller 03
 * rabbitmqctl stop_app
 * rabbitmqctl join_cluster rabbit@controller01
 * rabbitmqctl start_app
 * ```rabbitmqctl set_policy ha-all '^(?!amq\.).*' '{"ha-mode": "all"}'```
 * rabbitmqctl add_user openstack rabbitpwd
 * rabbitmqctl set_permissions openstack ".*" ".*" ".*"
- Bước 9: All Controller Node: Check trạng thái
 * rabbitmqctl cluster_status

### 5. Cài đặt Haproxy và Pacemaker
- Bước 1: All Controller Node: Cấu hình haproxy
 * vim /etc/haproxy/haproxy.cfg
 * File cấu hình [Tham khảo](https://github.com/hoangha1908/openstack-rocky/blob/master/haproxy.conf)
- Bước 2: All Controller Node
 *  echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf
 *  passwd hacluster
 *  systemctl enable pcsd.service pacemaker.service corosync.service haproxy.service
 *  systemctl start pcsd.service
- Bước 3: Controller 01: Thực hiện join 3 node vào cluster và cấu hình
 * pcs cluster auth controller01 controller02 controller03
 * pcs cluster setup --start --name ops-cluster controller01 controller02 controller03
 * pcs cluster enable --all
 * pcs status
 * pcs property set stonith-enabled=false
 * pcs property set default-resource-stickiness="INFINITY"
 * ```pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 ip=172.20.80.170 cidr_netmask=24 op monitor interval=30s```
 * ```pcs resource create HAproxy systemd:haproxy op monitor interval=5s```
 * pcs constraint colocation add HAproxy Virtual_IP INFINITY
 * pcs constraint order Virtual_IP then HAproxy
- Bước 4: Reboot lần lượt các Node 1, 2, 3. Lưu ý chờ start xong mới thực hiện tiếp

### 6. Cài đặt keystone
- Bước 1: Controller 01: Tạo database
 * mysql -u root -p
 * CREATE DATABASE keystone character set utf8;
 * GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystonedbpwd';
 * GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'controller' IDENTIFIED BY 'keystonedbpwd';
 * GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'controller01' IDENTIFIED BY 'keystonedbpwd';
 * GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'controller02' IDENTIFIED BY 'keystonedbpwd';
 * GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'controller03' IDENTIFIED BY 'keystonedbpwd';
 * GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystonedbpwd';
 * flush privileges;
- Bước 2: Controller 01: Cấu hình keystone và gen database, copy key sang 2 Node còn lại
 * crudini --set /etc/keystone/keystone.conf database connection mysql+pymysql://keystone:keystonedbpwd@controller/keystone
 * crudini --set /etc/keystone/keystone.conf database max_retries -1
 * crudini --set /etc/keystone/keystone.conf token provider fernet
 * scp /etc/keystone/keystone.conf controller02:/etc/keystone/
 * scp /etc/keystone/keystone.conf controller03:/etc/keystone/
 * su -s /bin/sh -c "keystone-manage db_sync" keystone
 * keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
 * keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
 * scp -r /etc/keystone/fernet-keys/ controller02:/etc/keystone/
 * scp -r /etc/keystone/credential-keys/ controller02:/etc/keystone/
 * scp -r /etc/keystone/fernet-keys/ controller03:/etc/keystone/
 * scp -r /etc/keystone/credential-keys/ controller03:/etc/keystone/
- Bước 3: Trên 2 Node còn lại: Phân lại quyền thư mục key
 * chown -R keystone:keystone /etc/keystone/credential-keys/
 * chown -R keystone:keystone /etc/keystone/fernet-keys/
- Bước 4: Controller 01: Tạo endpoint keystone

```
keystone-manage bootstrap --bootstrap-password Cloud2019 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

- Bước 3: All Controller Node: Sửa ServerName trong http.conf trùng với hostname
 * vim /etc/httpd/conf/httpd.conf
 * ServerName controller01

- Bước 4: All Controller Node: Link file keystone và sửa Listen IP. Lưu ý sửa IP trùng với host. Do cài 1 lần all package nên đổi luôn placement
 * ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
 * `sed -i -e 's/VirtualHost \*/VirtualHost 172.20.80.161/g' /etc/httpd/conf.d/wsgi-keystone.conf`
 * `sed -i -e 's/Listen 5000/Listen 172.20.80.161:5000/g' /etc/httpd/conf.d/wsgi-keystone.conf`
 * `sed -i -e 's/^Listen.*/Listen 172.20.80.161:80/g' /etc/httpd/conf/httpd.conf`
 * `sed -i -e 's/^Listen 8778/Listen 172.20.80.161:8778/g' /etc/httpd/conf.d/00-nova-placement-api.conf`
 * `sed -i -e 's/VirtualHost \*/VirtualHost 172.20.80.161/g' /etc/httpd/conf.d/00-nova-placement-api.conf`
 * systemctl enable httpd.service
 * systemctl restart httpd.service

- Bước 5: All Controller Node: Tạo file biến môi trường Admin
```
echo "export OS_PROJECT_DOMAIN_NAME=Default" >> admin-openrc
echo "export OS_USER_DOMAIN_NAME=Default" >> admin-openrc
echo "export OS_PROJECT_NAME=admin" >> admin-openrc
echo "export OS_USERNAME=admin" >> admin-openrc
echo "export OS_PASSWORD=Cloud2019" >> admin-openrc
echo "export OS_AUTH_URL=http://controller:5000/v3" >> admin-openrc
echo "export OS_IDENTITY_API_VERSION=3" >> admin-openrc
echo "export OS_IMAGE_API_VERSION=2" >> admin-openrc
```
- Bước 6: Controller 01: Tạo user, project test ...
 * openstack domain create --description "An Example Domain" example
 * openstack project create --domain default  --description "Service Project" service
 * openstack project create --domain default --description "Demo Project" myproject
 * openstack user create --domain default  --password myuser myuser
 * openstack role create myrole
 * openstack role add --project myproject --user myuser myrole
 
### 7. Cài đặt Glance
- Bước 1: Controller 01: Tạo database
 * mysql -u root -p
 * CREATE DATABASE glance character set utf8;
 * GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glancedbpwd';
 * GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'controller' IDENTIFIED BY 'glancedbpwd';
 * GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'controller01' IDENTIFIED BY 'glancedbpwd';
 * GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'controller02' IDENTIFIED BY 'glancedbpwd';
 * GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'controller03' IDENTIFIED BY 'glancedbpwd';
 * GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glancedbpwd';
 * flush privileges;
- Bước 2: Controller 01: Tạo user, project ...
 * openstack user create --domain default --password glancepwd glance
 * openstack role add --project service --user glance admin
 * openstack service create --name glance --description "OpenStack Image" image
 * openstack endpoint create --region RegionOne image public http://controller:9292
 * openstack endpoint create --region RegionOne image internal http://controller:9292
 * openstack endpoint create --region RegionOne image admin http://controller:9292
- Bước 3: Controller 01: Cấu hình glance
```
crudini --set /etc/glance/glance-api.conf DEFAULT bind_host 172.20.80.161
crudini --set /etc/glance/glance-api.conf DEFAULT registry_host 172.20.80.161
crudini --set /etc/glance/glance-api.conf database connection mysql+pymysql://glance:glancedbpwd@controller/glance
crudini --set /etc/glance/glance-api.conf database max_retries -1
crudini --set /etc/glance/glance-api.conf keystone_authtoken www_authenticate_uri  http://controller:5000
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_url  http://controller:5000
crudini --set /etc/glance/glance-api.conf keystone_authtoken memcached_servers  controller01:11211,controller02:11211,controller03:11211
crudini --set /etc/glance/glance-api.conf keystone_authtoken project_domain_name Default
crudini --set /etc/glance/glance-api.conf keystone_authtoken user_domain_name Default
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_type  password
crudini --set /etc/glance/glance-api.conf keystone_authtoken project_name  service
crudini --set /etc/glance/glance-api.conf keystone_authtoken username glance
crudini --set /etc/glance/glance-api.conf keystone_authtoken password glancepwd
crudini --set /etc/glance/glance-api.conf paste_deploy flavor keystone 
crudini --set /etc/glance/glance-api.conf glance_store stores file,http
crudini --set /etc/glance/glance-api.conf glance_store default_store file
crudini --set /etc/glance/glance-api.conf glance_store filesystem_store_datadir /var/lib/glance/images/
crudini --set /etc/glance/glance-registry.conf DEFAULT bind_host 172.20.80.161
crudini --set /etc/glance/glance-registry.conf DEFAULT registry_host 172.20.80.161
crudini --set /etc/glance/glance-registry.conf DEFAULT notification_driver noop
crudini --set /etc/glance/glance-registry.conf database connection mysql+pymysql://glance:glancedbpwd@controller/glance
crudini --set /etc/glance/glance-registry.conf database max_retries -1
crudini --set /etc/glance/glance-registry.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/glance/glance-registry.conf keystone_authtoken auth_url  http://controller:5000
crudini --set /etc/glance/glance-registry.conf keystone_authtoken memcached_servers controller01:11211,controller02:11211,controller03:11211
crudini --set /etc/glance/glance-registry.conf keystone_authtoken project_domain_name Default
crudini --set /etc/glance/glance-registry.conf keystone_authtoken user_domain_name Default
crudini --set /etc/glance/glance-registry.conf keystone_authtoken auth_type password
crudini --set /etc/glance/glance-registry.conf keystone_authtoken project_name service
crudini --set /etc/glance/glance-registry.conf keystone_authtoken username glance
crudini --set /etc/glance/glance-registry.conf keystone_authtoken password glancepwd
crudini --set /etc/glance/glance-registry.conf paste_deploy flavor keystone 
```
- Bước 4: 2 Node còn lại: Chạy như bước 3 nhưng chú ý `bind_host` bằng IP của Host
- Bước 5: Controller 01: Gen database
 * su -s /bin/bash glance -c "glance-manage db_sync"
- Bước 6: All Controller Node
 * systemctl enable openstack-glance-api.service  openstack-glance-registry.service
 * systemctl restart openstack-glance-api.service  openstack-glance-registry.service
- Bước 7: Controller 01: Tải image và tạo image cirros
 * wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
 * `openstack image create "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare  --public`
 * openstack image list


### 8. Cài đặt Nova

- Bước 1: Controller 01: Tạo database
 * mysql -u root -p
 * CREATE DATABASE nova character set utf8;
 * GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'controller' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'controller01' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'controller02' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'controller03' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'novadbpwd';

 * CREATE DATABASE nova_api character set utf8;
 * GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'controller' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'controller01' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'controller02' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'controller03' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'novadbpwd';

 * CREATE DATABASE nova_cell0 character set utf8;
 * GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'controller' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'controller01' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'controller02' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'controller03' IDENTIFIED BY 'novadbpwd';
 * GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'novadbpwd';

 * CREATE DATABASE placement character set utf8;
 * GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'placementdbpwd';
 * GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'controller' IDENTIFIED BY 'placementdbpwd';
 * GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'controller01' IDENTIFIED BY 'placementdbpwd';
 * GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'controller02' IDENTIFIED BY 'placementdbpwd';
 * GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'controller03' IDENTIFIED BY 'placementdbpwd';
 * GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'placementdbpwd';
 * flush privileges;
 
- Bước 2: Controller 01: Tạo user, project ...
 *  openstack user create --domain default --password novapwd nova
 *  openstack role add --project service --user nova admin
 *  openstack service create --name nova --description "OpenStack Compute" compute
 *  openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
 *  openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
 *  openstack endpoint create --region RegionOne  compute admin http://controller:8774/v2.1
 *  openstack user create --domain default --password placementpwd placement
 *  openstack role add --project service --user placement admin
 *  openstack service create --name placement --description "Placement API" placement
 *  openstack endpoint create --region RegionOne placement public http://controller:8778
 *  openstack endpoint create --region RegionOne placement internal http://controller:8778
 *  openstack endpoint create --region RegionOne placement admin http://controller:8778
 
- Bước 3: Controller 01: Cấu hình Nova
```
crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:rabbitpwd@controller01:5672,openstack:rabbitpwd@controller02:5672,openstack:rabbitpwd@controller03:5672
crudini --set /etc/nova/nova.conf DEFAULT my_ip 172.20.80.161
crudini --set /etc/nova/nova.conf DEFAULT use_neutron true
crudini --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
crudini --set /etc/nova/nova.conf DEFAULT osapi_compute_listen 172.20.80.161
crudini --set /etc/nova/nova.conf DEFAULT metadata_host 172.20.80.161
crudini --set /etc/nova/nova.conf DEFAULT metadata_listen 172.20.80.161
crudini --set /etc/nova/nova.conf DEFAULT metadata_listen_port 8775
crudini --set /etc/nova/nova.conf DEFAULT allow_resize_to_same_host True
crudini --set /etc/nova/nova.conf DEFAULT allow_migrate_to_same_host True
crudini --set /etc/nova/nova.conf DEFAULT block_device_allocate_retries 6000
crudini --set /etc/nova/nova.conf DEFAULT block_device_allocate_retries_interval 1
crudini --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_ha_queues true
crudini --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_retry_interval 1
crudini --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_retry_backoff 2
crudini --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_durable_queues true
crudini --set /etc/nova/nova.conf workarounds disable_libvirt_livesnapshot false
crudini --set /etc/nova/nova.conf api_database connection mysql+pymysql://nova:novadbpwd@controller/nova_api
crudini --set /etc/nova/nova.conf api_database  max_retries -1
crudini --set /etc/nova/nova.conf database connection mysql+pymysql://nova:novadbpwd@controller/nova
crudini --set /etc/nova/nova.conf database  max_retries -1
crudini --set /etc/nova/nova.conf placement_database connection mysql+pymysql://placement:placementdbpwd@controller/placement
crudini --set /etc/nova/nova.conf placement_database  max_retries -1
crudini --set /etc/nova/nova.conf api auth_strategy keystone
crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:5000/v3
crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers controller01:11211,controller02:11211,controller03:11211
crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name default
crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name default
crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
crudini --set /etc/nova/nova.conf keystone_authtoken username nova
crudini --set /etc/nova/nova.conf keystone_authtoken password novapwd
crudini --set /etc/nova/nova.conf vnc enabled true
crudini --set /etc/nova/nova.conf vnc server_listen '$my_ip'
crudini --set /etc/nova/nova.conf vnc server_proxyclient_address '$my_ip'
crudini --set /etc/nova/nova.conf vnc novncproxy_host '$my_ip'
crudini --set /etc/nova/nova.conf glance api_servers http://controller:9292
crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
crudini --set /etc/nova/nova.conf placement region_name RegionOne
crudini --set /etc/nova/nova.conf placement project_domain_name Default
crudini --set /etc/nova/nova.conf placement project_name service
crudini --set /etc/nova/nova.conf placement auth_type password
crudini --set /etc/nova/nova.conf placement user_domain_name Default
crudini --set /etc/nova/nova.conf placement auth_url http://controller:5000/v3
crudini --set /etc/nova/nova.conf placement username placement
crudini --set /etc/nova/nova.conf placement password placementpwd
```
```
sed -i '13 a <Directory /usr/bin>' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i '14 a <IfVersion >= 2.4>' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i '15 a Require all granted' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i '16 a </IfVersion>' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i '17 a <IfVersion < 2.4>' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i '18 a Order allow,deny' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i '19 a Allow from all' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i '20 a </IfVersion>' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i '21 a </Directory>' /etc/httpd/conf.d/00-nova-placement-api.conf
```
 * systemctl restart httpd
- Bước 4: 2 Node còn lại: Chạy như bước 3 nhưng chú ý `my_ip`, `osapi_compute_listen`, `metadata_host`, `metadata_listen` bằng IP của Host
- Bước 5: Controller 01: Restart http và gen database
 * systemctl restart httpd
 * su -s /bin/sh -c "nova-manage api_db sync" nova
 * su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
 * su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
 * su -s /bin/sh -c "nova-manage db sync" nova
 * su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
 
- Bước 6: All Node Controller: Start dịch vụ
 * systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service   openstack-nova-novncproxy.service
 * systemctl start openstack-nova-api.service   openstack-nova-consoleauth.service openstack-nova-scheduler.service   openstack-nova-conductor.service openstack-nova-novncproxy.service
  
- Bước 7: Computer Node: Cài đặt gói phần mềm
 * yum -y install centos-release-openstack-rocky
 * yum update -y
 * yum upgrade -y
 * yum install openstack-utils openstack-nova-compute -y
 
- Bước 8: Computer Node: Cài đặt nova
```
crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:rabbitpwd@controller01:5672,openstack:rabbitpwd@controller02:5672,openstack:rabbitpwd@controller03:5672
crudini --set /etc/nova/nova.conf DEFAULT my_ip 172.20.80.164
crudini --set /etc/nova/nova.conf DEFAULT use_neutron True
crudini --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
crudini --set /etc/nova/nova.conf DEFAULT block_device_allocate_retries 6000
crudini --set /etc/nova/nova.conf DEFAULT block_device_allocate_retries_interval 1
crudini --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_ha_queues true
crudini --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_retry_interval 1
crudini --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_retry_backoff 2
crudini --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_durable_queues true
crudini --set /etc/nova/nova.conf workarounds disable_libvirt_livesnapshot false
crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:5000/v3
crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers controller01:11211,controller02:11211,controller03:11211
crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name default
crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name default
crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
crudini --set /etc/nova/nova.conf keystone_authtoken username nova
crudini --set /etc/nova/nova.conf keystone_authtoken password novapwd
crudini --set /etc/nova/nova.conf api auth_strategy keystone
crudini --set /etc/nova/nova.conf vnc enabled true
crudini --set /etc/nova/nova.conf vnc server_listen 0.0.0.0
crudini --set /etc/nova/nova.conf vnc server_proxyclient_address \$my_ip
crudini --set /etc/nova/nova.conf vnc novncproxy_base_url http://172.20.80.170:6080/vnc_auto.html
crudini --set /etc/nova/nova.conf glance api_servers http://controller:9292
crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
crudini --set /etc/nova/nova.conf placement region_name RegionOne
crudini --set /etc/nova/nova.conf placement project_domain_name Default
crudini --set /etc/nova/nova.conf placement project_name service
crudini --set /etc/nova/nova.conf placement auth_type password
crudini --set /etc/nova/nova.conf placement user_domain_name Default
crudini --set /etc/nova/nova.conf placement auth_url http://controller:5000/v3
crudini --set /etc/nova/nova.conf placement username placement
crudini --set /etc/nova/nova.conf placement password placementpwd
vtx=`egrep -c '(vmx|svm)' /proc/cpuinfo`
if [ $vtx == 0 ]; then
crudini --set /etc/nova/nova.conf libvirt virt_type qemu
crudini --set /etc/nova/nova.conf libvirt cpu_mode none
else
crudini --set /etc/nova/nova.conf libvirt virt_type kvm
crudini --set /etc/nova/nova.conf libvirt cpu_mode none
fi
```
- Bước 9: Computer Node:
 * systemctl enable libvirtd.service openstack-nova-compute.service
 * systemctl start libvirtd.service openstack-nova-compute.service

- Bước 10: Controller 01: Thực hiện Add Compute host
 * openstack compute service list --service nova-compute
 * su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
 * openstack hypervisor list
 * nova-status upgrade check

### 9. Cài đặt Neutron
- Bước 1: Controller 01: Tạo database
 * mysql -u root -p
 * CREATE DATABASE neutron character set utf8;
 * GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'neutrondbpwd';
 * GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'controller' IDENTIFIED BY 'neutrondbpwd';
 * GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'controller01' IDENTIFIED BY 'neutrondbpwd';
 * GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'controller02' IDENTIFIED BY 'neutrondbpwd';
 * GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'controller03' IDENTIFIED BY 'neutrondbpwd';
 * GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutrondbpwd';
- Bước 2: Controller 01: Tạo user, project
 * openstack user create --domain default --password neutronpwd neutron
 * openstack role add --project service --user neutron admin
 * openstack service create --name neutron  --description "OpenStack Networking" network
 * openstack endpoint create --region RegionOne  network public http://controller:9696
 * openstack endpoint create --region RegionOne network internal http://controller:9696
 * openstack endpoint create --region RegionOne  network admin http://controller:9696
- Bước 3: Controller 01: Cài đặt Neutron
```
crudini --set /etc/neutron/neutron.conf database connection mysql+pymysql://neutron:neutrondbpwd@controller/neutron
crudini --set /etc/neutron/neutron.conf database  max_retries -1
crudini --set /etc/neutron/neutron.conf DEFAULT bind_host 172.20.80.161
crudini --set /etc/neutron/neutron.conf DEFAULT verbose true
crudini --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins router
crudini --set /etc/neutron/neutron.conf DEFAULT allow_overlapping_ips true
crudini --set /etc/neutron/neutron.conf DEFAULT router_distributed False
crudini --set /etc/neutron/neutron.conf DEFAULT l3_ha true
crudini --set /etc/neutron/neutron.conf DEFAULT l3_ha_net_cidr 169.254.192.0/18
crudini --set /etc/neutron/neutron.conf DEFAULT max_l3_agents_per_router 3
crudini --set /etc/neutron/neutron.conf DEFAULT min_l3_agents_per_router 2
crudini --set /etc/neutron/neutron.conf DEFAULT dhcp_agents_per_network 3
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:rabbitpwd@controller01:5672,openstack:rabbitpwd@controller02:5672,openstack:rabbitpwd@controller03:5672
crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes true
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes true
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_ha_queues true
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_retry_interval 1
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_retry_backoff 2
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_durable_queues true
crudini --set /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers controller01:11211,controller02:11211,controller03:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password neutronpwd
crudini --set /etc/neutron/neutron.conf nova auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf nova auth_type password
crudini --set /etc/neutron/neutron.conf nova project_domain_name default
crudini --set /etc/neutron/neutron.conf nova user_domain_name default
crudini --set /etc/neutron/neutron.conf nova region_name RegionOne
crudini --set /etc/neutron/neutron.conf nova project_name service
crudini --set /etc/neutron/neutron.conf nova username nova
crudini --set /etc/neutron/neutron.conf nova password novapwd
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers flat,vlan,vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers linuxbridge,l2population
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers port_security
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks provider
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1:1000
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings provider:ens256
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip 172.20.80.161
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan l2_population False
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini agent prevent_arp_spoofing true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
crudini --set /etc/neutron/l3_agent.ini DEFAULT verbose true
crudini --set /etc/neutron/l3_agent.ini DEFAULT interface_driver linuxbridge
crudini --set /etc/neutron/l3_agent.ini DEFAULT use_namespaces true
crudini --set /etc/neutron/l3_agent.ini DEFAULT external_network_bridge
crudini --set /etc/neutron/l3_agent.ini DEFAULT router_delete_namespaces true
crudini --set /etc/neutron/l3_agent.ini DEFAULT agent_mode legacy
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT verbose true
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver linuxbridge
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT use_namespaces true
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_delete_namespaces true
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata true
crudini --set /etc/neutron/metadata_agent.ini DEFAULT verbose true
crudini --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host controller
crudini --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret METADATA_SECRET
crudini --set /etc/nova/nova.conf neutron url http://controller:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://controller:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name default
crudini --set /etc/nova/nova.conf neutron user_domain_name default
crudini --set /etc/nova/nova.conf neutron region_name RegionOne
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password neutronpwd
crudini --set /etc/nova/nova.conf neutron service_metadata_proxy true
crudini --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret METADATA_SECRET
```
- Bước 4: Controller 01: Chỉnh sửa OS và đảm bảo `sysctl -a | grep net.bridge.bridge-nf-call-ip` = 1. Tạo soft link file
 * modprobe br_netfilter
 * ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
 
- Bước 5: Controller 01: Gen database
 * su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

- Bước 6: Trên 2 Node còn lại. Thực hiện bước 3, bước 4. Chú ý thay `bind_host` và `local_ip` bằng IP của Host.
- Bước 7: All Controller Node: Restart và khởi động dịch vụ
 * systemctl restart openstack-nova-api.service
 * systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service   neutron-metadata-agent.service neutron-l3-agent.service
 * systemctl start neutron-server.service  neutron-linuxbridge-agent.service neutron-dhcp-agent.service   neutron-metadata-agent.service neutron-l3-agent.service
 
- Bước 8: Computer Node: Cài đặt Neutron
 * yum install openstack-neutron-linuxbridge ebtables ipset -y
 ```
 crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:rabbitpwd@controller01:5672,openstack:rabbitpwd@controller02:5672,openstack:rabbitpwd@controller03:5672
crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_ha_queues true
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_retry_interval 1
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_retry_backoff 2
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_durable_queues true
crudini --set /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers controller01:11211,controller02:11211,controller03:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password neutronpwd
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings provider:em1
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip 172.20.80.164
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan l2_population False
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini agent prevent_arp_spoofing true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
crudini --set /etc/nova/nova.conf neutron url http://controller:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://controller:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name default
crudini --set /etc/nova/nova.conf neutron user_domain_name default
crudini --set /etc/nova/nova.conf neutron region_name RegionOne
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password neutronpwd
  ```
- Bước 9: Compute Node: Start dịch vụ
 * modprobe br_netfilter
 * echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
 * echo '1' > /proc/sys/net/bridge/bridge-nf-call-ip6tables
 * sysctl -p
 * systemctl restart openstack-nova-compute.service
 * systemctl enable neutron-linuxbridge-agent.service
 * systemctl start neutron-linuxbridge-agent.service

- Bước 10: Controller 01: Verify dịch vụ
 * openstack extension list --network
 * openstack network agent list


### 10. Cài đặt Horizon
- Bước 1: Controller 01: Sửa file `/etc/openstack-dashboard/local_settings`
 * vim /etc/openstack-dashboard/local_settings
```  
OPENSTACK_HOST = "172.20.80.161"
ALLOWED_HOSTS = ['*',]
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': ['controller01:11211', 'controller01:11211', 'controller03:11211']
    }
}
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
TIME_ZONE = "Asia/Ho_Chi_Minh"
```
- Bước 2: Controller 01: Sửa file ` /etc/httpd/conf.d/openstack-dashboard.conf` thêm vào ngay đầu `WSGIApplicationGroup %{GLOBAL}`
 * vim  /etc/httpd/conf.d/openstack-dashboard.conf
 * WSGIApplicationGroup %{GLOBAL}
- Bước 3: 2 Node còn lại: Thực hiện lại bước 1, bước 2. Lưu ý `OPENSTACK_HOST` = địa chỉ của Host  
- Bước 4: All Controller Node: Restart dịch vụ
 * systemctl restart httpd.service memcached.service
  
### 11. Cài đặt Cinder
- Lưu ý: Phần này sẽ cài đặt Cinder api, Cinder scheduler, Cinder Volume ngay trên cả 3 Node Controller
- Backend sẽ sử dụng hệ thống Ceph
- Cấp cho tất cả các Node Controller + Computer 1 dải mạng thông tới Ceph
- Theo recommend của hãng thì Cinder sẽ chạy Active/Passive sử dụng Pacemaker

- Bước 1: Controller 01: Tạo database
 * mysql -u root -p
 * CREATE DATABASE cinder character set utf8;
 * GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'cinderdbpwd';
 * GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'controller' IDENTIFIED BY 'cinderdbpwd';
 * GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'controller01' IDENTIFIED BY 'cinderdbpwd';
 * GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'controller02' IDENTIFIED BY 'cinderdbpwd';
 * GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'controller03' IDENTIFIED BY 'cinderdbpwd';
 * GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'cinderdbpwd';
- Bước 2: Controller 01: Tạo user, project
 * openstack user create --domain default --password cinderpwd cinder
 * openstack role add --project service --user cinder admin
 * openstack service create --name cinderv2  --description "OpenStack Block Storage" volumev2
 * openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
```
openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
```
- Bước 3: Ceph Node: Một số bước tham khảo link dưới
[Tham khảo](https://github.com/hoangha1908/Openstack/blob/master/Cinder_with_Ceph.md)

- Bước 4: Controller 01: Cấu hình Cinder
```
crudini --set /etc/cinder/cinder.conf database connection mysql+pymysql://cinder:cinderdbpwd@controller/cinder
crudini --set /etc/cinder/cinder.conf database max_retries -1
crudini --set /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:rabbitpwd@controller01:5672,openstack:rabbitpwd@controller02:5672,openstack:rabbitpwd@controller03:5672
crudini --set /etc/cinder/cinder.conf DEFAULT auth_strategy keystone
crudini --set /etc/cinder/cinder.conf DEFAULT enabled_backends rbd
crudini --set /etc/cinder/cinder.conf DEFAULT my_ip 172.20.80.161
crudini --set /etc/cinder/cinder.conf DEFAULT control_exchange cinder
crudini --set /etc/cinder/cinder.conf DEFAULT host controller
crudini --set /etc/cinder/cinder.conf DEFAULT enable_v3_api true
crudini --set /etc/cinder/cinder.conf DEFAULT glance_api_servers http://controller:9292
crudini --set /etc/cinder/cinder.conf DEFAULT glance_api_version 2
crudini --set /etc/cinder/cinder.conf DEFAULT osapi_volume_listen controller
crudini --set /etc/cinder/cinder.conf DEFAULT osapi_volume_listen_port 8776
crudini --set /etc/cinder/cinder.conf DEFAULT debug true
crudini --set /etc/cinder/cinder.conf oslo_messaging_rabbit rabbit_ha_queues true
crudini --set /etc/cinder/cinder.conf oslo_messaging_rabbit rabbit_retry_interval 1
crudini --set /etc/cinder/cinder.conf oslo_messaging_rabbit rabbit_retry_backoff 2
crudini --set /etc/cinder/cinder.conf oslo_messaging_rabbit rabbit_durable_queues true
crudini --set /etc/cinder/cinder.conf oslo_concurrency lock_path /var/lib/cinder/tmp
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken memcached_servers controller01:11211,controller02:11211,controller03:11211
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_type password
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_domain_name default
crudini --set /etc/cinder/cinder.conf keystone_authtoken user_domain_name default
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_name service
crudini --set /etc/cinder/cinder.conf keystone_authtoken username cinder
crudini --set /etc/cinder/cinder.conf keystone_authtoken password cinderpwd
crudini --set /etc/cinder/cinder.conf rbd volume_driver cinder.volume.drivers.rbd.RBDDriver
crudini --set /etc/cinder/cinder.conf rbd rbd_pool volumes
crudini --set /etc/cinder/cinder.conf rbd rbd_ceph_conf /etc/ceph/ceph.conf
crudini --set /etc/cinder/cinder.conf rbd rbd_flatten_volume_from_snapshot false
crudini --set /etc/cinder/cinder.conf rbd rbd_max_clone_depth 5
crudini --set /etc/cinder/cinder.conf rbd rbd_store_chunk_size 4
crudini --set /etc/cinder/cinder.conf rbd rados_connect_timeout -1
crudini --set /etc/cinder/cinder.conf rbd glance_api_version 2
crudini --set /etc/cinder/cinder.conf rbd rbd_user volumes
crudini --set /etc/cinder/cinder.conf rbd rbd_secret_uuid 6b36ee8b-a719-4e9f-8c12-ed1d3eaecbab
crudini --set /etc/nova/nova.conf cinder os_region_name RegionOne
```
- Bước 5: 2 Controller còn lại. Thực hiện lại bước 4. Lưu ý thay `my_ip` = địa chỉ của Host
- Bước 6: Controller 01: Gen database
 * su -s /bin/sh -c "cinder-manage db sync" cinder
- Bước 7: Controller 01: Cấu hình Pacemaker
```
pcs resource create openstack-cinder-api systemd:openstack-cinder-api --clone interleave=true
pcs resource create openstack-cinder-scheduler systemd:openstack-cinder-scheduler --clone interleave=true
pcs resource create openstack-cinder-volume systemd:openstack-cinder-volume
pcs constraint order start Virtual_IP then openstack-cinder-api-clone
pcs constraint colocation add openstack-cinder-api-clone with Virtual_IP
pcs constraint order start openstack-cinder-api-clone then openstack-cinder-scheduler-clone
pcs constraint colocation add openstack-cinder-scheduler-clone with openstack-cinder-api-clone
pcs constraint order start openstack-cinder-scheduler-clone then openstack-cinder-volume
pcs constraint colocation add openstack-cinder-volume with openstack-cinder-scheduler-clone
```
- Bước 8: ALl Controller Node:
 * systemctl restart openstack-nova-api.service
 * systemctl enable target.service
 * systemctl restart target.service
