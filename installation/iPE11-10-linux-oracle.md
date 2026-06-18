# TIBCO iProcess Engine 11.10.0 — Installation Guide
**Target:** CentOS Stream 9 (64-bit) | Oracle 19 | TIBCO EMS

---

## Overview

TIBCO iProcess Engine consists of a set of background server processes that run on a UNIX host and connect to an Oracle database. The installer (`swinstall`) is a console-mode wizard that creates the database schema, installs the software into a directory (`$SWDIR`), and registers the node with the operating system.

### Key concepts

| Term | Meaning |
|---|---|
| `$SWDIR` | The iProcess Engine installation directory (e.g. `/opt/tibco/iprocess/<NODE_NAME>`) |
| **Background user** | The Linux account that owns and runs the iProcess processes (e.g. `staffpro`) |
| **Admin user** | The Linux account used to administer iProcess (can be same as background user) |
| **iProcess group** | Linux group to which all iProcess users belong (e.g. `staffgrp`) |
| **Schema owner** | Oracle user that owns the iProcess schema tables (e.g. `swpro`) |
| **DB user** | Oracle user that iProcess uses for runtime read access (e.g. `swuser`) |
| **Node name** | Unique name for this iProcess Engine instance (e.g. `node001`) |

## Index

1. [Part 1 — Prerequisites](#part-1--prerequisites)
2. [Part 2 — Pre-Installation Tasks](#part-2--pre-installation-tasks)
3. [Part 3 — Oracle Database Preparation](#part-3--oracle-database-preparation)
4. [Part 4 — Run the Installer (swinstall)](#part-4--run-the-installer-swinstall)
5. [Part 5 — Post-Installation Tasks](#part-5--post-installation-tasks)
6. [Part 6 — Start iProcess Engine](#part-6--start-iprocess-engine)
7. [Quick Reference — Values for Your Installation](#quick-reference--values-for-your-installation)
8. [Important Notes](#important-notes)

---

## Part 1 — Prerequisites

### System requirements (confirmed for your platform)

- **OS:** CentOS Stream 9.x (64-bit) ✓
- **Database:** Oracle 19 ✓
- **Disk space (installation set extract):** ~697 MB + ~1 GB for `$SWDIR` + Oracle tablespace
- **Oracle Client:** Must be installed on the iProcess Engine host to connect remotely to Oracle 19 at `<ORACLE_DB_HOST_OR_IP>`
- **Korn Shell (KSH):** Must be installed — note: `pdksh` (public domain ksh) is **not** supported; use `ksh` from the OS packages
- **Java:** Java 17 is bundled and installed into `$SWDIR/java` automatically
- **JMS:** TIBCO EMS 8.7.0 or later required for activity publishing

### Required information before starting

| Parameter | Your Value |
|---|---|
| Oracle server host/IP | `<ORACLE_DB_HOST_OR_IP>` |
| Oracle DBA username | `ipeuser` |
| Oracle DBA password | `ipeuser` |
| Oracle TNS identifier | *(your Oracle service name / TNS alias — e.g. `<ORACLE_TNS_ALIAS>`)* |
| EMS server host/IP | `<EMS_HOST_OR_IP>` |
| EMS user | `admin` |
| EMS password | `admin` |
| iProcess node name | *(choose, e.g. `<NODE_NAME>`)* |
| `$SWDIR` path | *(choose, e.g. `/opt/tibco/iprocess/<NODE_NAME>`)* |
| Background Linux user | *(choose, e.g. `staffpro`)* |
| iProcess group | *(choose, e.g. `staffgrp`)* |
| Schema owner Oracle user | *(choose, e.g. `swpro`)* |
| DB runtime Oracle user | *(choose, e.g. `swuser`)* |

**Reference placeholders used in this document**

- `<ORACLE_DB_HOST_OR_IP>`: Oracle database server hostname or IP address
- `<EMS_HOST_OR_IP>`: EMS server hostname or IP address
- `<ORACLE_TNS_ALIAS>`: Oracle TNS alias configured in `tnsnames.ora`
- `<ORACLE_SERVICE_NAME>`: Oracle service name used by the TNS alias
- `<NODE_NAME>`: iProcess node name used in `$SWDIR` paths and server naming

> **Note:** You will need to know your Oracle TNS alias (the name registered in `tnsnames.ora` on the iProcess host that resolves to `<ORACLE_DB_HOST_OR_IP>`). If it is a remote Oracle instance, you must configure TNS connectivity first.

---

## Part 2 — Pre-Installation Tasks

### Step 1 — Install required OS packages

```bash
# Install Korn Shell (must be ksh, not pdksh)
sudo dnf install -y ksh

# Installer prerequisite services (CentOS/RHEL equivalent of rpcbind + nfs-common)
sudo dnf install -y rpcbind nfs-utils
sudo systemctl enable --now rpcbind

# Verify
ksh --version
rpcinfo -p localhost

# Install Oracle Instant Client or full Oracle Client (to connect to remote Oracle 19)
# Download oracle-instantclient-basic, oracle-instantclient-sqlplus
# from Oracle's website for RHEL 9 / x86_64, then install:
sudo dnf install -y oracle-instantclient19.*.x86_64  # adjust version
```

### Step 2 — Create Linux users and groups

```bash
# Log in as root
sudo su -

# Create the iProcess group
groupadd staffgrp

# Create the iProcess Background user (owns and runs iProcess processes)
useradd -g staffgrp -m -s /bin/ksh staffpro
passwd staffpro          # set a password

# Create the iProcess Admin user (can be the same as the background user)
# If same user, skip this. If separate:
# useradd -g staffgrp -m staffadm
# passwd staffadm

# Verify group membership
id staffpro
```

### Step 3 — Configure Oracle environment variables

Add the following to `/home/staffpro/.kshrc`:

```bash
# Oracle environment (for Oracle Instant Client)
export ORACLE_HOME=/usr/lib/oracle/19.31/client64   # adjust to your Instant Client path
export ORACLE_SID=DUMMY                             # dummy value when using TNS/remote Oracle
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH
export PATH=$ORACLE_HOME/bin:$PATH

# iProcess installation directory
export SWDIR=/opt/tibco/iprocess/<NODE_NAME>
export PATH=$SWDIR/bin:$PATH

# set nice ksh prompt
 sudo echo "export PS1='${LOGNAME}@$(hostname):${PWD} $ '" >> /etc/.kshrc
```

Also add the same variables to the admin user's profile if separate.

```bash
# Apply immediately
source /home/staffpro/.kshrc
```

### Step 4 — Configure Oracle TNS connectivity

On the iProcess Engine host, create or edit `/etc/oracle/network/admin/tnsnames.ora` (adjust path for Instant Client):

```
<ORACLE_TNS_ALIAS> =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <ORACLE_DB_HOST_OR_IP>)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = <ORACLE_SERVICE_NAME>)   # replace with your actual Oracle service name
    )
  )
```

Test connectivity:

```bash
su - staffpro
sqlplus ipeuser/ipeuser@<ORACLE_TNS_ALIAS>
# Should connect successfully
exit
```

If `ipeuser` is not created yet, continue with the next step first.

### Step 5 — Create Oracle DBA user (ipeuser)

Connect as `SYS` and run:

```sql
-- If this is a CDB, switch to your application PDB first
SHOW PDBS;
ALTER SESSION SET CONTAINER = <YOUR_PDB_NAME>;

-- Create iProcess DBA user
CREATE USER ipeuser IDENTIFIED BY "ipeuser"
  DEFAULT TABLESPACE USERS
  TEMPORARY TABLESPACE TEMP
  PROFILE DEFAULT
  ACCOUNT UNLOCK;

-- Required privileges for installer and setup checks
GRANT CREATE SESSION TO ipeuser;
GRANT DBA TO ipeuser;
GRANT UNLIMITED TABLESPACE TO ipeuser;
GRANT SELECT_CATALOG_ROLE TO ipeuser;
GRANT EXECUTE_CATALOG_ROLE TO ipeuser;
GRANT AQ_ADMINISTRATOR_ROLE TO ipeuser;
GRANT AQ_USER_ROLE TO ipeuser;
GRANT EXECUTE ON sys.dbms_aqadm TO ipeuser WITH GRANT OPTION;
GRANT EXECUTE ON sys.dbms_aq TO ipeuser WITH GRANT OPTION;

-- Verify user exists and is open
SELECT username, account_status, default_tablespace, temporary_tablespace
FROM dba_users
WHERE username = 'IPEUSER';

-- Verify grants
SELECT granted_role
FROM dba_role_privs
WHERE grantee = 'IPEUSER'
ORDER BY granted_role;

SELECT privilege
FROM dba_sys_privs
WHERE grantee = 'IPEUSER'
ORDER BY privilege;
```

### Step 6 — Configure Oracle database settings

Connect to Oracle as a DBA and verify/set required settings:

```sql
-- Connect to Oracle as ipeuser (DBA rights) on <ORACLE_DB_HOST_OR_IP>
sqlplus ipeuser/ipeuser@<ORACLE_TNS_ALIAS>

-- Verify OPEN_CURSORS (must be >= 200)
SELECT value FROM v$parameter WHERE name = 'open_cursors';
-- If < 200:
ALTER SYSTEM SET open_cursors=500 SCOPE=BOTH;

-- Verify an UNDO tablespace exists
SELECT tablespace_name, contents FROM dba_tablespaces WHERE contents = 'UNDO';

-- Disable Oracle Flashback Query (required by iProcess)
-- Check if flashback is enabled:
SELECT flashback_on FROM v$database;
-- If 'YES', disable it:
ALTER DATABASE FLASHBACK OFF;

-- Verify Advanced Queuing (AQ) is installed (required for iProcess)
SELECT * FROM dba_objects 
WHERE object_name = 'DBMS_AQ' AND object_type = 'PACKAGE';

-- Check Oracle character set (UTF-8 recommended)
SELECT value FROM nls_database_parameters WHERE parameter = 'NLS_CHARACTERSET';
```

### Step 7 — Create Oracle tablespaces and accounts (optional if using DBA credentials)

Since `ipeuser` has DBA rights, the installer can create the schema objects automatically. However, you may pre-create tablespaces for better control:

```sql
-- Create data tablespace for iProcess
CREATE TABLESPACE SWDATA
  DATAFILE '/opt/oracle/oradata/tablespaces/ipe/swdata01.dbf' SIZE 500M AUTOEXTEND ON NEXT 100M MAXSIZE 4G
  LOGGING EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;

-- Create temporary tablespace (can reuse TEMP)
-- If you want a dedicated one:
CREATE TEMPORARY TABLESPACE SWTEMP
  TEMPFILE '/opt/oracle/oradata/tablespaces/ipe/swtemp01.dbf' SIZE 200M AUTOEXTEND ON NEXT 50M MAXSIZE 2G;
```

> **Note:** The installer will prompt for tablespace names. You can use `USERS` and `TEMP` (Oracle defaults) if you don't want to create dedicated ones.

### Step 8 — Set file descriptor limits

Add to `/etc/security/limits.conf`:

```
staffpro    soft    nofile    65536
staffpro    hard    nofile    65536
root        soft    nofile    65536
root        hard    nofile    65536
```

Verify after login:

```bash
su - staffpro
ulimit -n     # should show 65536
```

---

## Part 3 — Extract the Installation Set

```bash
# Log in as staffpro (or root)
su - staffpro

# Create a temporary directory for the installer
mkdir -p /tmp/ipe_install
cd /tmp/ipe_install

# Copy the installation TAR file (staffwar.tar.Z) from your distribution media
# Example: copy from a mounted ISO or network share
cp /path/to/media/TIB_ipe-oracle_11.10.0_linux_x86_64.tar.Z .

# Decompress and extract
gzip -d TIB_ipe-oracle_11.10.0_linux_x86_64.tar.Z
tar xvf TIB_ipe-oracle_11.10.0_linux_x86_64.tar

# The extracted directory is your DistDir (e.g. /tmp/ipe_install)
```

### Apply Hotfix 1 (before Part 4)

Apply Hotfix 1 after extracting the main software and before running `swinstall`:

1. Download the hotfix from the TIBCO Support site:
  `TIB_ipe-oracle_11.10.0_HF-001_linux_x86_64.tar.Z`
2. Place the hotfix file in the DistDir (/tmp/ipe_install) iProcess server.
3. Uncompress and extract the hotfix archive.

```bash
# Example: work from a temporary hotfix directory
cd /tmp/ipe_install 

# Place/copy TIB_ipe-oracle_11.10.0_HF-001_linux_x86_64.tar.Z here first
gzip -d TIB_ipe-oracle_11.10.0_HF-001_linux_x86_64.tar.Z
tar -xfv TIB_ipe-oracle_11.10.0_HF-001_linux_x86_64.tar
```
Now hotfix 1 is integration with the base installation packages.
---

## Part 4 — Run the Installer (Console Mode)

### Step 1 — Set the `$SWDIR` variable (optional but recommended)

```bash
# /opt is root-owned, so create the directory once as root and hand it to staffpro
sudo mkdir -p /opt/tibco/iprocess/<NODE_NAME>
sudo chown -R staffpro:staffgrp /opt/tibco

# Then continue as the iProcess background user
su - staffpro
export SWDIR=/opt/tibco/iprocess/<NODE_NAME>
mkdir -p "$SWDIR"
```

### Step 2 — Launch the installer

```bash
cd /tmp/ipe_install
./swinstall
```

> If you get `Please install rpcbind and nfs-common packages...`, on CentOS/RHEL run:
>
> ```bash
> sudo dnf install -y rpcbind nfs-utils
> sudo systemctl enable --now rpcbind
> rpcinfo -p localhost
> ```

### Step 3 — Walk through the installer menus

**License Agreement**

```
Do you accept all the terms of the License Agreement? (Y/N - default N): Y
```

**Installation Directory**

```
In which directory will TIBCO iProcess Engine reside?
Enter the full path name (blank to quit): /opt/tibco/iprocess/<NODE_NAME>
```

> If the directory does not exist, answer `Y` to create it.

**Policy to Install Schema and Files Menu**

For a new installation, select **"Install Schema and Files"** (installs both the OS files and creates the Oracle schema in one run).

---

### Installer Menu Walkthroughs

#### Location, Identification, and OS Accounts Menu

| Prompt | Your Value |
|---|---|
| Node name | `<NODE_NAME>` *(or your chosen name)* |
| iProcess Engine Installation Directory | `/opt/tibco/iprocess/<NODE_NAME>` |
| Background user name | `staffpro` |
| Admin user name | `staffpro` *(or `staffadm` if separate)* |
| iProcess User Group | `staffgrp` |
| Licensee name | *(your company name)* |

---

#### Configuration Options Menu

| Prompt | Recommended Value |
|---|---|
| Enable iProcess Objects Server | `N` *(unless needed)* |
| Enable iProcess Objects Director | `N` *(unless needed)* |
| Enable Activity Publishing (JMS) | `Y` *(enable for EMS integration)* |
| Enable autostart | `Y` *(starts iProcess at system boot)* |
| Enable Case Data Normalization | `N` *(default)* |
| Enable System Event Logging | `Y` |
| Enable Prediction | `N` *(unless licensed)* |

---

#### IAP Configuration Menu *(only shown if Activity Publishing enabled)*

| Prompt | Your Value |
|---|---|
| JMS Provider | `TIBCO EMS` |
| URL for JMS Provider | `tcp://<EMS_HOST_OR_IP>:7222` |
| Context Factory Name | `com.tibco.tibjms.naming.TibjmsInitialContextFactory` |
| Connection Factory Name | `GenericConnectionFactory` |
| Topic Name | `iap.ipe` *(default, can leave as-is)* |
| Security Principal (JMS user) | `admin` |
| Security Credentials (JMS password) | `admin` |

> **Note:** The TIBCO EMS JAR files (`tibjms.jar`, `tibjmsadmin.jar`) must be available on the iProcess host, but do not place them under `$SWDIR/java/lib/ext/`. That legacy extension-directory approach breaks with Java 17. Instead, keep the JARs in a normal directory such as `$SWDIR/java/lib/jms/` or `/opt/tibco/ems/current/lib/`, and make sure `$SWDIR/etc/iapjms.classpath.properties` points to that location.

---

#### Oracle Database Installation Method Menu

Choose based on your preference:

- **"Installer creates and populates the schema"** — easiest; the installer uses your DBA credentials to do everything.

---

#### Oracle Database Connection and Account Details Menu

```
1 ) Oracle DB TNS Identifier                 : <ORACLE_TNS_ALIAS>
2 ) Oracle DB Administrator Name             : ipeuser
3 ) Oracle DB Administrator Password         : ipeuser
4 ) iProcess Engine DB Schema Owner Name     : swpro
5 ) iProcess Engine DB Schema Owner Password : swpro1
6 ) iProcess Engine DB User Name             : swuser
7 ) iProcess Engine DB User Password         : swuser1
8 ) Support Unicode Encoding                 : Y   (if your Oracle DB uses AL32UTF8)
9 ) Data Tablespace Name                     : SWDATA   (or USERS)
10) Temporary Tablespace Name                : SWTEMP   (or TEMP)
```

> `swpro` and `swuser` are Oracle accounts the installer will create. Choose passwords as appropriate.

Enter `C` to confirm and proceed.

---

#### Configuration Summary

Review all settings, then:

```
Ready to install TIBCO iProcess Engine - continue (Y/N - default Y): Y
```

---

#### License Activation

```
Enter TIBCO Activation Service URL: <your-activation-url>
```

Since in-product activation is used, press enter to skip. <br>
The in-product activation will be performed in a later step.

---

#### Installation Verification Test

```
Run the verification test now? (Y/N - default Y): Y
```

Expected output:

```
TIBCO iProcess Engine Nodename (<NODE_NAME>) checked OK.
TIBCO iProcess Engine RPC Number (xxxxxx) checked OK.
TIBCO iProcess Engine service ports checked OK
TIBCO iProcess Engine process entries OK
```

---

## Part 5 — Post-Installation Tasks

### Step 1 — Check for TODO file

```bash
cat $SWDIR/logs/TODO
```

If this file exists and is non-empty, it contains commands that could not be run automatically (usually because root permissions were needed). Execute each command listed.

### Step 2 — Run the rootscript (if listed in TODO)

```bash
sudo su -

# Set environment
export SWDIR=/opt/tibco/iprocess/<NODE_NAME>
export ORACLE_HOME=/usr/lib/oracle/19.x/client64
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH

# Run rootscript as root
$SWDIR/logs/rootscript
```

### Step 3 — Run swpostinst

```bash
su - staffpro
$SWDIR/util/swpostinst
```

### Step 4 — Copy EMS client JARs

Copy the EMS client JAR files from your EMS server to the iProcess host:

```bash
# On the EMS server (<EMS_HOST_OR_IP>), locate JARs:
# /opt/tibco/ems/<version>/lib/tibjms.jar
# /opt/tibco/ems/<version>/lib/tibjmsadmin.jar

# Copy to a normal directory on the iProcess host (Java 17-safe):
mkdir -p $SWDIR/java/lib/jms
scp user@<EMS_HOST_OR_IP>:/opt/tibco/ems/current/lib/tibjms.jar $SWDIR/java/lib/jms/
scp user@<EMS_HOST_OR_IP>:/opt/tibco/ems/current/lib/tibjmsadmin.jar $SWDIR/java/lib/jms/
```

Update `$SWDIR/etc/iapjms.classpath.properties` to include these JARs, for example by setting the EMS base directory to `$SWDIR/java/lib/jms` (or the local EMS client directory you chose).

### Step 5 — Configure JMS security credentials (if needed)

Edit `$SWDIR/etc/iapjms.properties` to confirm/update:

```properties
SecurityPrinciple=admin
SecurityCredentials=admin
```

### Step 6 — Configure Oracle shared library path

Ensure the Oracle libraries are accessible:
Replace the '19.x' in below command to the version installed.
```bash
# Add to /etc/ld.so.conf.d/oracle.conf
echo "/usr/lib/oracle/19.x/client64/lib" | sudo tee /etc/ld.so.conf.d/oracle.conf
sudo ldconfig
```

### Step 7 — Configure firewall

```bash
# iProcess uses dynamic RPC ports. Open the base port range.
# Check $SWDIR/etc/staffcfg for port ranges.
# The /etc/services file will contain entries like:
#   <NODE_NAME>_worker  nnnn/tcp
#   <NODE_NAME>_watcher mmmm/tcp
# If Oracle AQ is used across a firewall, also allow the configured AQ port range
# from the database server to the iProcess host.

sudo firewall-cmd --permanent --add-port=<worker-port>/tcp
sudo firewall-cmd --permanent --add-port=<watcher-port>/tcp
sudo firewall-cmd --reload
```

If your database server communicates with iProcess Engine using Oracle AQ, add firewall rules on the iProcess host to allow the AQ port range from the database server's IP address. Configure and verify the AQ port range used by iProcess before testing startup.

### Step 8 — License activation

Complete the license activation process after installation:

1. Download the license file from the TIBCO License Portal.
2. Copy the license file to a directory on the iProcess server (for example, `/opt/tibco/license`).
3. Make the iProcess user the owner of the license file.

```bash
sudo mkdir -p /opt/tibco/license
sudo chown staffpro:staffgrp /opt/tibco/license/<license-file>
```

4. Register the license using `swadm`:

```bash
su - staffpro
cd $SWDIR/bin
./swadm LICENSE_IMPORT /opt/tibco/license/<license-file>
```

---

## Part 6 — Start iProcess Engine

```bash
su - staffpro
source ./.kshrc

# Start iProcess Engine
$SWDIR/bin/swstart -p

# Optional: start (without process list)
# $SWDIR/bin/swstart

# Check running processes
ps -ef | grep -E 'staffbg|staffpro|staffwrk|iapjms' | grep -v grep

# View logs
tail -f $SWDIR/logs/sw_error.log
tail -f $SWDIR/logs/sw_warn.log
```

Expected processes (visible via `ps aux | grep sw`):

- `staffbg` — Background process
- `staffpro` — Procedure process
- `staffwrk` — Worker process(es)
- `iapjms` — JMS bridge (if activity publishing enabled)

---

## Quick Reference — Values for Your Installation

| Parameter | Value |
|---|---|
| Oracle TNS ID | *(your TNS alias pointing to `<ORACLE_DB_HOST_OR_IP>`)* |
| Oracle DBA | `ipeuser` / `ipeuser` |
| Schema Owner | `swpro` / *(choose password)* |
| DB Runtime User | `swuser` / *(choose password)* |
| Data Tablespace | `SWDATA` (or `USERS`) |
| Temp Tablespace | `SWTEMP` (or `TEMP`) |
| Linux background user | `staffpro` |
| Linux group | `staffgrp` |
| `$SWDIR` | `/opt/tibco/iprocess/<NODE_NAME>` |
| Node name | `<NODE_NAME>` |
| EMS URL | `tcp://<EMS_HOST_OR_IP>:7222` |
| EMS user/pass | `admin` / `admin` |
| JMS Context Factory | `com.tibco.tibjms.naming.TibjmsInitialContextFactory` |
| JMS Connection Factory | `GenericConnectionFactory` |

---

## Important Notes

1. **Oracle 19 with TNS (remote):** Since Oracle is on a separate server, you must install Oracle Instant Client (or full client) on the CentOS host and configure `tnsnames.ora` before running the installer. The `ORACLE_SID` environment variable should be set to a dummy value; the actual connection uses the TNS identifier.

2. **EMS JARs must be present before or right after installation:** The `iapjms` process won't start without `tibjms.jar`. Copy the JARs from your EMS installation.

3. **Oracle OPEN_CURSORS:** Must be set to at least 200. The installer checks this and will warn/fail if it is less.

4. **TODO file:** Always check `$SWDIR/logs/TODO` after installation. If it exists with content, those root-level commands must be executed manually.

5. **License activation:** The product will not function without a valid license. You must activate it via the TIBCO Activation Service using a license downloaded from `tibco.com/downloads`.

6. **In-product (file-based) activation prerequisite:** Install **TIBCO iProcess Engine Hotfix 1** before using in-product activation (`swadm LICENSE_IMPORT`).

7. **Korn shell:** CentOS Stream 9 does not include `ksh` by default. Install it with `dnf install ksh` before running the installer.

8. **SPO server not starting (name resolution/case check):** If the SPO server does not start, check the value of `MACHINE_NAME` from `swadm show_servers`. If `MACHINE_NAME` is uppercase, hostname resolution may fail. Verify this with `ping <MACHINE_NAME>`. If the case is incorrect, update it in the database under schema `SWPRO`, table `NODE_CLUSTER`.

9. **`swadm` login returns `Incorrect password`:** This usually indicates a file system permissions issue. To fix it:

```bash
1. Shut down the iProcess Engine.
2. Run `fixperms -r -y $SWDIR` as the root user in the `$SWDIR` directory.
3. Restart the iProcess Engine.
4. Check the `ls -lR` output again. You should now see `swrpcudp` spawned by the root user and the setuid bit updated to `s`.
5. After step 4 is validated, test it from your BW environment.
```
