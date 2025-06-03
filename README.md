# Oracle Data Guard Fast-Start Failover (FSFO) Configuration Guide

Panduan lengkap konfigurasi FSFO Oracle Data Guard, mulai dari instalasi observer, konfigurasi flashback, broker, hingga failover otomatis.

---

## Prasyarat

- Oracle Database 12c/19c atau lebih baru
- Data Guard Configuration sudah aktif dan berjalan (Primary dan Standby)
- Oracle Net Listener sudah berjalan dan listener service sudah dibuat
- TNS names sudah diset pada observer dan server database
- Oracle Database Client (untuk observer) terinstall jika observer dijalankan di mesin terpisah

---

## Langkah 1: Aktifkan Flashback Database pada Primary dan Standby

FSFO memerlukan fitur Flashback Database aktif untuk dapat melakukan automatic failover dan reinstate.

### Pada Primary Database

1. Pastikan database dalam mode ARCHIVELOG

```sql
SQL> archive log list;
```

2. Aktifkan Flashback

```sql
SQL> shutdown immediate;
SQL> startup mount;
SQL> alter database flashback on;
SQL> alter database open;
```

3. Buat flashback recovery area (jika belum ada)

```sql
SQL> show parameter db_recovery_file_dest;

-- Jika belum ada:
SQL> alter system set db_recovery_file_dest='/u01/app/oracle/fast_recovery_area' scope=both;
SQL> alter system set db_recovery_file_dest_size=10G scope=both;
```

### Pada Standby Database

Lakukan hal yang sama seperti di primary:

```sql
SQL> shutdown immediate;
SQL> startup mount;
SQL> alter database flashback on;
SQL> alter database open read only;
```

---

## Langkah 2: Konfigurasi Data Guard Broker

1. Pastikan broker diaktifkan di `init.ora` atau `spfile` pada primary dan standby

```sql
SQL> show parameter dg_broker_start;

-- Jika OFF, aktifkan dengan:
SQL> alter system set dg_broker_start=true scope=both;
```

2. Buat file konfigurasi broker (biasanya di `$ORACLE_HOME/network/admin`)

Contoh parameter listener dan service sudah benar untuk primary dan standby.

3. Connect ke broker via DGMGRL di primary atau standby

```bash
$ dgmgrl sys@primary_db
DGMGRL> show configuration;
```

---

## Langkah 3: Aktifkan Fast-Start Failover

Jalankan perintah berikut di DGMGRL:

```bash
DGMGRL> enable fast_start failover;
```

Jika ada error:

```
Warning: ORA-16827: Flashback Database is disabled
```

Pastikan flashback sudah aktif (lihat Langkah 1).

---

## Langkah 4: Setup Observer

Observer adalah proses yang berjalan di host terpisah untuk memonitor primary dan standby.

1. Pastikan observer berjalan pada host yang berbeda (contoh: Availability Domain lain)

2. Install Oracle Database Client di host observer, agar ada `dgmgrl`

3. Pastikan file `tnsnames.ora` di observer sudah ada entry untuk primary dan standby

Contoh `tnsnames.ora`:

```
mydb_prd =
  (DESCRIPTION=
    (ADDRESS=(PROTOCOL=TCP)(HOST=primary_host)(PORT=1521))
    (CONNECT_DATA=(SERVICE_NAME=mydb_prd))
  )

mydb_stb =
  (DESCRIPTION=
    (ADDRESS=(PROTOCOL=TCP)(HOST=standby_host)(PORT=1521))
    (CONNECT_DATA=(SERVICE_NAME=mydb_stb))
  )
```

4. Jalankan observer via DGMGRL

```bash
$ dgmgrl sys@primary_db
DGMGRL> start observer;
```

Jika berhasil, output akan mirip:

```
Observer 'observer-dg-broker' started
```

---

## Langkah 5: Verifikasi FSFO

Di DGMGRL, cek status FSFO:

```bash
DGMGRL> show fast_start failover;
```

Contoh output:

```
Fast-Start Failover: Enabled in Potential Data Loss Mode
Protection Mode: MaxPerformance
Active Target: mydb_stb
Observer: observer-dg-broker
Shutdown Primary: TRUE
Auto-reinstate: TRUE
```

---

## Langkah 6: Simulasi Failover Manual

Anda dapat memicu failover secara manual dari standby:

```bash
DGMGRL> failover to mydb_stb;
```

Pastikan failover berhasil dan primary lama otomatis direinstate sebagai standby.

---

## Troubleshooting

- **ORA-12154 TNS:could not resolve the connect identifier specified**  
  Pastikan `tnsnames.ora` sudah benar dan environment variabel `TNS_ADMIN` diarahkan ke lokasi yang benar.

- **ORA-16827 Flashback Database is disabled**  
  Aktifkan flashback database pada primary dan standby.

- Observer gagal connect ke primary atau standby  
  Cek listener status, firewall, dan akses jaringan.

---

## Referensi

- Oracle Data Guard Concepts and Administration Guide  
- Oracle Cloud Infrastructure Blog (FSFO Post)  
- Oracle Support Notes

---

# Selesai

File ini dapat digunakan sebagai panduan lengkap untuk konfigurasi Fast-Start Failover di Oracle Data Guard.

