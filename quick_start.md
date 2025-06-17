[//]: # (This cheat sheet complements the [title]&#40;youtube_link&#41; video tutorial.)

# Prerequisites

To demonstrate how to initialize Greengage DBMS, four hosts with the following names and IP addresses are used:

* `ggdb-cdw` / `10.92.41.125` - the coordinator host.
* `ggdb-scdw` / `10.92.42.240` - the standby coordinator host.
* `ggdb-sdw1` / `10.92.43.73`, `ggdb-sdw2` / `10.92.41.230` - the segment hosts.

Replace the hostnames and IP addresses with values that fit your environment.


# Prepare the environment

## Create the gpadmin user

1. Create the `gpadmin` group:
   ```shell
   sudo groupadd gpadmin
   ```
2. Create a system `gpadmin` user and add it to the `gpadmin` group:
   ```shell
   sudo useradd gpadmin -r -m -g gpadmin
   ```
3. Set the password for the `gpadmin` user:
   ```shell
   sudo passwd gpadmin
   ```


## Configure an operating system

1. Edit the _/etc/sysctl.conf_ file:
   ```shell
   sudo vi /etc/sysctl.conf
   ```
   Add the following content:
   ```properties
   vm.overcommit_memory = 2
   vm.overcommit_ratio = 95
   net.ipv4.ip_local_port_range = 10000 65535
   kernel.sem = 250 2048000 200 8192
   ```
   Apply the changes:
   ```shell
   sudo sysctl --system
   ```
2. Edit the _/etc/security/limits.conf_ file:
   ```shell
   sudo vi /etc/security/limits.conf
   ```
   Add the following content:
   ```
   gpadmin soft nofile 524288
   gpadmin hard nofile 524288
   gpadmin soft nproc 150000
   gpadmin hard nproc 150000
   ```
3. Edit the _/etc/ssh/sshd_config_ file:
   ```shell
   sudo vi /etc/ssh/sshd_config
   ```
   Comment out the `Include` directive:
   ```
   # Include /etc/ssh/sshd_config.d/*.conf
   ```
   Set `PasswordAuthentication` to `yes`:
   ```
   PasswordAuthentication yes
   ```
   Restart `sshd`:
   ```shell
   sudo systemctl restart sshd
   ```
4. Edit _/etc/hosts_:
   ```shell
   sudo vi /etc/hosts
   ```
   Add the following lines:
   ```
   10.92.41.125 ggdb-cdw
   10.92.42.240 ggdb-scdw
   10.92.43.73  ggdb-sdw1
   10.92.41.230 ggdb-sdw2
   ```


# Build from sources

## Clone the Greengage DB repository

1. Verify Git is installed:
   ```shell
   git --version
   ```
2. Clone the `greengage` repository:
   ```shell
   git clone --recurse-submodules https://github.com/GreengageDB/greengage.git
   ```
3. Change the current directory to _greengage_ and check out the latest tag:
   ```shell
   cd greengage
   git checkout tags/7.3.0
   ```
   

## Install the dependencies

1. Execute the _README.Ubuntu.bash_ script to install the dependencies:
   ```shell
   sudo ./README.Ubuntu.bash
   ```
2. Check that the `en_US.utf8` locale is installed:
   ```shell
   locale -a
   ```
   

## Build and install Greengage DB

1. Configure the build environment:
   ```shell
   ./configure --with-perl --with-python --with-libxml --with-uuid=e2fs --with-openssl --with-gssapi --with-ldap --enable-ic-proxy --enable-orafce --prefix=/usr/local/gpdb
   ```
2. Compile Greengage DB:
   ```shell
   make -j$(nproc)
   ```
3. Install Greengage DB:
   ```shell
   sudo make -j$(nproc) install
   ```
4. Change the owner and group of the installed files to `gpadmin`:
   ```shell
   sudo chown -R gpadmin:gpadmin /usr/local/gpdb* && sudo chgrp -R gpadmin /usr/local/gpdb*
   ```
   

# Initialize the database system


## Create the data storage areas

1. Create the _/data1/coordinator_ directories on the coordinator hosts:
   ```shell
   sudo mkdir -p /data1/coordinator
   sudo chown gpadmin:gpadmin /data1/coordinator
   ```
2. Create directories on the segment hosts:
   ```shell
   sudo mkdir -p /data1/primary
   sudo mkdir -p /data1/mirror
   sudo chown -R gpadmin /data1/*
   ```


## Switch to gpadmin

1. Switch to the `gpadmin` user:
   ```shell
   sudo su - gpadmin
   ```
2. Switch to a Bash shell:
   ```shell
   bash
   ```
   

## Set Greengage DB environment variables

1. Open the _.bashrc_ file:
   ```shell
   vi ~/.bashrc
   ```
2. Add the following lines:
   ```bash
   source /usr/local/gpdb/greengage_path.sh
   export COORDINATOR_DATA_DIRECTORY=/data1/coordinator/gpseg-1
   export PGPORT=5432
   export PGUSER=gpadmin
   export PGDATABASE=postgres
   ```
3. Apply the changes:
   ```shell
   source ~/.bashrc
   ```
   

## Generate SSH key pairs

1. Generate an SSH key pair for the `gpadmin` user:
   ```shell
   ssh-keygen -t rsa -b 4096
   ```
2. Press `Enter` to use the default paths for SSH key files.
3. Press `Enter` two times to skip specifying a passphrase.


## Create the host files

1. Create the _hostfile_all_hosts_ file:
   ```shell
   vi hostfile_all_hosts
   ```
   Add all the host names:
   ```
   ggdb-cdw
   ggdb-scdw
   ggdb-sdw1
   ggdb-sdw2
   ```
2. Create the _hostfile_segment_hosts_ file:
   ```shell
   vi hostfile_segment_hosts
   ```
   Add the host names of segment hosts:
   ```
   ggdb-sdw1
   ggdb-sdw2
   ```
   

## Enable 1-n passwordless SSH

1. Execute `ssh-copy-id` to add the `gpadmin` user's public key to the `authorized_keys` SSH file on the `scdw` host:
   ```shell
   ssh-copy-id ggdb-scdw
   ```
2. Enter the password of the `gpadmin` user for the `scdw` host and press `Enter`.
3. Repeat the steps for the segment hosts:
   ```shell
   ssh-copy-id ggdb-sdw1
   ssh-copy-id ggdb-sdw2
   ```
   

## Enable n-n passwordless SSH

1. Use the gpssh-exkeys utility:
   ```shell
   gpssh-exkeys -f hostfile_all_hosts
   ```
   

## Initialize a cluster

1. Create the _init_config_ file:
   ```shell
   vi init_config
   ```
2. Specify the following settings:
   ```
   ARRAY_NAME="Greengage DB cluster"
   SEG_PREFIX=gpseg
   PORT_BASE=10000
   declare -a DATA_DIRECTORY=(/data1/primary /data1/primary )
   COORDINATOR_HOSTNAME=ggdb-cdw
   COORDINATOR_DIRECTORY=/data1/coordinator
   COORDINATOR_PORT=5432
   TRUSTED_SHELL=ssh
   CHECK_POINT_SEGMENTS=8
   ENCODING=UNICODE
   MIRROR_PORT_BASE=10500
   declare -a MIRROR_DATA_DIRECTORY=(/data1/mirror /data1/mirror )
   ```
3. Run the initialization utility:
   ```shell
   gpinitsystem -c init_config -h hostfile_segment_hosts -s ggdb-scdw -n en_US.UTF-8
   ```
   

## Check the cluster state

1. Execute the `gpstate` command:
   ```shell
   gpstate
   ```
2. List all databases:
   ```shell
   psql -l
   ```
3. Connect to the default database:
   ```shell
   psql
   ```
4. Get the version information:
   ```sql
   SELECT version();
   ```
5. Query the `gp_segment_configuration` table:
   ```sql
   SELECT * FROM gp_segment_configuration;
   ```
   

# Load sample data

1. Create a database and connect to it:
   ```sql
   CREATE DATABASE marketplace;
   \c marketplace
   ```
2. Create a table:
   ```sql
   CREATE TABLE orders
   (
       order_id    INTEGER,
       customer_id INTEGER,
       amount      DECIMAL(6, 2)
   )
       WITH (appendoptimized = true)
       DISTRIBUTED BY (customer_id);
   ```
3. Insert data:
   ```sql
   INSERT INTO orders (order_id, customer_id, amount)
   SELECT order_number                               AS order_id,
          FLOOR(RANDOM() * 1000 + 1)::INTEGER        AS customer_id,
          ROUND((100 + RANDOM() * 1000)::NUMERIC, 2) AS amount
   FROM generate_series(1, 100000) AS order_number;
   ```
4. Check data distribution:
   ```sql
   SELECT get_ao_distribution('orders');
   ```
   

# Set up user access

1. Create a role:
   ```sql
   CREATE USER sampleuser WITH PASSWORD '123456';
   ```
2. Grant privileges:
   ```sql
   GRANT SELECT ON TABLE orders TO sampleuser;
   ```
3. Try to connect to the database:
   ```shell
   psql -d marketplace -U sampleuser -h 10.92.41.125 -p 5432
   ```
4. Exit `psql`:
   ```sql
   \q
   ```
5. Edit the _pg_hba.conf_ file:
   ```shell
   vi $COORDINATOR_DATA_DIRECTORY/pg_hba.conf
   ```
   Add the following line:
   ```
   host     marketplace sampleuser      10.92.12.8/32         md5
   ```
6. Reload the configuration:
   ```shell
   gpstop -u
   ```
7. Connect to the database:
   ```shell
   psql -d marketplace -U sampleuser -h 10.92.41.125 -p 5432
   ```
8. Select data:
   ```sql
   SELECT * FROM orders ORDER BY amount DESC LIMIT 5;
   ```
