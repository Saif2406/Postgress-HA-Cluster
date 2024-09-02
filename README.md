# Postgress-HA-Cluster
Application can handle failures and outages is crucial, and the availability of your application is only as good as the availability of your PostgreSQL instance. With that in mind, you may be wondering which PostgreSQL high availability (HA) deployment option is best for your application.

![image](https://github.com/user-attachments/assets/0575f5c8-5b73-44f3-bcc3-d7100dca0a3e)

The client application connects to a virtual IP address whose configuration supports the sending of traffic across zone A,B and C. During normal operations, the virtual IP forwards requests to the active instance of Pgpool in availability zone A. The primary PostgreSQL instance runs in the same zone and receives requests from the above mentioned Pgpool instance.

The standby instance is located in availability zone B & C on the basis of setting prority level in Keepalived Service. The replication from the primary instance to standby instances is synchronous

A highly available architecture assumes that any single points of failure will be eliminated, which is why the primary Pgpool instance from zone A is complemented with a standby one in zone B and second in zone C . The Keepalived service of Pgpool work in all 3 zones and monitor the healthiness of the Pgpool processes.

Example : Imagine that the primary PostgreSQL instance goes down. If that happens, the active Pgpool instance in zone A will failover to the standby PostgreSQL instance in zone B and zone B instance act as Master and zone C instance would be next Standy instance If the entire zone A goes down, the virtual IP can be reassigned to the standby Pgpool instance in zone B and if zone B goes down then zone C, and that instance will forward all requests to the standby PostgreSQL instance within the same zone.
