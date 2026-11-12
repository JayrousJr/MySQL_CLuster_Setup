# MySQL CLuster Configurations

## 1. Setting up the network

### In Cluster Management we always have the following

#### A. Management node

- Holds the cluster configuration, starts and stops nodes, and acts as arbitrator. Runs `ndb_mgmd` Program

#### B. Data node

- Stores and replicates the data. Runs `ndbd` or `ndbmtd` (multi-threaded) Program

#### C. SQL node

- The normal MySQL server your apps connect to; it reads and writes via the data nodes
  Command. Runs `mysqld`

Your apps talk to the SQL nodes on port 3306. The SQL nodes talk to the data nodes, and the management node only coordinates.

Adding the

```
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
cp mysql-cluster-26.10.0-linux-glibc2.28-aarch64.tar.xz /usr/local
rm -r mysql-cluster-26.10.0-linux-glibc2.28-aarch64
cd /usr/local
tar xf mysql-cluster-26.10.0-linux-glibc2.28-aarch64.tar.xz
ln -s mysql-cluster-26.10.0-linux-glibc2.28-aarch64 mysql
rm -r mysql-cluster-26.10.0-linux-glibc2.28-aarch64.tar.xz
echo 'export PATH=$PATH:/usr/local/mysql/bin' >> ~/.bashrc
source ~/.bashrc
ndb_mgmd --version

```

```
[ndbd default]
NoOfReplicas=3
DataMemory=256M
DataDir=/var/lib/mysql-cluster

[ndb_mgmd]
NodeId=1
HostName=172.15.14.176
DataDir=/home/legolas/mysql-cluster

[ndbd]
NodeId=2
HostName=172.15.14.161

[ndbd]
NodeId=3
HostName=172.15.14.162

[ndbd]
NodeId=4
HostName=172.15.14.163

[mysqld]
NodeId=11
HostName=172.15.14.161

[mysqld]
NodeId=12
HostName=172.15.14.162

[mysqld]
NodeId=13
HostName=172.15.14.163
```
