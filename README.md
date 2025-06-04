# Oracle Data Guard Fast-Start Failover (FSFO) Configuration Guide

Panduan lengkap konfigurasi FSFO Oracle Data Guard, mulai dari instalasi observer, konfigurasi flashback, broker, hingga failover otomatis.

---

## Primary (node1):

- Hostname: zubair – IP: 192.168.30.60
- ORACLE_UNQNAME: mydb
- ORACLE_SID: mydb

## Standby (node2):

- Hostname: zubair-standby – IP: 192.168.30.61
- ORACLE_UNQNAME: mydb_stb
- ORACLE_SID: mydb

## Observer

- Hostname: observer-dg-broker – IP: 192.168.30.62
- Sudah menggunakan Data Guard Broker (dgmgrl)
- Listener port: 1528
- TNS yang akan digunakan Observer (mydb_prd dan mydb_stb)

```
mydb_prd =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = zubair)(PORT = 1528))
    )
    (CONNECT_DATA =
      (SID = mydb)
    )
  )

mydb_stb =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = zubair-standby)(PORT = 1528))
    )
    (CONNECT_DATA =
      (SID = mydb)
    )
  )
```
 

---

## Langkah 1: Pastikan konfigurasi broker sudah aktif di kedua DB:
- harus bernilai TRUE
```sql
SHOW PARAMETER dg_broker_start;
```

## Langkah 2: Aktifkan Observer dan FSFO
```sql
ENABLE FAST_START FAILOVER;
```
lalu cek:
```sql
show configuration;
SHOW DATABASE VERBOSE mydb;
SHOW DATABASE VERBOSE mydb_stb;
```

## Langkah 3:🔐 Aktifkan Fast-Start Failover

#### A. Aktifkan Flashback pada primary dan standby
Pada Primary Masuk ke sqlplus
```sql
show parameter recover
SELECT flashback_on FROM v$database;
ALTER SYSTEM SET db_recovery_file_dest='/oracle/product/orabase/fast_recovery_area' SCOPE=BOTH;
ALTER SYSTEM SET db_recovery_file_dest_size=10G SCOPE=BOTH;
ALTER DATABASE FLASHBACK ON;
SELECT flashback_on FROM v$database;
```
Pada Standby Masuk ke sqlplus
```sql
show parameter recover
SELECT flashback_on FROM v$database;
ALTER SYSTEM SET db_recovery_file_dest='/oracle/product/orabase/fast_recovery_area' SCOPE=BOTH;
ALTER SYSTEM SET db_recovery_file_dest_size=10G SCOPE=BOTH;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE FLASHBACK ON;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
SELECT flashback_on FROM v$database;
```

#### B. Aktifkan Observer dan FSFO
Masuk ke dgmgrl dari salah satu node:
```
dgmgrl sys@mydb_prd
ENABLE FAST_START FAILOVER;
```
output : 
```
DGMGRL> ENABLE FAST_START FAILOVER;
Enabled in Potential Data Loss Mode.
DGMGRL>
```
Status "Enabled in Potential Data Loss Mode" berarti Fast-Start Failover (FSFO) aktif, tapi masih dalam mode proteksi MaxPerformance (ASYNC) — bukan MaxAvailability (SYNC)


## Langkah 4: Jalankan Observer di Node Terpisah (192.168.30.62)

#### A. Siapkan environment di observer:
- Install oracle-database-preinstall-19c
yum install -y oracle-database-preinstall-19c
- Download LINUX.X64_193000_client pada link berikut :
```bash
https://www.oracle.com/id/database/technologies/oracle19c-linux-downloads.html
```

