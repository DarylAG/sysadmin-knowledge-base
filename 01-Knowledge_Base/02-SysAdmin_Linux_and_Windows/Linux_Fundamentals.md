Documentación técnica, comandos esenciales y procedimientos para la administración de sistemas operativos basados en Linux.

## 📁 Gestión de Directorios y Archivos
```bash
# Ver el directorio actual de trabajo
pwd

# Listar archivos (incluyendo ocultos y formato detallado)
ls -la

# Crear directorios de forma anidada
mkdir -p /ruta/a/directorios/padre/hijo

# Copiar archivos y directorios de forma recursiva y modo verbose
cp -rv /origen/archivo_o_carpeta /destino/

# Mover o renombrar archivos/directorios
mv archivo_antiguo.txt archivo_nuevo.txt

# Eliminar archivos o directorios de forma recursiva y forzada
rm -rf /ruta/a/eliminar
```


## 👥 Gestión de Usuarios y Permisos

### Usuarios y Grupos
```bash
# Crear un nuevo usuario asignándole directorio home y shell bash
sudo useradd -m -s /bin/bash nombre_usuario

# Establecer o cambiar la contraseña de un usuario
sudo passwd nombre_usuario

# Añadir un usuario existente a un grupo suplementario (ej. sudo)
sudo usermod -aG sudo nombre_usuario

# Modificar permisos de un archivo o directorio (ej. 755 o 644)
chmod -R 755 /ruta/al/directorio

# Cambiar el propietario y grupo de un recurso
sudo chown -R usuario:grupo /ruta/al/recurso
```

## 🔑 Gestión de SSH y Autenticación por Llaves

### Generación de Pares de Llaves
- **Opción A (Sin contraseña / Sin passphrase):** Ideal para automatización o scripts donde no se requiere intervención manual. Al solicitar la contraseña, simplemente presiona `Enter` dos veces para dejarla vacía.
```Bash
ssh-keygen -t rsa -b 4096
```

- **Opción B (Cifrada con contraseña / Con passphrase):** Más segura para entornos de producción. Protege la llave privada en el disco con una contraseña adicional que se solicitará al usarla (o al cargarla en el agente SSH).
```Bash
ssh-keygen -t rsa -b 4096 -N "tu_contraseña_segura" -f ~/.ssh/id_rsa_cifrada
```

### Copiar la Llave al Servidor Remoto
```Bash
ssh-copy-id usuario@ip_o_host_del_servidor
```

## ⚙️ Procesos y Servicios (Demonios)
```Bash
# Listar todos los procesos activos con información detallada (equivalente a Task Manager)
ps aux

# Monitorear procesos en tiempo real
top
# (Alternativa más moderna y visual: htop)

# Terminar un proceso por su ID (PID) de forma limpia y forzada si no responde
kill 1234
kill -9 1234

# Gestionar demonios del sistema (systemd)
sudo systemctl start nombre_servicio
sudo systemctl stop nombre_servicio
sudo systemctl restart nombre_servicio
sudo systemctl enable --now nombre_servicio
sudo systemctl status nombre_servicio
```

## 📶 Conectividad y Redes

### NetworkManager (`nmcli`)
```Bash
# Listar las interfaces de red disponibles y su estado
nmcli device status

# Listar las conexiones de red configuradas
nmcli connection show

# Levantar o bajar una interfaz/conexión de red específica
nmcli connection up "Nombre_Conexion"
nmcli connection down "Nombre_Conexion"

# Configurar una IP estática en una conexión
nmcli connection modify "Nombre_Conexion" ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8 ipv4.method manual
```

### systemd-networkd
```Bash
# Verificar el estado del servicio de red
sudo systemctl status systemd-networkd

# Los archivos de configuración de red se ubican en /etc/systemd/network/
# Ejemplo de recarga de configuración:
sudo networkctl reload
```

## 💾 Montaje y Desmontaje de Unidades
```Bash
# Listar discos y particiones conectadas al sistema
lsblk
sudo fdisk -l

# Montar una partición o unidad en un directorio destino
sudo mount /dev/sdb1 /mnt/punto_montaje

# Montar una unidad indicando el tipo de sistema de archivos explícitamente
sudo mount -t ext4 /dev/sdb1 /mnt/punto_montaje

# Desmontar una unidad de forma segura
sudo umount /mnt/punto_montaje

# Desmontaje forzado (en caso de que el recurso esté ocupado)
sudo umount -f /mnt/punto_montaje
```

