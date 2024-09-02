PostgreSQL Master name and IP address: PGMaster and 192.168.100.20
PostgreSQL Slave/Replica name and IP address: PGSlave and 192.168.100.21 and 192.168.100.22
All 3 instance binding with 192.168.100.200(cluster Ip address for application to communicate with)
On Master and Slave servers, install PostgreSQL 16 and start the postgressql 

------------------------------------------------Steps for MasterNode-------------------------------------------------------------

1.On master server, configure the IP address(es) listen to for connections from clients in postgresql.conf by removing # in front of listen_address and give *. 
all. listen_addresses = '*' 

2. Now, connect to PostgreSQL on master server and create replica login
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'admin@123'; 

3. Enter the following entry pg_hba.conf file which is located in /etc/postgresql/16/main on Ubuntu(debian systems). 
nano /etc/postgresql/16/main/pg_hba.conf
host replication replicator 192.168.100.94/24 md5
host replication replicator 192.168.100.111/24 md5

---------------------------------------------------Steps for Worker Nodes---------------------------------------------------------

Configurations on slave(standby) servers 

1.We have to stop PostgreSQL on Slave server by using following command. 
   a.	sudo systemctl stop postgresql 
   b.	sudo systemctl status postgresql 

2.Now, switch to postgres user and take backup of main(data) directory. su - postgres cp -R /var/lib/postgresql/16/main/ /var/lib/postgresql/16/main_old/ 

3.Now, remove the contents of main(data) directory on slave server. rm -rf /var/lib/postgresql/16/main/ 

4.Now, use basebackup to take the base backup with the right ownership with postgres(or any user with right permissions). 
    pg_basebackup -h 192.168.72.128 -D /var/lib/postgresql/16/main/ -U replicator -P -v -R -X stream -C -S slaveslot1
Then provide the password for user replicator created in master server. 
     it should look like this : pg_basebackup: initiating base backup, 
                                waiting for checkpoint to complete .................................... 
                                pg_basebackup: syncing data to disk ... 
                                pg_basebackup: base backup completed

5.Notice that standby.signal is created and the connection settings are appended to postgresql.auto.conf
      ls -ltrh /var/lib/postgresql/16/main/ 

6.postgresql.conf and there is a standby.signal file present in the data directory.

7.Now connect the master server, you should be able to see the replication slot called slotslave1 when you open the pg_replication_slots view as follows.
      SELECT * FROM pg_replication_slots; 

8.Test replication setup 1. Now start PostgreSQL on slave(standby) server. 
       systemctl start postgresql 

9.Now, try to create object or database in slave(standby) server. It throws error, because slave(standby) is read-only server. create database slave1;

10.WE can check the status on standby using below command. SELECT * FROM pg_stat_wal_receiver;

11.Now, verify the replication type synchronous or aynchronous using below command on master database server. 
        SELECT * FROM pg_stat_replication; 

12.Lets create a database in master server and verify its going to replicate to slave or not. 
        create database stream; 

13.Now, connect to slave and verify the database copied or not. 
        select datname from pg_database; 

14.. If you want to enable synchronous, the run the below command on master database server and reload postgresql service
         ALTER SYSTEM SET synchronous_standby_names TO '*'; 
         systemctl reload postgresql

----------------------------------------------------------Keepalived--------------------------------------------------------------------------

The goal is to ensure high availability, where the VIP will always point to the active primary node. If the primary fails, one of the standby nodes will take over.

Initial PostgreSQL Keepalived Setup (Primary Node - 192.168.100.20):

1.	Edit postgresql.conf:
       sudo nano /etc/postgresql/16/main/postgresql.conf
Ensure the following settings are enabled:
        listen_addresses = '*'
        wal_level = replica
        max_wal_senders = 10
        hot_standby = on
2.	Configure pg_hba.conf:
       sudo nano /etc/postgresql/16/main/pg_hba.conf
Add the following lines to allow replication:
      host replication replicator 192.168.100.94/32 md5
      host replication replicator 192.168.100.111/32 md5
Install and Configure Keepalived
Install keepalived on all three nodes.
    sudo apt-get install keepalived

Configure Keepalived on All Nodes:
    1.	Edit /etc/keepalived/keepalived.conf on all nodes with the following configuration.
    2.	List network interfaces
    3.	Handle the keepalived_script User Warning

List network interfaces:
    ip link show
Look for the correct interface name (it might be something like ens33, eth1, or similar).
Handle the keepalived_script User Warning
You can either create the keepalived_script user or run the script as the root user.
 Create the user:
     sudo useradd -r -s /bin/false keepalived_script
This creates a system user with no shell access, just for script execution.

     Nano /etc/keepalived/keepalived.conf
     vrrp_script chk_pgsql {
      script "pgrep postgres"
       interval 2
        weight 2
       user root
}

 vrrp_instance VI_1 {
    state MASTER
    interface ens32
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass gama1234
    }

    virtual_ipaddress {
        192.168.100.200  # This is the VIP
    }

    track_script {
        chk_pgsql
    }
}

On 192.168.100.21 (Standby Node 1):
List network interfaces:
ip link show
Look for the correct interface name (it might be something like ens33, eth1, or similar).

Handle the keepalived_script User Warning
You can either create the keepalived_script user or run the script as the root user.
 Create the user:
sudo useradd -r -s /bin/false keepalived_script
This creates a system user with no shell access, just for script execution.

Nano /etc/keepalived/keepalived.conf

vrrp_script chk_pgsql {
    script "pgrep postgres"
    interval 2
    weight 2
    user root 
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens32
    virtual_router_id 51
    priority 90
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass gama1234
    }

    virtual_ipaddress {
        192.168.100.200
    }

    track_script {
        chk_pgsql
    }
}

On 192.168.100.23 (Standby Node 2):
List network interfaces:
ip link show
Look for the correct interface name (it might be something like ens33, eth1, or similar).
Handle the keepalived_script User Warning
You can either create the keepalived_script user or run the script as the root user.
 Create the user:
sudo useradd -r -s /bin/false keepalived_script
This creates a system user with no shell access, just for script execution.

Nano /etc/keepalived/keepalived.conf
vrrp_script chk_pgsql {
    script "pgrep postgres"
    interval 2
    weight 2
    user root
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens32
    virtual_router_id 51
    priority 80
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass gama1234
    }

    virtual_ipaddress {
        192.168.100.200
    }

    track_script {
        chk_pgsql
    }
}
•	state: Set to MASTER on the primary node and BACKUP on the standby nodes.
•	priority: The node with the highest priority will be the MASTER. Lower priorities are used for backup nodes.
•	virtual_ipaddress: This is the VIP that will float between the nodes.
2.	Start Keepalived:
On each node, start and enable keepalived:
           sudo systemctl start keepalived
           sudo systemctl enable keepalived
           sudo systemctl status keepalived

ip addr show (Check in all nodes of biding 192.168.100.200 with current IP addr 192.168.100.21)

then you will able to use the database and login through UI DBaver using database credentails and ip address :2181

Thanks.








