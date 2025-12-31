Build Dockerfile

```bash
docker build -t pgBackupRest-image:1.0 .
```
Setup pgbackup_server host/container

```bash
# Gen ssh key
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# Copy to target
ssh-copy-id postgres@postgres_coord1

# Test ssh
#ssh 'pgbackrest@postgres_coord1'
ssh postgres@postgres_coord1 date
```

# Remove known_hosts
```bash
ssh-keygen -f '/home/pgbackrest/.ssh/known_hosts' -R 'postgres_work1-1'
```

Setup cooord/worker - host/container
```bash
# 1. Enter the coordinator container as root
docker exec -it -u root postgres_coord1 bash

# 2. Update and install OpenSSH server
apt-get update && apt-get install -y openssh-server pgbackrest sudo

# 3. Create the required privilege separation directory
mkdir -p /var/run/sshd

# 4. Start the SSH service
/usr/sbin/sshd
```
Create pgbackrest on the coord/worker node
```bash
useradd -m -d pgbackrest
passwd pgbackrest
usermod -aG postgres pgbackrest
```

## Gen ssh keys on the coord/worker node
```bash
docker exec -it -u postgres postgres_coord1 bash
# Gen ssh key
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# Copy to target
ssh-copy-id pgbackrest@pgbackup_server

# Test ssh
#ssh 'pgbackrest@pgbackup_server'
ssh pgbackrest@pgbackup_server date
```
Use postgres user to perform pgbackrest
```bash
# 1. Fix the home directory (must not be group-writable)
chown postgres:postgres /var/lib/postgresql
chmod 755 /var/lib/postgresql

# 2. Fix the .ssh directory (must be 700)
mkdir -p /var/lib/postgresql/.ssh
chown postgres:postgres /var/lib/postgresql/.ssh
chmod 700 /var/lib/postgresql/.ssh

# 3. Fix the authorized_keys file (must be 600)
touch /var/lib/postgresql/.ssh/authorized_keys
chown postgres:postgres /var/lib/postgresql/.ssh/authorized_keys
chmod 600 /var/lib/postgresql/.ssh/authorized_keys

# 4. Ensure the postgres user has a valid shell
usermod -s /bin/bash postgres
```

Run these commands as root on coord/worker nodes when using the pgbackrest user separately

```bash
# 1. Clean the folder one last time
rm -rf /tmp/pgbackrest
mkdir -p /tmp/pgbackrest

# 2. Change group ownership to 'postgres' (the common group both users should belong to)
# Note: Ensure the user 'pgbackrest' is in the 'postgres' group: 
# usermod -aG postgres pgbackrest
chown postgres:postgres /tmp/pgbackrest

# 3. Give both User and Group full R/W/X permissions
chmod 775 /tmp/pgbackrest

# 4. Set the 'Setgid' bit so new files inherit the group 'postgres'
chmod g+s /tmp/pgbackrest

# On postgres_coord1
sudo usermod -aG postgres pgbackrest
```

## Allow pgbackrest to do the backup on cluster nodes (pgbackrest user separately)
Using ACLs (Recommended)
```bash
# 1. Install the acl package
apt-get update && apt-get install -y acl

# 2. Grant the pgbackrest user READ and EXECUTE permissions on the data folder
setfacl -R -m u:pgbackrest:rx /var/lib/postgresql/data

# 3. Ensure future files created by Postgres also allow pgbackrest to read
setfacl -R -d -m u:pgbackrest:rx /var/lib/postgresql/data
```

Locking on coord/worker nodes
```bash
chown -R postgres:postgres /var/lib/pgbackrest
chown -R postgres:postgres /var/log/pgbackrest

chmod 775 /var/lib/pgbackrest
chmod 775 /var/log/pgbackrest
chmod g+s /var/lib/pgbackrest
```

Init/Run the bgbackrest

## Create postgres user bgbackrest on the primary coord/worker nodes

```bash
docker exec -it postgres_coord3 psql -U postgres -c "CREATE USER pgbackrest SUPERUSER;"
docker exec -it postgres_coord3 bash -c "echo 'local all pgbackrest trust' >> /var/lib/postgresql/data/pgdata/pg_hba.conf"
docker exec -it postgres_coord3 psql -U postgres -c "SELECT pg_reload_conf();"
```

Init pgbackrest
```bash
pgbackrest --stanza=citus_cluster stanza-create
```

```bash
pgbackrest --stanza=citus_worker1 stanza-create
```

Run pgbackrest start
```bash
pgbackrest --stanza=citus_coord  start
```

Run pgbackrest check
```bash
pgbackrest --stanza=citus_coord check
```

Run pgbackrest backup
```bash
pgbackrest --stanza=citus_coord --type=full --log-level-console=info backup
```

Run pgbackrest info
```bash
pgbackrest info
```

CLEAN UP the pgbackrest created

```bash
# Stop
pgbackrest --stanza=citus_cluster stop
# Delete
pgbackrest --stanza=citus_cluster --force stanza-delete
```