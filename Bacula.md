
# En el servidor

Instalación de dependencias
```bash
 apt update && apt upgrade -y
  apt install apache2 mariadb-server mariadb-client php -y

apt install bacula bacula-client bacula-common-mysql bacula-director-mysql bacula-sd bacula-server bacula-tray-monitor -y

apt install perl libnet-ssleay-perl openssl libauthen-pam-perl libpam-runtime libio-pty-perl apt-show-versions libdbd-mysql-perl wget -y

wget -q https://sourceforge.net/projects/webadmin/files/webmin/2.670/webmin_2.670_all.deb -O webmin_2.670_all.deb

dpkg -i webmin_2.670_all.deb || apt-get install -f -y
```

preparacion_directorio
```bash
mkdir -p /copias
chmod 777 /copias
```

Configuracion de bacula
```bash
cp /etc/bacula/bacula-dir.conf /etc/bacula/bacula-dir.conf.bak
nano /etc/bacula/bacula-dir.conf
```

```text
# SERVICIO: El Director es el cerebro del clúster. Orquesta las tareas de respaldo,

# se comunica con los clientes (FD), envía los datos al almacenamiento (SD)

# y registra los metadatos en el catálogo (MariaDB).

Director {

Name = bacula-dir

DIRport = 9101

QueryFile = "/etc/bacula/scripts/query.sql"

WorkingDirectory = "/var/lib/bacula"

PidDirectory = "/run/bacula"

Maximum Concurrent Jobs = 20

# SEGURIDAD: Llave criptográfica requerida por bconsole u otros directores

# para enviar instrucciones a este Director.

Password = "${BACULA_DIR_PASSWORD}"

Messages = Daemon

DirAddress = ${SERVER_IP}

}

  

JobDefs {

Name = "DefaultJob"

Type = Backup

Level = Incremental

Client = bacula-fd

FileSet = "Full Set"

Schedule = "WeeklyCycle"

Storage = File

Messages = Standard

Pool = File

SpoolAttributes = yes

Priority = 10

Write Bootstrap = "/var/lib/bacula/%c.bsr"

}

  

Job {

Name = "BackupClient1"

JobDefs = "DefaultJob"

}

  

# SERVICIO: Trabajo crítico que respalda la propia base de datos (Catálogo)

# de Bacula. Sin esto, un desastre mayor impediría saber qué hay en los volúmenes.

Job {

Name = "BackupCatalog"

JobDefs = "DefaultJob"

Level = Full

FileSet="Catalog"

Schedule = "WeeklyCycleAfterBackup"

RunBeforeJob = "/etc/bacula/scripts/make_catalog_backup.pl MyCatalog"

RunAfterJob = "/etc/bacula/scripts/delete_catalog_backup"

Write Bootstrap = "/var/lib/bacula/%n.bsr"

Priority = 11

}

  

Job {

Name = "RestoreFiles"

Type = Restore

Client = bacula-fd

Storage = File

FileSet = "Full Set"

Pool = File

Messages = Standard

Where = /nonexistent/path/to/file/archive/dir/bacula-restores

}

  

FileSet {

Name = "Full Set"

Include {

Options {

signature = MD5

}

File = /usr/sbin

}

Exclude {

File = /var/lib/bacula

File = /proc

File = /tmp

File = /sys

File = /.journal

File = /.fsck

}

}

  

Schedule {

Name = "WeeklyCycle"

Run = Full 1st sun at 23:05

Run = Differential 2nd-5th sun at 23:05

Run = Incremental mon-sat at 23:05

}

  

Schedule {

Name = "WeeklyCycleAfterBackup"

Run = Full sun-sat at 23:10

}

  

FileSet {

Name = "Catalog"

Include {

Options {

signature = MD5

}

File = "/var/lib/bacula/bacula.sql"

}

}

  

Client {

Name = bacula-fd

Address = ${SERVER_IP}

FDPort = 9102

Catalog = MyCatalog

# RELACIÓN CRUZADA: El Director se autentica contra el demonio de archivos (File Daemon)

# del cliente utilizando esta contraseña para extraer sus datos.

Password = "${BACULA_FD_PASSWORD}"

File Retention = 60 days

Job Retention = 6 months

AutoPrune = yes

}

  

Storage {

Name = File

SDPort = 9103

# RELACIÓN CRUZADA: El Director presenta esta clave al Storage Daemon para que

# le permita escribir los flujos de datos entrantes en los discos duros.

Password = "${BACULA_SD_PASSWORD}"

Device = FileStorage

Media Type = File

Address = ${SERVER_IP}

}

  

Catalog {

Name = MyCatalog

dbname = "bacula"; DB Address = "localhost"; DB Port = "3306"; dbuser = "bacula";

# RELACIÓN CRUZADA: Conexión nativa con MariaDB. Esta contraseña vincula a Bacula

# con los esquemas SQL donde guarda nombres de archivos, permisos, volúmenes y hashes.

dbpassword = "${DB_PASSWORD}"

}

  

Messages {

Name = Standard

mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: %t %e of %c %l\" %r"

operatorcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: Intervention needed for %j\" %r"

mail = root = all, !skipped

operator = root = mount

console = all, !skipped, !saved

append = "/var/log/bacula/bacula.log" = all, !skipped

catalog = all

}

  

Messages {

Name = Daemon

mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula daemon message\" %r"

mail = root = all, !skipped

console = all, !skipped, !saved

append = "/var/log/bacula/bacula.log" = all, !skipped

}

  

Pool {

Name = Default

Pool Type = Backup

Recycle = yes

AutoPrune = yes

Volume Retention = 365 days

Maximum Volume Bytes = 50G

Maximum Volumes = 100

}

  

Pool {

Name = File

Pool Type = Backup

Recycle = yes

AutoPrune = yes

Volume Retention = 365 days

Maximum Volume Bytes = 50G

Maximum Volumes = 100

Label Format = "Vol-"

}

  

Pool {

Name = Scratch

Pool Type = Backup

}

  

Console {

Name = bacula-mon

# RELACIÓN CRUZADA: Configuración del monitor restringido (solo lectura de estados).

Password = "${BACULA_MON_PASSWORD}"

CommandACL = status, .status

}
```