- Set profile
```bash
export PATH
export ORACLE_HOSTNAME=observer-dg-broker
export ORACLE_BASE=/oracle/app/oracle/
export ORACLE_HOME=/oracle/app/oracle/product/19.0.0/client_1
export ORA_INVENTORY=/oracle/app/oraInventory
export PATH=/usr/sbin:/usr/local/bin:$PATH
export PATH=$ORACLE_HOME/bin:$PATH
export TNS_ADMIN=/oracle/opc/network
```
- Create directories
```bash
mkdir -p /oracle/app/oracle/
mkdir -p /oracle/app/oracle/product/19.0.0/client_1
mkdir -p /oracle/app/oraInventory
mkdir -p /oracle/opc/network
chown -R oracle:dba /oracle/
chmod -R 775 /oracle/
```
- Set /etc/hosts
```bash
192.168.30.60 zubair
192.168.30.61 zubair-standby
192.168.30.62 observer-dg-broker
```
- install & unzip Oracle Client LINUX.X64_193000_client.zip
- add TNS dan cek konektivitas
```bash
cat /oracle/opc/network/tnsnames.ora
mydb_prd =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = zubair)(PORT = 1528))
    )
    (CONNECT_DATA =
      (SID = mydb)
    )
  )

mydb_stb =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = zubair-standby)(PORT = 1528))
    )
    (CONNECT_DATA =
      (SID = mydb)
    )
  )

tnsping mydb_prd
tnsping mydb_stb
```

#### B. Start Observer :
masuk dgmgrl
```bash
connect sys@mydb_prd
start observer
```

- verifikasi observer
buka new session pada node-observer masuk dgmgrl
```sql
DGMGRL> connect sys@mydb_prd
Password:
Connected to "mydb"
Connected as SYSDBA.
DGMGRL> show observer

Configuration - my_dg_config

  Primary:            mydb
  Active Target:      mydb_stb

Observer "observer-dg-broker" - Master

  Host Name:                    observer-dg-broker
  Last Ping to Primary:         2 seconds ago
  Last Ping to Target:          2 seconds ago

DGMGRL> SHOW FAST_START FAILOVER;

Fast-Start Failover: Enabled in Potential Data Loss Mode

  Protection Mode:    MaxPerformance
  Lag Limit:          30 seconds

  Threshold:          30 seconds
  Active Target:      mydb_stb
  Potential Targets:  "mydb_stb"
    mydb_stb   valid
  Observer:           observer-dg-broker
  Shutdown Primary:   TRUE
  Auto-reinstate:     TRUE
  Observer Reconnect: (none)
  Observer Override:  FALSE

Configurable Failover Conditions
  Health Conditions:
    Corrupted Controlfile          YES
    Corrupted Dictionary           YES
    Inaccessible Logfile            NO
    Stuck Archiver                  NO
    Datafile Write Errors          YES

  Oracle Error Conditions:
    (none)

DGMGRL>
```

Observer	          ✔ Aktif dan jalan sebagai Master
Fast-Start Failover	✔ Sudah Enabled
Flashback Database	✔ Sudah aktif di Primary dan Standby


## Langkah 5: Failover
#### A. Kill database primary (192.168.30.60 zubair)
Masuk sqlplus 
```sql
shutdown abort;
```

