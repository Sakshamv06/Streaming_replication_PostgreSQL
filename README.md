# Setting Up Master-Slave PostgreSQL in VMs

## Overview
Setting up a master-slave PostgreSQL configuration in Virtual Machines (VMs) means creating a system where one PostgreSQL database server (the master) handles all the write operations, and one or more other servers (the slaves) replicate the data from the master to handle read operations.

## Goals
The goal of setting up a Master-Slave PostgreSQL system is to ensure smooth database operation and high availability. The master handles writing data, while the slave copies the data and helps with reading it. This setup enhances backup, performance, and fault tolerance.

## Prerequisites
- One VM for the Master PostgreSQL server
- One VM for the Slave PostgreSQL server

## Step 1: Installation of PostgreSQL
### On Master and Slave
Ubuntu includes PostgreSQL by default. To install PostgreSQL on Ubuntu, use the following command:
```bash
apt install postgresql
```
If the default version is not desired, use the PostgreSQL Apt Repository:

Create a script to configure the repository:
```bash
vim psql.sh
```
Add the following content:
```bash
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo apt update
sudo apt -y install postgresql
```
Save and exit the file (`wq!`).

Give execute permissions and run the script:
```bash
chmod +x psql.sh
./psql.sh
```

Verify PostgreSQL installation:
```bash
psql -U postgres
psql --version
```

## Step 2: Configure the Master Server
Modify the PostgreSQL configuration files to allow replication.

Edit `postgresql.conf`:
```bash
sudo vim /etc/postgresql/17/main/postgresql.conf
```
Modify the following parameters:
```ini
listen_addresses = '*'
ssl = off
shared_buffers = 256MB
wal_level = replica
max_wal_size = 1GB
min_wal_size = 80MB
archive_mode = on
max_wal_senders = 10
wal_keep_size = 16MB
hot_standby = on
```
Save and exit (`wq!`).

Edit `pg_hba.conf` to allow access from the Slave server:
```bash
sudo vim /etc/postgresql/17/main/pg_hba.conf
```
Add the following lines:
```ini
host	all		all		192.168.122.0/24	trust
host	replication	test	192.168.122.196/32	md5
```

## Step 3: Enable Replication on the Master
Create a replication user:
```sql
psql -U postgres
CREATE ROLE test WITH REPLICATION PASSWORD '1234' LOGIN;
```
Verify the role creation:
```sql
\du
```

## Step 4: Configure the Slave Server
Stop PostgreSQL and remove old data:
```bash
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/17/main/*
```

## Step 5: Start the Replication
Initiate backup from the Master:
```bash
sudo -u postgres pg_basebackup -h 192.168.122.101 -p 5432 -U test -D /var/lib/postgresql/17/main/ -Fp -Xs -R
```
Restart PostgreSQL on the Slave:
```bash
sudo systemctl start postgresql
```

## Step 6: Verify Replication
Check replication status on the Master:
```sql
SELECT client_addr, state FROM pg_stat_replication;
```
Alternatively:
```sql
SELECT * FROM pg_stat_replication;
SELECT * FROM pg_stat_activity WHERE usename = 'test';
```
On the Slave:
```sql
SELECT * FROM pg_stat_wal_receiver;
