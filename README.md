# MariaDB HA Cluster with MaxScale

![MariaDB](https://img.shields.io/badge/MariaDB-10.5.22-blue)

## Overview
This repository contains a detailed guide on the architecture, setup, and management of a high-availability (HA) MariaDB cluster consisting of three database nodes and two MaxScale nodes for load balancing and auto-failover.

## Architecture
### Components
- **MariaDB Cluster:** Three nodes using GTID-based replication. One is the master and the rests being the slaves.
- **MaxScale Nodes:** Two instances of MaxScale for high availability and load balancing.
- **Clients:** Applications connecting to the database via MaxScale.

### High Availability Mechanism
- GTID-based replication ensures data consistency.
- MaxScale provides automatic failover and query load balancing.
- Keepalived for virtual IP (VIP) management between MaxScale nodes.

## Prerequisites
| Component  | Version   |
|------------|-----------|
| MariaDB    | 10.5.22   |
| MaxScale   | 24.02.4   |
| Keepalived | v2.2.8    |

- Network connectivity between all nodes.
- Root or sudo access to configure services.
- Open specific ports like 3306 for MariaDB and 8989 for maxscale to be used with the GUI.

## MariaDB Cluster Setup
### Install MariaDB on Each Node(I am using Rocky Linux v9.4)
```sh
sudo dnf install -y mariadb-server
```
### Configure GTID-Based Replication
Modify the MariaDB configuration file (/etc/my.cnf.d/mariadb-server.cnf) on each node:
```sh
#
# These groups are read by MariaDB server.
# Use it for options that only the server (but not clients) should see
#
# See the examples of server my.cnf files in /usr/share/mysql/
#

# this is read by the standalone daemon and embedded servers
[server]

# this is only for the mysqld standalone daemon
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mysqld/mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mariadb/mariadb.log
pid-file=/run/mariadb/mariadb.pid
binlog_format=ROW
gtid_strict_mode=ON
log_slave_updates=ON
log_bin=mariadb-bin
log_bin_index=mariadb-bin.index
relay_log=relay-bin
relay_log_index=relay-bin.index
relay_log_info_file=relay-bin.info
server_id=1
gtid_domain_id=1
#shutdown_wait_for_slaves=ON
expire_logs_days=7
session_track_system_variables=last_gtid

#
# * Galera-related settings
#
[galera]
# Mandatory settings
#wsrep_on=ON
#wsrep_provider=
#wsrep_cluster_address=
#binlog_format=row
#default_storage_engine=InnoDB
#innodb_autoinc_lock_mode=2
#
# Allow server to accept connections on all interfaces.
#
bind-address=0.0.0.0
#
# Optional setting
#wsrep_slave_threads=1
#innodb_flush_log_at_trx_commit=0

# this is only for embedded server
[embedded]

# This group is only read by MariaDB servers, not by MySQL.
# If you use the same .cnf file for MySQL and MariaDB,
# you can put MariaDB-only options here
[mariadb]

# This group is only read by MariaDB-10.5 servers.
# If you use the same .cnf file for MariaDB of different versions,
# use this group for options that older servers don't understand
[mariadb-10.5]
```

### Set Up Replication

On the primary node, create a replication user:
```sh
CREATE USER 'replication_user'@'%' IDENTIFIED BY 'Pass@123';
GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
FLUSH PRIVILEGES;
```
On each replica node, configure replication:
```sh
CHANGE MASTER TO MASTER_HOST='192.168.xxx.xxx',
MASTER_USER='replication_user', MASTER_PASSWORD='Pass@123', MASTER_PORT=3306, 
MASTER_USE_GTID=slave_pos, MASTER_CONNECT_RETRY=10;
```
Start the slave:
```sh
START SLAVE;
```
Verify replication status:
```sh
SHOW SLAVE STATUS\G
```
## MaxScale Setup

### Install MaxScale
Download the repository of maxscale and install it:
```sh
sudo dnf install -y maxscale
```
Create maxscale admin user for monitoring the cluster in the MariaDB master server(It will eventually be replicated across the cluster):
```sh
CREATE USER 'maxadmin'@'%' IDENTIFIED BY 'Admin@123';
GRANT ALL ON *.* TO 'maxadmin'@'%';
FLUSH PRIVILEGES; 
```
### Configure MaxScale
Modify /etc/maxscale.cnf:
```sh
#####################################################
# MaxScale documentation:                           #
# https://mariadb.com/kb/en/mariadb-maxscale-24-02/ #
#####################################################

#######################################################################################################
# Global parameters                                                                                   #
#                                                                                                     #
# Complete list of configuration options:                                                             #
# https://mariadb.com/kb/en/mariadb-maxscale-2402-maxscale-2402-mariadb-maxscale-configuration-guide/ #
#######################################################################################################
[maxscale]
threads=auto
log_debug=1

admin_host=0.0.0.0
admin_port=8989
admin_secure_gui=false
#admin_ssl_key=/etc/certs/server-key.pem
#admin_ssl_cert=/etc/certs/server-cert.pem
#admin_ssl_ca_cert=/etc/certs/ca-cert.pem
############################################################################
# Server definitions                                                       #
#                                                                          #
# Set the address of the server to the network address of a MariaDB server.#
############################################################################

#[server1]
#type=server
#address=127.0.0.1
#port=3306

[cloudstackmariadb01]
type=server
address=192.168.XX.XX
port=3306
protocol=MariaDBBackend

[cloudstackmariadb02]
type=server
address=192.168.XX.XX
port=3306
protocol=MariaDBBackend

[cloudstackmariadb03]
type=server
address=192.168.XX.XX
port=3306
protocol=MariaDBBackend


##################################################################################
# Uncomment this and add MaxScale's IP to proxy_protocol_networks in MariaDB for #
# easier user management: https://mariadb.com/kb/en/proxy-protocol-support/      #
##################################################################################
# proxy_protocol=true

##################################################################################################
# Monitor for the servers                                                                        #
#                                                                                                #
# This will keep MaxScale aware of the state of the servers.                                     #
# MariaDB Monitor documentation:                                                                 #
# https://mariadb.com/kb/en/maxscale-24-02monitors/                                              #
#                                                                                                #
# The GRANTs needed by the monitor user depend on the actual monitor.                            #
# The GRANTs required by the MariaDB Monitor can be found here:                                  #
# https://mariadb.com/kb/en/mariadb-maxscale-2402-maxscale-2402-mariadb-monitor/#required-grants #
##################################################################################################

[MariaDB-Monitor]
#type=monitor
#module=mariadbmon
#servers=server1
#user=monitor_user
#password=monitor_pw
monitor_interval=2000ms
type=monitor
module=mariadbmon
servers=cloudstackmariadb01,cloudstackmariadb02,cloudstackmariadb03
user=maxadmin
password=Admin@123
#monitor_interval=5000ms
replication_user=replication_user
replication_password=Pass@123
backend_connect_timeout=2s
backend_write_timeout=2s
backend_read_timeout=2s
backend_connect_attempts=1
master_conditions=connected_slave,running_slave
auto_failover=true
auto_rejoin=true
#failcount=5
#master_failure_timeout=2s
verify_master_failure=true
enforce_read_only_slaves=true
#switchover_timeout=10s
#failover_timeout=10s
#switchover_timeout=90s
##################################################################################################################
# Uncomment these to enable automatic node failover:                                                             #
# https://mariadb.com/kb/en/mariadb-maxscale-2402-maxscale-2402-mariadb-monitor/#cluster-manipulation-operations #
#                                                                                                                #
# The GRANTs required for automatic node failover can be found here:                                             #
# https://mariadb.com/kb/en/mariadb-maxscale-2402-maxscale-2402-mariadb-monitor/#cluster-manipulation-grants     #
##################################################################################################################
# auto_failover=true
# auto_rejoin=true
enforce_simple_topology=true
# replication_user=<username used for replication>
# replication_password=<password used for replication>
#########################################################################################################
# Uncomment this if you use more than one MaxScale with automatic node failover:                        #
# https://mariadb.com/kb/en/mariadb-maxscale-2402-maxscale-2402-mariadb-monitor/#cooperative-monitoring #
#########################################################################################################
cooperative_monitoring_locks=majority_of_all

#########################################################################################################
# Service definitions                                                                                   #
#                                                                                                       #
# Service Definition for a read-only service and a read/write splitting service.                        #
#                                                                                                       #
# The GRANTs needed by the service user can be found here:                                              #
# https://mariadb.com/kb/en/mariadb-maxscale-2402-maxscale-2402-authentication-modules/#required-grants #
#########################################################################################################

################################################################################
# ReadConnRoute documentation:                                                 #
# https://mariadb.com/kb/en/mariadb-maxscale-2402-maxscale-2402-readconnroute/ #
################################################################################

#[Read-Only-Service]
#type=service
#router=readconnroute
#servers=server1
#user=service_user
#password=service_pw
#router_options=slave

#################################################################################
# ReadWriteSplit documentation:                                                 #
# https://mariadb.com/kb/en/mariadb-maxscale-2402-maxscale-2402-readwritesplit/ #
#################################################################################

[Read-Write-Service]
type=service
router=readwritesplit
servers=cloudstackmariadb01,cloudstackmariadb02,cloudstackmariadb03
user=maxadmin
password=Admin@123
max_sescmd_history = 1500
causal_reads = global
causal_reads_timeout =10s
transaction_replay=true
transaction_replay_max_size = 1Mi
delayed_retry = 1
master_reconnection = 1
master_failure_mode = fail_on_write
slave_selection_criteria = ADAPTIVE_ROUTING
####################################################################################################
# Uncomment these to enable transparent transaction replay on node failure:                        #
# https://mariadb.com/kb/en/mariadb-maxscale-2402-maxscale-2402-readwritesplit/#transaction_replay #
####################################################################################################
# transaction_replay=true
transaction_replay_timeout=30s

####################################################################
# Listener definitions for the services                            #
#                                                                  #
# These listeners represent the ports the services will listen on. #
####################################################################

#[Read-Only-Listener]
#type=listener
#service=Read-Only-Service
#port=4008
#port=4006

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=3306
address=0.0.0.0
```
Start MaxScale:
```sh
sudo systemctl restart maxscale
```
## Load Balancing and Failover

MaxScale automatically routes read/write queries.

If a database node fails, MaxScale removes it from the pool.

If a MaxScale node fails, Keepalived can be used to manage a VIP.

### Configuring Keepalived for MaxScale HA

Install Keepalived:
```sh
sudo dnf install -y keepalived
```
Modify /etc/keepalived/keepalived.conf:
```sh
! Configuration File for keepalived


vrrp_script maxscale_health {

#        script "/etc/keepalived/maxscale_health.sh"
script "pidof maxscale"
        interval 2
#       weight -10
   # fall 3
    #rise 3
    }


vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
   #nopreempt
    authentication {
        auth_type PASS
        auth_pass Pass@123
    }
    virtual_ipaddress {
        192.168.17.91
    }
        track_script{
                        maxscale_health
                }
#       notify_master "/etc/keepalived/notify_script.sh MASTER"
#       notify_backup "/etc/keepalived/notify_script.sh BACKUP"
notify "/etc/keepalived/notify_script.sh"
}
```
A scriptfile- 'notify_script.sh' is required to notify keepalived when the service on the maxscale nodes go down so that the VIP can be assigned to the 
next in-line/ backup maxscale node.
```sh
vi /etc/keepalived/notify_script.sh
```
```sh
#!/bin/bash
TYPE=$1
NAME=$2
STATE=$3
OUTFILE=/home/aedc/state.txt

case $STATE in
 "MASTER") echo "Setting this MaxScale node to active mode" > $OUTFILE
              maxctrl alter maxscale passive false
              exit 0
              ;;
 "BACKUP") echo "Setting this MaxScale node to passive mode" > $OUTFILE
              maxctrl alter maxscale passive true
              exit 0
              ;;
 "FAULT")  echo "MaxScale failed the status check." > $OUTFILE
              maxctrl alter maxscale passive true
              exit 0
              ;;
    *)     echo "Unknown state" > $OUTFILE
              exit 1
              ;;
esac
```
Allow permission to the file:
```sh
sudo chmod +x /etc/keepalived/notify_script.sh
```
Restart Keepalived:
```sh
sudo systemctl restart keepalived
```
## Monitoring and Maintenance
Check MaxScale status:
```sh
maxctrl list servers
```
Monitor Replication Status:
```sh
SHOW SLAVE STATUS\G;
```
Backup using MariaDB Backup:
```sh
mariabackup --backup --target-dir=/backup --user=root
```

**Contributing

Contributions are welcome! Please submit a pull request or open an issue for any suggestions.

License

This project is licensed under the MIT License.

References

MariaDB Documentation

MaxScale Documentation
