```bash
#!/bin/bash

echo "Instalando nfs-kernel-server..."
apt-get update
apt-get install -y nfs-kernel-server

echo "Creando el directorio principal para las imágenes de Clonezilla..."
mkdir -p /srv/nfs/clonezilla

echo "Asignando permisos abiertos para evitar problemas de escritura..."
# Asignamos al usuario genérico y damos permisos de lectura/escritura totales
chown -R nobody:nogroup /srv/nfs/clonezilla
chmod -R 777 /srv/nfs/clonezilla

echo "Configurando /etc/exports..."
# Usamos 'no_root_squash' porque Clonezilla Live corre como root en los clientes 
# y necesita permisos para escribir los archivos de imagen sin restricciones.
cat <<EOF > /etc/exports
# Directorio único para imágenes de Clonezilla/DRBL
/srv/nfs/clonezilla 192.168.100.0/24(rw,sync,no_root_squash,no_subtree_check)
EOF

echo "Aplicando configuración y reiniciando el servicio NFS..."
exportfs -arv
systemctl daemon-reload
systemctl restart nfs-kernel-server
systemctl enable nfs-kernel-server

echo "¡Servidor NFS listo para Clonezilla!"
```