## 📦 Compresión y Descompresión de Paquetes
```Bash
# Comprimir un directorio en un archivo tar.gz
tar -czvf archivo_comprimido.tar.gz /ruta/a/carpeta/

# Descomprimir un archivo tar.gz en el directorio actual
tar -xzvf archivo_comprimido.tar.gz

# Comprimir en formato zip
zip -r archivo.zip /ruta/a/carpeta/

# Descomprimir un archivo zip
unzip archivo.zip
```

## 🛠️ Administración de Paquetes (APT - Debian/Ubuntu/Mint)
```Bash
# Actualizar la lista de repositorios locales
sudo apt update

# Actualizar todos los paquetes instalados en el sistema
sudo apt upgrade -y

# Instalar una nueva aplicación o herramienta
sudo apt install nombre_paquete -y

# Desinstalar un paquete conservando o purgando sus archivos de configuración
sudo apt remove nombre_paquete
sudo apt purge nombre_paquete

# Limpiar paquetes huérfanos y caché innecesaria
sudo apt autoremove -y && sudo apt clean
```

## 🔍 Filtrado y Manipulación de Texto
```Bash
# grep: Buscar patrones o palabras clave dentro de archivos o salidas de comandos
cat /etc/passwd | grep "root"

# sed: Reemplazar texto al vuelo (ej. cambiar "hola" por "adios" en un archivo)
sed -i 's/hola/adios/g' archivo.txt

# awk: Procesar columnas o campos delimitados (ej. imprimir la primera columna)
ls -l | awk '{print $1, $9}'

# nl: Numerar las líneas de un archivo o salida estándar
cat archivo.txt | nl

# wc: Contar líneas, palabras y caracteres (Word Count)
wc -l archivo.txt
```

## 📝 Uso de `EOF` (End Of File) en Scripting
El operador `EOF` (delimitador de fin de archivo, comúnmente usado junto con `cat`) permite enviar múltiples líneas de texto directamente a un archivo o a la entrada estándar de un comando sin necesidad de abrir un editor interactivo (como `nano` o `vim`). Es fundamental en scripts de automatización para generar archivos de configuración, JSONs o scripts temporales.

### Ejemplo 1: Crear un archivo estructurado (como un JSON)
```Bash
cat <<EOF > request.json
{
  "usuario": "administrador",
  "permisos": "total",
  "estado": "activo"
}
EOF
```

### Ejemplo 2: Creación de scripts o configuraciones con variables

- **Con expansión de variables (comportamiento por defecto):** Si usas `<<EOF`, (como <<EOF Bash `$USER`) ```bash antes archivo. cat de el escribir internas las por reemplazará su valor variables> info.txt El usuario actual es: $USER La fecha de ejecución es: $(date) EOF

- **Evitando la expansión (comillas simples o escapando el delimitador):** Si usas `'<<EOF'`, el texto se escribe exactamente tal cual, respetando los signos de dólar `$`.
```
cat <<'EOF' > script_ejemplo.sh
echo "Esta variable no se evalúa: $VARIABLE"
EOF
```

## 🔵 Apéndice: Configuración Rápida de Bluetooth por Terminal
```Bash
# Verificar el estado del servicio bluetooth
systemctl status bluetooth

# Ingresar a la interfaz interactiva de control bluetooth
bluetoothctl

# (Una vez dentro de la consola bluetoothctl, ejecutar los siguientes comandos):
power on
agent on
default-agent
scan on
pair XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
exit
```


Ejemplo de copia con cambio de extension de archivos .bak
```bash
for archivo in *.bak; do cp "$archivo" "${archivo%.bak}.docu"; done
```


 Imprime mensajes informativos en verde para saber por dónde va el script 
 ```bash
 log_info() { echo -e "\n\e[32m[INFO]\e[0m $1" } # Imprime errores en rojo y sale del script log_error() { echo -e "\n\e[31m[ERROR]\e[0m $1" >&2 exit 1 }
 ```


Verifica que el script se ejecute con privilegios de root 
```bash
verificar_root() { if [[ "${EUID}" -ne 0 ]]; then log_error "Este script debe ejecutarse como root (usa sudo)." fi }
```
