PostgreSQL Disaster Recovery, PITR & Streaming Replication LabA comprehensive guide to PostgreSQL administration on Ubuntu, covering logical backups, Write-Ahead Log (WAL) archiving, Point-in-Time Recovery (PITR), and live streaming standby replication.1. Environment TopologyOperating System: Ubuntu LinuxPostgreSQL Version: 18Database Name: bootcampLinux User / Role: PrezzoKuok (Password: prezzo211)Replication Role: replicator (Password: reppass)Primary Instance: Port 5432Standby Instance: Port 5433 (Data: ~/standby/, Sockets: ~/standby/socket/)Directories:Logical/Physical Backups: ~/backups/base/WAL Archives: ~/backups/wal/2. Step-by-Step Lab ExecutionStep 1: Logical Backups & RestorationCreate a compressed custom-format backup of the bootcamp database and restore it to a verification database.Create backup folders:mkdir -p ~/backups/base ~/backups/wal
Take a compressed logical dump:pg_dump -Fc bootcamp > ~/backups/bootcamp.dump
Verify restoration to bootcamp_check:createdb bootcamp_check
pg_restore -d bootcamp_check ~/backups/bootcamp.dump
Step 2: WAL Archiving & Physical Base BackupConfigure continuous WAL archiving to record all database changes for point-in-time restoration.Edit /etc/postgresql/18/main/postgresql.conf:wal_level = replica
archive_mode = on
archive_command = 'cp %p /home/PrezzoKuok/backups/wal/%f'
Restart PostgreSQL:sudo systemctl restart postgresql
Perform a physical base backup:pg_basebackup -U postgres -D ~/backups/base -F p -P -z
Step 3: Simulated Data Loss & Point-in-Time Recovery (PITR)Simulate accidental table deletion and restore the database state right before the disaster.Simulate Data Loss:DELETE FROM students;
Perform Recovery:Stop PostgreSQL: sudo systemctl stop postgresqlReplace active data directory contents with ~/backups/base/Touch recovery.signal in the data folder.Append recovery targets in postgresql.auto.conf:restore_command = 'cp /home/PrezzoKuok/backups/wal/%f %p'
recovery_target_time = 'YYYY-MM-DD HH:MM:SS'
Start PostgreSQL to complete recovery.Step 4: Setting Up Streaming Standby ReplicationConfigure a live read-only standby server on port 5433.Configure Replication Permissions (Primary):
Add to /etc/postgresql/18/main/pg_hba.conf:host replication replicator 127.0.0.1/32 md5
Reload PostgreSQL: sudo systemctl reload postgresqlStream Primary Data to Standby Directory:pg_basebackup -h 127.0.0.1 -U replicator -D ~/standby -R -P
Configure Standby Settings:Create local socket folder:mkdir -p ~/standby/socket
Ensure pg_hba.conf exists in ~/standby/:sudo cp /etc/postgresql/18/main/pg_hba.conf ~/standby/
sudo chown PrezzoKuok:PrezzoKuok ~/standby/pg_hba.conf
Configure ~/standby/postgresql.conf:port = 5433
unix_socket_directories = '/home/PrezzoKuok/standby/socket'
Start Standby Instance:/usr/lib/postgresql/18/bin/pg_ctl -D ~/standby -l ~/standby/logfile start
Step 5: Monitoring & VerificationCheck Replication Health on Primary:SELECT application_name, state, 
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes 
FROM pg_stat_replication;
Verify Data Synchronization on Standby:Insert data on primary:sudo -u postgres psql -d bootcamp -c "INSERT INTO students (name) VALUES ('Test Student');"
Query standby replica:psql -h /home/PrezzoKuok/standby/socket -p 5433 -d bootcamp -c "SELECT * FROM students;"
Verify recovery mode state on standby:SELECT pg_is_in_recovery(); -- Returns 't'
3. Challenges Encountered & Technical SolutionsChallenge / ErrorRoot CauseResolutionEmpty postgresql.conf in NanoFile created prior to backup completion or edited in wrong directory.Populated port = 5433 directly or re-ran pg_basebackup.FATAL: password authentication failed for user "replicator"Password mismatch or un-reloaded pg_hba.conf rules.Re-set password: ALTER ROLE replicator WITH PASSWORD 'reppass'; and updated pg_hba.conf to md5.pg_ctl: command not foundBinary directory /usr/lib/postgresql/18/bin not in default user $PATH.Invoked full binary path: /usr/lib/postgresql/18/bin/pg_ctl.FATAL: could not create lock file "/var/run/postgresql/.s.PGSQL.5433.lock": Permission deniedNon-root user PrezzoKuok lacks write access to /var/run/postgresql/.Created local folder ~/standby/socket and set unix_socket_directories = '/home/PrezzoKuok/standby/socket'.FATAL: could not load /home/PrezzoKuok/standby/pg_hba.confConfig files excluded during raw directory streaming.Copied pg_hba.conf manually to ~/standby/ and adjusted ownership using chown.
