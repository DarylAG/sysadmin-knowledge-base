
# Validar ejecución como root
```bash
if [ "$EUID" -ne 0 ]; then

echo "Por favor, ejecuta este script como root o usando sudo."

exit 1

fi
```

# Capturar entrada del usuario de forma segura (`read`)
Para solicitar datos al usuario en la terminal y guardarlos en variables, se utiliza el comando `read`. La mejor práctica es siempre utilizarlo con las banderas `-r` y `-p`.

**Sintaxis correcta:**_(Nota: La bandera `-p` debe ir seguida inmediatamente por el texto entre comillas)._
```bash
read -r -p "Nombre del Dominio (ej. lab.local): " DOMINIO
```

**Explicación de los parámetros:**

- **`read`**: Es el comando interno de Bash que lee una línea de la entrada estándar (lo que teclea el usuario) y asigna ese valor a la variable indicada al final (en este caso, `DOMINIO`).

- **`-r` (Raw / Modo crudo)**: Le indica a Bash que trate la entrada del usuario de forma completamente **literal**. Evita que las barras invertidas (`\`) sean interpretadas como caracteres especiales de escape.

- **`-p "Mensaje"` (Prompt)**: Permite mostrar un mensaje explicativo en la pantalla esperando la respuesta en la misma línea. Nos ahorra tener que escribir un comando `echo` antes del `read`.

# Verifica que el script se ejecute con privilegios de root 
```bash
verificar_root() { if [[ "${EUID}" -ne 0 ]]; then log_error "Este script debe ejecutarse como root (usa sudo)." fi }
```

# Ejemplo de copia con cambio de extension de archivos .bak
```bash
for archivo in *.bak; do cp "$archivo" "${archivo%.bak}.docu"; done
```


 Imprime mensajes informativos en verde para saber por dónde va el script 
 ```bash
 log_info() { echo -e "\n\e[32m[INFO]\e[0m $1" } # Imprime errores en rojo y sale del script log_error() { echo -e "\n\e[31m[ERROR]\e[0m $1" >&2 exit 1 }
 ```