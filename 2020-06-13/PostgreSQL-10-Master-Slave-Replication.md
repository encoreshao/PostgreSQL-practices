# PostgreSQL 10 Master-Slave Replication on Virtualbox

## Preparations
  - Needs to download tools
    - [Virtualbox](https://www.virtualbox.org/)
    - [Vagrant](https://www.vagrantup.com/)
  - Discover Vagrant Boxes - Linux
    - [Ubuntu 18.04.4 LTS](https://app.vagrantup.com/boxes/search)
  - Two Vagrant boxes
    - Master IP:  192.168.33.11 – read/write
    - Slave IP:   192.168.33.22 – read only

## Postgres Master Configuration
  - Download and installation
    - vagrant init ubuntu/bionic64
  - Update IP and hostname in `Vagrantfile`
    - config.vm.network "private_network", ip: "192.168.33.11"
    - config.vm.hostname = 'master'
  - Start up box
    - vagrant up
  - Connected to the virtual box and install postgreSQL
    ➜  vagrant ssh
    - root@master:~# sudo su
    - root@master:~# apt update
    - root@master:~# apt install postgresql -y
  - Create a role dedicated to the replication
    - root@master:~# sudo su - postgres
    - postgres@master:~$ psql
      - CREATE ROLE replicate WITH REPLICATION LOGIN;
      - \password replicate
  - Modify the `/etc/postgresql/10/main/postgresql.conf`
    ```
    listen_addresses  = '*'
    wal_level         = replica
    max_wal_senders   = 3         # Max number of wal sender processes
    wal_keep_segments = 64        # In logfile segments, 16MB each
    ```
  - Modify the `/etc/postgresql/10/main/pg_hba.conf`, allow access from Slave server
    ```
    host    replication    replicate    192.168.33.22/32    md5
    ```
  - Restart postgreSQL
    - root@master:~# service postgresql restart

## Postgres Slave Configuration
  - Download and installation
    - vagrant init ubuntu/bionic64
  - Update IP and hostname in `Vagrantfile`
    - config.vm.network "private_network", ip: "192.168.33.22"
    - config.vm.hostname = 'master'
  - Start up box
    - vagrant up
  - Connected to the virtual box and install postgreSQL
    ➜  vagrant ssh
    - root@slave:~# sudo su
    - root@slave:~# apt update
    - root@slave:~# apt install postgresql -y
  - Stop the postgreSQL first
    - root@slave:~# service postgresql stop
  - Modify the `/etc/postgresql/10/main/postgresql.conf`
    ```
    hot_standby = on
    ```
  - Modify the `/etc/postgresql/10/main/pg_hba.conf`, allow access from the Master server
    ```
    host    replication    replicate    192.168.33.11/24    md5
    ```
  - Next, we need to delete all the files inside the `PGDATA` folder. need to check the data folder from `postgresql.conf`.
    ```
    root@slave:~# rm -rf /var/lib/postgresql/10/main/*
    ```
  - Now, we will copy all the data from the master with the `pg_basebackup` command. we also need to run this command as the postgres user.
    ```
    sudo su - postgres
    root@slave:~# pg_basebackup -h 192.168.33.11 -D /var/lib/postgresql/10/main/ -P -U replicate --wal-method=stream
    Password:
    23687/23687 kB (100%), 1/1 tablespace
    ```
  - Because archive mode has been activated, a new directory must now be created for archiving in the `PGDATA` directory. with the following commands you create the directory, assign the necessary permissions and change the owner to User postgres.
    ```
    root@slave:~# sudo su
    root@slave:~# chmod 700 /var/lib/postgresql/10/main/
    root@slave:~# chown -R postgres:postgres /var/lib/postgresql/10/main/
    ```
  - Create a new file named `recovery.conf` in `/var/lib/postgresql/10/main`, adding following command to this file:
    ```
    standby_mode          = 'on'
    primary_conninfo      = 'host=192.168.123.10 port=5432 user=replicate password=[password]'
    trigger_file = '/tmp/MasterNow'
    #restore_command = 'cp /home/postgresql_wal/%f "%p"'
    ```
    * Explanation for each line:
      - standby_mode = on: specifies that the server must start as a standby server
      - primary_conninfo: the parameters to use to connect to the master
      - trigger_file: if this file exists, the server will stop the replication and act as a master
      - restore_command: this command is only needed if you have used the archive_command on the master
  - All done, let's start postgreSQL on Slave
    - service postgresql start

####  If everything is fine, we have completed all the configuration of master-slave replication, let's verify the data.
  - check the `replicate` status
    ```
    vagrant@master:~$ sudo su - postgres
    postgres@master:~$ psql
    psql (10.12 (Ubuntu 10.12-0ubuntu0.18.04.1))
    Type "help" for help.
    postgres=# \x on
    Expanded display is on.
    postgres=# SELECT * FROM pg_stat_activity WHERE usename = 'replicate';
    -[ RECORD 1 ]----+------------------------------
    datid            |
    datname          |
    pid              | 7248
    usesysid         | 16388
    usename          | replicate
    application_name | walreceiver
    client_addr      | 192.168.33.22
    client_hostname  |
    client_port      | 42434
    backend_start    | 2020-05-30 08:22:17.163417+00
    xact_start       |
    query_start      |
    state_change     | 2020-05-30 08:22:17.170025+00
    wait_event_type  | Activity
    wait_event       | WalSenderMain
    state            | active
    backend_xid      |
    backend_xmin     |
    query            |
    backend_type     | walsender
    ```
  - Create `testdb` from  master PostgreSQL
    ```
    postgres=# \l
                                  List of databases
       Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
    -----------+----------+----------+---------+---------+-----------------------
     postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
     template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
               |          |          |         |         | postgres=CTc/postgres
     template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
               |          |          |         |         | postgres=CTc/postgres
    (3 rows)

    postgres=# create database testdb;
    CREATE DATABASE
    postgres=#
    ```
  - Read `testdb` from  Slave PostgresSQL
    ```
    postgres=# \l
                              List of databases
       Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
    -----------+----------+----------+---------+---------+-----------------------
     postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
     template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
               |          |          |         |         | postgres=CTc/postgres
     template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
               |          |          |         |         | postgres=CTc/postgres
     testdb    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
    (4 rows)
    ```
  - Nice, all good.

#### Tip: Sometimes, we may need to clean and reinstall PostgreSQL. The following script will help you quickly remove PostgreSQL on Ubuntu 18.04.
  - apt-get — purge remove postgresql
  - dpkg -l | grep postgres
  - apt-get — purge remove pgdg-keyring postgresql*
  - rm -rf /var/lib/postgresql/ /var/log/postgresql/ /etc/postgresql/
  - deluser postgres
  - delgroup postgres