Log Observer 
```sql
DGMGRL> start observer
[W000 2025-06-04T23:45:35.617+07:00] FSFO target standby is mydb_stb
Observer 'observer-dg-broker' started
[W000 2025-06-04T23:45:35.731+07:00] Observer trace level is set to USER
[W000 2025-06-04T23:45:35.731+07:00] Try to connect to the primary.
[W000 2025-06-04T23:45:35.731+07:00] Try to connect to the primary mydb_prd.
[W000 2025-06-04T23:45:35.743+07:00] The standby mydb_stb is ready to be a FSFO target
[W000 2025-06-04T23:45:36.743+07:00] Connection to the primary restored!
[W000 2025-06-04T23:45:38.743+07:00] Disconnecting from database mydb_prd.
[W000 2025-06-05T00:07:17.281+07:00] Primary database cannot be reached.
[W000 2025-06-05T00:07:17.281+07:00] Fast-Start Failover threshold has not exceeded. Retry for the next 30 seconds
[W000 2025-06-05T00:07:18.282+07:00] Try to connect to the primary.
[W000 2025-06-05T00:07:19.367+07:00] Primary database cannot be reached.
[W000 2025-06-05T00:07:20.367+07:00] Try to connect to the primary.
[W000 2025-06-05T00:07:44.223+07:00] Primary database cannot be reached.
[W000 2025-06-05T00:07:44.223+07:00] Fast-Start Failover threshold has not exceeded. Retry for the next 3 seconds
[W000 2025-06-05T00:07:45.224+07:00] Try to connect to the primary.
[W000 2025-06-05T00:07:46.295+07:00] Primary database cannot be reached.
[W000 2025-06-05T00:07:46.295+07:00] Fast-Start Failover threshold has not exceeded. Retry for the next 1 second
[W000 2025-06-05T00:07:47.296+07:00] Try to connect to the primary.
[W000 2025-06-05T00:07:48.357+07:00] Primary database cannot be reached.
[W000 2025-06-05T00:07:48.357+07:00] Fast-Start Failover threshold has expired.
[W000 2025-06-05T00:07:48.357+07:00] Try to connect to the standby.
[W000 2025-06-05T00:07:48.357+07:00] Making a last connection attempt to primary database before proceeding with Fast-Start Failover.
[W000 2025-06-05T00:07:48.357+07:00] Check if the standby is ready for failover.
[S002 2025-06-05T00:07:48.365+07:00] Fast-Start Failover started...

2025-06-05T00:07:48.365+07:00
Initiating Fast-Start Failover to database "mydb_stb"...
[S002 2025-06-05T00:07:48.365+07:00] Initiating Fast-start Failover.
Performing failover NOW, please wait...
Failover succeeded, new primary is "mydb_stb"
2025-06-05T00:08:00.601+07:00
[S002 2025-06-05T00:08:00.601+07:00] Fast-Start Failover finished...
[W000 2025-06-05T00:08:00.601+07:00] Failover succeeded. Restart pinging.
[W000 2025-06-05T00:08:00.623+07:00] Primary database has changed to mydb_stb.
[W000 2025-06-05T00:08:00.626+07:00] Try to connect to the primary.
[W000 2025-06-05T00:08:00.626+07:00] Try to connect to the primary mydb_stb.

```

#### B. Failover Berhasil dan cek status New Standby Database
Masuk sqlplus
```sql
select name, open_mode, database_role from v$database;
show pdbs;
```

## Jika ingin Pergunakan Primary Lama menjadi standby (REINSTATE)

- Pada Primary Old
Masuk sqlplus :
```sql
STARTUP MOUNT;
```

- ke Observer dan connect ke standby (New Primary)
masuk dgmgrl : 
```sql
DGMGRL> connect sys@mydb_stb
Password:
Connected to "MYDB_STB"
Connected as SYSDBA.
DGMGRL> SHOW CONFIGURATION;

Configuration - my_dg_config

  Protection Mode: MaxPerformance
  Members:
  mydb_stb - Primary database
    Warning: ORA-16824: multiple warnings, including fast-start failover-related warnings, detected for the database

    mydb     - (*) Physical standby database (disabled)
      ORA-16661: the standby database needs to be reinstated

Fast-Start Failover: Enabled in Potential Data Loss Mode

Configuration Status:
WARNING   (status updated 5 seconds ago)

DGMGRL> REINSTATE DATABASE mydb;
Reinstating database "mydb", please wait...
Reinstatement of database "mydb" succeeded
DGMGRL>
```

- Cek Status New Standby
```sql
SQL> select name, open_mode, database_role from v$database;

NAME      OPEN_MODE            DATABASE_ROLE
--------- -------------------- ----------------
MYDB      MOUNTED              PHYSICAL STANDBY

SQL> show pdbs

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       MOUNTED
         3 JAKDB                          MOUNTED
         4 PDB1                           MOUNTED
         5 PDB2                           MOUNTED
SQL> !hostname
zubair
```
#### Primary Lama telah menjadi standby setelah failover

#### Log Observer
```bash
2025-06-05T00:18:21.022+07:00
[W000 2025-06-05T00:18:22.017+07:00] Successfully reinstated database mydb.
[W000 2025-06-05T00:18:24.022+07:00] The standby mydb needs to be reinstated
[W000 2025-06-05T00:18:24.023+07:00] Try to connect to the new standby mydb.
[W000 2025-06-05T00:18:27.023+07:00] Try to connect to the primary mydb_stb.
[W000 2025-06-05T00:18:28.030+07:00] Connection to the primary restored!
[W000 2025-06-05T00:18:29.031+07:00] Wait for new primary to be ready to reinstate.
[W000 2025-06-05T00:18:30.031+07:00] New primary is now ready to reinstate.
[W000 2025-06-05T00:18:30.031+07:00] Issuing REINSTATE command.
```
