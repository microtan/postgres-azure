Setting up PostgreSQL on Microsoft Azure Virtual Machines (IaaS)
================================================================

# Login to Azure

- Download publish settings at https://manage.windowsazure.com/publishsettings/index?client=xplat 

```console
npm install azure-cli
azure account clear
azure account import "Windows Azure MSDN - Visual Studio Ultimate-credentials.publishsettings"
azure account set "internal"
azure account list
```

# Get the image

- Debian Wheezy Image from https://vmdepot.msopentech.com/Vhd/Show?vhdId=65&version=400

```console
azure vm list --json
azure vm create DNS_PREFIX --community vmdepot-65-6-32 --virtual-network-name  -l "West Europe" USER_NAME [PASSWORD] [--ssh] [other_options]

Create an A5 instance
```

# Command line for creating a PostgreSQL machine

Command line:

```console
azure vm create-from cloudservicename machine.json --connect --verbose --json
```

## machine.json

See the [REST API](http://msdn.microsoft.com/en-us/library/azure/jj157194.aspx) for details. 

```JSON
{
	"RoleName": "database-vm-1",
	"RoleType": "PersistentVMRole",
	"RoleSize": "A5",
	"AvailabilitySetName" : "databases",
	"OSVirtualHardDisk": {
		"OS": "Linux",
		"HostCaching": "ReadWrite",
		"DiskName": "database-vm-1-disk",
		"DiskLabel": "database-vm-1-disk",
		"SourceImageName": "Debian-Wheezy-635506180993665396",
		"RemoteSourceImageLink": "http://account.blob.core.windows.net/vmdepot-images/TE-2014-11-03-debianwheezy-os-2014-11-03.vhd",
		"MediaLink" : "http://account.blob.core.windows.net/vmdepot-images/database-vm-1-disk.vhd"
	},
	"DataVirtualHardDisks": [
		{ "DiskLabel": "database-vm-1-data1", "Lun": "0", "MediaLink" : "http://account.blob.core.windows.net/vmdepot-images/database-vm-1-data1.vhd", "HostCaching": "ReadOnly", "LogicalDiskSizeInGB": "1023" },
		{ "DiskLabel": "database-vm-1-data2", "Lun": "1", "MediaLink" : "http://account.blob.core.windows.net/vmdepot-images/database-vm-1-data2.vhd", "HostCaching": "ReadOnly", "LogicalDiskSizeInGB": "1023" },
		{ "DiskLabel": "database-vm-1-xlog1", "Lun": "2", "MediaLink" : "http://account.blob.core.windows.net/vmdepot-images/database-vm-1-xlog1.vhd", "HostCaching": "ReadOnly", "LogicalDiskSizeInGB": "1023" } 
	],
	"ConfigurationSets": [
		{
			"ConfigurationSetType" : "LinuxProvisioningConfiguration",
			"HostName" : "database-vm-1",
			"UserName" : "ruth",
			"UserPassword" : "Supersecret123!!",
			"DisableSshPasswordAuthentication" : false
		},
		{
			"ConfigurationSetType": "NetworkConfiguration",
			"SubnetNames": [ "mysubnet" ],
			"StaticVirtualNetworkIPAddress": "10.10.0.7",
			"InputEndpoints": [],
			"PublicIPs": [],
			"StoredCertificateSettings": []
		}
	],
	"ProvisionGuestAgent": "true",
	"ResourceExtensionReferences": []
}
```

# Bring Linux up to date

```console
$ aptitude update
$ aptitude update && aptitude upgrade
$ aptitude install rsync
$ aptitude install mdadm
$ aptitude install lvm2
$ aptitude install xfsprogs

```

# Setup striping

- Multiple data disks in a [RAID](http://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-configure-raid/)  in order to achieve higher I/O, given current limitation of 500 IOPS per data disk. 
- One data disk for pg_xlog

- Use 'cfdisk' on /dev/sdc and create a primary partition of tyoe 'FD' (RAID autodetect) for RAID for pg_data
- Use 'cfdisk' on /dev/sdd and create a primary partition of tyoe 'FD' (RAID autodetect) for RAID for pg_data
- Use 'cfdisk' on /dev/sde and create a primary partition of tyoe '8E' (LVM) for pg_xlog


```console
$ mdadm --create /dev/md0 --level 0 --raid-devices 2 /dev/sdc1 /dev/sdd1

# create physical volume
$ pvcreate /dev/md0

# create volume group
$ vgcreate data /dev/md0

# create logical volume 
$ lvcreate -n pgdata -l100%FREE data

# show volume group information
$ vgdisplay

# Now 
$ ls -als /dev/data/pgdata

# mkfs -t xfs /dev/data/pgdata
$ mkfs.xfs /dev/data/pgdata
```

Setup automount

```console
$ tail /etc/fstab

/dev/mapper/data-pgdata /space/pgdata xfs defaults 0 0
/dev/mapper/xlog-pgxlog /space/pgxlog xfs defaults 0 0
```






# PostgreSQL base install



## Install PostgreSQL 

Install PostgreSQL, as documented under https://wiki.postgresql.org/wiki/Apt 

```
$ sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

$ aptitude install wget ca-certificates

$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

$ aptitude install postgresql-9.3

$ aptitude install repmgr 
```


## Move database files into striped volume 

```
mv    /var/lib/postgresql/9.3 /space/pgdata/
ln -s /space/pgdata/9.3       /var/lib/postgresql/9.3

mv    /space/pgdata/9.3/main/pg_xlog /space/pgxlog/9.3
ln -s /space/pgxlog/9.3              /space/pgdata/9.3/main/pg_xlog
```


# Distribute SSH keys for user postgres across the cluster

Whatever it takes




## Edit /etc/postgresql/9.3/main/postgresql.conf

Uncomment listen_addresses (Database only reachable through jump host)

```
listen_addresses = '*'
```

Switch off SSL

```
ssl = false
```

Rule of thumb for shared buffers: 25% of RAM should be shared buffers, on an A5 

```
shared_buffers = 4GB
work_mem = 256MB
maintenance_work_mem = 512MB
```

The write-ahead-log needs to be merged at regular checkpoints into the tables:

```
checkpoint_segments = 64                # was 3 previously in logfile segments, min 1, 16MB each
checkpoint_timeout = 1min               # range 30s-1h
checkpoint_completion_target = 0.8      # checkpoint target duration, 0.0 - 1.0
```

Put replication configuration into dedicated file

```
include_if_exists = 'replication.conf'
```

# Edit /etc/postgresql/9.3/main/replication.conf

## wal_keep_segments

How many segments to hold in xlog folder. Having a longer value allows slaves to keep up. 500*16 MB = 8 GB

```
wal_keep_segments=500
```
## wal_level

All nodes (master and slaves) need to be in hot_standby to that slaves can become master. 

```
wal_level='hot_standby'
```

## archive_mode and archive_command

```
archive_mode=on
archive_command='cd .'
```

## max_wal_senders

Number of machines, takes away from max_connections. Should be very similar to the size of the cluster. Between 5 and 10 is "OK". Numbers like 500 kill the machine

```
max_wal_senders=5
```

## hot_standby

Allows slaves to already answer to read queries.

```
hot_standby=on
```

# Add postgres user with replication priviledge and superuser 

```console
# username "repl" 
# -P = get password from <stdin>
# -S = super user, in order to run repmgr (otherwise, error "permission denied for language C" comes)

$ su postgres
$ createuser --replication -P -S  repl 
```

## Edit /etc/postgresql/9.3/main/pg_hba.conf

As postgres user, add the following line to allow subnet 10.10.0.0/16 to do replication with user ID "repl"

```
host   all            all     10.10.0.0/16    md5
host   replication    repl    10.10.0.0/16    md5
```

## Configure /var/lib/postgresql/repmgr.conf on all hosts. 

As postgres user, These settings are written to the DB, and visible in the cluster, so IPs must be real ones . 

```
All nodes have the same cluster name
cluster=my_application_cluster
pg_bindir='/usr/lib/postgres/9.3/bin'

# This must unique be for each respective node
node=1      
node_name=postgresvm1
conninfo='host=10.10.0.7 user=repl dbname=repmgr'
```


## Setup repmgr internals on master node.  

### vim /var/lib/postgresql/.pgpass

```
# hostname:port:database:username:password
*:*:*:repl:supersecret123.-
```

Permissions

```console
$ chmod 0600  /var/lib/postgresql/.pgpass
```

### Setup repmgr on master node

```console
$ sudo postgres    / su - postgres
$ repmgr -f /var/lib/postgresql/repmgr.conf --verbose master register
```

### Setup repmgr on standby nodes (slaves) *before starting postgres on the slaves*

```console
$ sudo postgres    / su - postgres

$ service postgresql stop

# -d database
# -U user
# -R rsync user
# -D data dir
# -w WAL keep segments (default is 5000, which is too large)
# 10.10.0.7 IP address of master

$ repmgr -d repmgr \
	-U repl \
	-R postgres -w 500 \
	-D /var/lib/postgres/9.3/main \
	-f /var/lib/postgresql/repmgr.conf \
	--verbose \
	standby clone 10.10.0.7

$ service postgresql start

$ repmgr -f /var/lib/postgresql/repmgr.conf \
	--verbose standby register
```















# Local agent

## When the current master goes down

When master gets shutdown signal, 

1. Refuse additional (new) connections: 
	-	change file [pg_hba.conf](http://www.postgresql.org/docs/9.1/static/auth-pg-hba-conf.html) to reject new connections
	- "SELECT pg_reloadconf();" or "pg_ctl reload conf" or "kill -HUP" to enact configuration
2. [Drop existing sessions](http://www.devopsderek.com/blog/2012/11/13/list-and-disconnect-postgresql-db-sessions/)
	- "SELECT pg_terminate_backend( <procpid> )"
3. Instruct PostgreSQL to write (flush) remaining transaction log (WAL records) to tables by creating a checkpoint 


### Refuse additional (new) connections

Edit [pg_hba.conf](http://www.postgresql.org/docs/9.1/static/auth-pg-hba-conf.html): Uncomment the "all/all" line, so that nobody can create additional connections. 

```
# host   all            all     10.10.0.0/16    md5
```

And reload config

```SQL
SELECT pg_reload_conf();
```

### Drop existing sessions from the web tier

```SQL
SELECT pg_terminate_backend(pid) 
	FROM pg_stat_activity
	WHERE usename='webfrontend';
```

	(Yes, it is 'usename', not 'username')...


### Create a checkpoint on master via SQL

```SQL
CHECKPOINT;
```

### Determine xlog location 

Determine xlog location of the current xlog position, something after the previously made checkpoint. Here, we can be sure that after the checkpoint, only non-relevant changes (like vacuuming) happened to the tables. 

Now fetch (once) after the checkpoint operation on the master the XLOG location, and store it in a variable `checkpointXlog`:  

```SQL
SELECT pg_current_xlog_location();
```

Determine replication lag for the slaves. When the `pg_xloc_location_diff(...)` function call returns 0, all slaves have catched up. Running below code gives a current view: 

```SQL
SELECT client_addr, 
	flush_location, 
	pg_current_xlog_location(), 
	pg_xlog_location_diff(
		pg_current_xloc_location(), 
		flush_location) 
	from pg_stat_replication;
```

Using the post-checkpoint variable `checkpointXlog`, you can now determine whether it is safe to kill the master. 

```ruby
var checkpointXlog = eval("SELECT pg_current_xlog_location();")

checkpointXlog == '0/14047810'
```

Now we can determine the concrete replication lag:

```SQL
SELECT client_addr, 
	flush_location, 
	'0/14047810', 
	pg_xlog_location_diff(
		'0/14047810', 
		flush_location) 
	from pg_stat_replication;
```

- When the `pg_xlog_location_diff` column has non-positiv values, it's safe to shoot the master in the head. 
- When we compare against `flush_location`, we not it's on the harddisk of the slave. 
- When we compare against `replay_location`, we know it's in the actual database tables. 

Stop old master server 

```console
$ sudo postgres    / su - postgres
$ service postgresql stop
```

## Turn one of the slaves into the new master (`repmgr standby promote`)

Use either `repmgr standby promote` (as a convenient wrapper) or naked `pg_ctl promote`.

```bash
$ sudo postgres    / su - postgres
$ repmgr -f /var/lib/postgresql/repmgr.conf --verbose standby promote
```

## Tell slaves to sync against the new master (`repmgr standby follow`)

All nodes (master and slaves) know each other. When calling `repmgr standby follow` is forced upon the slaves, they ask around (via SQL) to determine who the new master is. This is done by calling `pg_is_in_recovery()`, which is `false` on a master. This step recreates the `recovery.conf` file, which lists the IP of the new master. 

```
$ sudo postgres    / su - postgres
$ repmgr -f /var/lib/postgresql/repmgr.conf --verbose standby follow
```

## Turn old master into new slave (`repmgr standby clone`)

1. Stop PostgreSQL
2. Enable `all/all` in `pg_hba.conf` again
3. Clone from new master (`repmgr standby clone`)
4. Start PostgreSQL 
5. Hook up to synchronisation (`repmgr standby register`)

Make a `repmgr standby clone` against a previous slave, who became master

```console		
$ sudo postgres    / su - postgres

$ vim /etc/postgresql/9.3/main/pg_hba.conf

$ service postgresql stop

$ repmgr -d repmgr \
	-U repl \
	-R postgres -w 500 \
	-D /var/lib/postgres/9.3/main \
	-f /var/lib/postgresql/repmgr.conf \
	--verbose \
	standby clone 10.10.0.5

$ service postgresql start

$ repmgr -f /var/lib/postgresql/repmgr.conf \
	--verbose standby register
```

# pgbouncer

## /etc/pgbouncer/pgbouncer.ini contents

Assumptions: 

- pgbouncer service is on 10.10.0.20 (and similarly on other boxes), behind an internal load balancer
- current master is 10.10.0.5
- current slaves are 10.10.0.6 and 10.10.0.7
- The application logic uses two different endpoints for updates (writes) and pure queries (reads).


```ini
[databases]
myapp-write    = host=primary  port=5433
myapp-readonly = port=5434

[pgbouncer]
listen_addr = 10.10.0.20, 127.0.0.1
listen_port = 5432
pool_mode = transaction
max_client_conn = 500
default_pool_size = 20
```

## Configure internal load balancer

```powershell
Add-AzureAccount
Set-AzureSubscription -SubscriptionName "BizSpark Plus" -SubscriptionId "8eefc6f2-7216-4aef-8394-fce57df325a3"
Select-AzureSubscription -SubscriptionName "BizSpark Plus" -Default

Add-AzureInternalLoadBalancer -InternalLoadBalancerName pgbouncer -ServiceName fantasyweb -SubnetName fantasy -StaticVNetIPAddress 10.10.0.100

Get-AzureVM -ServiceName fantasyweb -Name pooler1 | Add-AzureEndpoint -Name "pgbouncer" -LBSetName "pgbouncer" -Protocol tcp -LocalPort 5432 -PublicPort 5432 -ProbePort 5432 -ProbeProtocol tcp -ProbeIntervalInSeconds 10 -InternalLoadBalancerName pgbouncer | Update-AzureVM

```





















# PostgreSQL admin stuff

```console
$ createuser application_admin

# -O owner
$ createdb -O application_admin my_database

# Add application_admin to administrators in /etc/postgresql/9.3/main/pg_hba.conf
$ vim  /etc/postgresql/9.3/main/pg_hba.conf

# -s n scaling factor
# -i initialization
# -U username
% pgbench -s 10 -i -U application_admin my_database

$ service postgresql start
```

# pg_control.rb

```

if (stop) {
	var isSlave = `SELECT pg_is_in_recovery()`;
	var isMaster = ! isSlave;
	if (isMaster) {
		// refuse new connections
		modify("pg_hba.conf") && sqleval("localhost", "SELECT pg_reloadconf();");

		// drop existing connections
		sqleval("localhost", "SELECT pg_terminate_backend(pid) \
			FROM pg_stat_activity \
			WHERE usename='webfrontend';");

		// create checkoint 
		sqleval("localhost", "CHECKPOINT;");

		// determine current location
		var flush_location = sqleval("localhost", "SELECT pg_current_xlog_location();");

		string diffStatement = "pg_xlog_location_diff(pg_current_xloc_location(), " + flush_location + ") from pg_stat_replication;"
		string determineReplicationLag = "SELECT client_addr, pg_current_xlog_location()," + diffStatement;

		bool allSlavesSynced = false;
		while (!allSlavesSynced) {
			bool foundUnsyncedSlave = false;
			var replicationStates = sqleval("localhost", determineReplicationLag);
			foreach (var replicationState in replicationStates) {
				(client,current,diff) = replicationState;
				if (diff > 0) {
					foundUnsyncedSlave = true;
				}
			}			
			allSlavesSynced = !foundUnsyncedSlave;
		}
	}
}

```








## Determine whether you're on master or slave

`SELECT pg_is_in_recovery()` returns `false` on the master (who is not in recovery), and `true` for slaves (who are in constant recovery mode).

# Enable fiddler to sniff azure-cli 

http://blogs.msdn.com/b/avkashchauhan/archive/2013/01/30/using-fiddler-to-decipher-windows-azure-powershell-or-rest-api-https-traffic.aspx

```console
SET HTTP_PROXY=http://127.0.0.1:8888/
SET HTTPS_PROXY=http://127.0.0.1:8888/
SET HTTPPROXY=http://127.0.0.1:8888/
SET HTTPSPROXY=http://127.0.0.1:8888/
SET NODE_TLS_REJECT_UNAUTHORIZED=0
```

## CustomRules.js

```javascript
static function OnBeforeRequest(oSession: Session) {
	oSession["https-Client-Certificate"]= "C:\\Users\\chgeuer\\Desktop\\txxx.cer"; 
```

# Questions:

- For the block device driver for Azure Linux IaaS, what's supported or optimal? open_datasync, fdatasync (default on Linux), fsync, fsync_writethrough, open_sync. This is relevant to configure wal_sync_method
- It seems that the guest OS does not see a detached data disk. An attached disk shows up in dmesg, while a detach process doesn't show up. When trying to open a formerly attached device (with cfdisk), the 

# Arbitrary Unix vodoo :-)

## See what's happening 

```console
watch dmesg  \| tail -5
```

# References

- [Azure command-line tool for Mac and Linux](http://azure.microsoft.com/en-us/documentation/articles/command-line-tools/)
- [Exporting and Importing VM settings with the Azure Command-Line Tools](http://blogs.msdn.com/b/silverlining/archive/2012/10/25/exporting-and-importing-vm-settings-with-the-azure-command-line-tools.aspx)
- [Create Virtual Machine Deployment REST API](http://msdn.microsoft.com/en-us/library/azure/jj157194.aspx)
- [Linux and Graceful Shutdowns](http://azure.microsoft.com/blog/2014/05/06/linux-and-graceful-shutdowns-2/)
- [Internal Load Balancing](http://azure.microsoft.com/blog/2014/05/20/internal-load-balancing/)
- [How to Use Service Bus Topics/Subscriptions from Ruby](https://github.com/Azure/azure-content/blob/master/articles/service-bus-ruby-how-to-use-topics-subscriptions.md)