CONFIGURACIÓN DE BACULA (STORAGE DAEMON)
```bash
cp /etc/bacula/bacula-sd.conf /etc/bacula/bacula-sd.conf.bak
```

```text
Storage {

Name = bacula-sd

SDPort = 9103

WorkingDirectory = "/var/lib/bacula"

Pid Directory = "/run/bacula"

Plugin Directory = "/usr/lib/bacula"

Maximum Concurrent Jobs = 20

SDAddress = ${SERVER_IP}

}

  

# RELACIÓN CRUZADA: El SD declara quién está autorizado a enviarle datos.

# En este caso, solo el 'bacula-dir' que presente la clave criptográfica correcta.

Director {

Name = bacula-dir

Password = "${BACULA_SD_PASSWORD}"

}

  

Director {

Name = bacula-mon

Password = "${BACULA_MON_PASSWORD}"

Monitor = yes

}

  

# SERVICIO: Configuración del destino final físico de los datos. En una arquitectura

# empresarial real, esto puede apuntar a cintas LTO, cabinas iSCSI o NAS.

Device {

Name = FileStorage

Media Type = File

Archive Device = /copias

LabelMedia = yes;

Random Access = Yes;

AutomaticMount = yes;

RemovableMedia = no;

AlwaysOpen = no;

Maximum Concurrent Jobs = 5

}

  

Messages {

Name = Standard

director = bacula-dir = all

}
```

CONFIGURACIÓN DE BACULA (BCONSOLE)
```bash
cp /etc/bacula/bconsole.conf /etc/bacula/bconsole.conf.bak
```

```text
Director {

Name = bacula-dir

DIRport = 9101

address = ${SERVER_IP}

# RELACIÓN CRUZADA: Autenticación de bconsole (la interfaz de línea de comandos).

# Utiliza la llave principal para gobernar el Director.

Password = "${BACULA_DIR_PASSWORD}"

}
```

REINICIAR Y VERIFICAR SERVICIOS
```bash
systemctl restart bacula-sd bacula-dir bacula-fd
systemctl status bacula-sd bacula-dir bacula-fd --no-pager
```

# En el Cliente
Instalación de paquetes
```bash
apt-get update
apt-get install -y bacula-fd
```

Configuracion bacula-fd
```bash
cp /etc/bacula/bacula-fd.conf /etc/bacula/bacula-fd.conf.bak

nano /etc/bacula/bacula-fd.conf
```

```text
FileDaemon {

# SERVICIO: Identidad de esta máquina. El Director debe tener un recurso 'Client'

# que coincida exactamente con este nombre.

Name = bacula-fd

FDport = 9102

WorkingDirectory = /var/lib/bacula

Pid Directory = /run/bacula

Maximum Concurrent Jobs = 20

Plugin Directory = /usr/lib/bacula

}

  

Director {

Name = bacula-dir

# RELACIÓN CRUZADA: Muro de seguridad del cliente. El Director remoto debe

# presentar esta clave para que el File Daemon le permita leer el disco duro local.

Password = "${BACULA_FD_PASSWORD}"

}

  

Director {

Name = bacula-mon

# RELACIÓN CRUZADA: Clave compartida para permitir que herramientas de

# monitorización (como tray-monitors) consulten el estado del agente.

Password = "${BACULA_MON_PASSWORD}"

Monitor = yes

}

  

Messages {

Name = Standard

director = bacula-dir = all, !skipped, !restored, !verified, !saved

}

```

