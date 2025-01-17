# Gitlab CE

## Modificaciones:

### 1. Uso de volumenes

✅ Ventajas de usar volúmenes de Docker:

- Docker maneja automáticamente el almacenamiento de los datos.
- Facilita la migración entre máquinas sin preocuparse por permisos de archivos.
- Menos problemas con compatibilidad entre sistemas de archivos.
- Mejor aislamiento y mantenimiento.

Si se necesita migrar a este sistema, para llevar la data del disco a los volumenes
```bash
docker run --rm -v $(pwd)/config:/old -v gitlab_config:/new busybox sh -c "cp -a /old/. /new/"
docker run --rm -v $(pwd)/logs:/old -v gitlab_logs:/new busybox sh -c "cp -a /old/. /new/"
docker run --rm -v $(pwd)/data:/old -v gitlab_data:/new busybox sh -c "cp -a /old/. /new/"
```
📌 Explicación:
Estos comandos copian datos de archivos locales (dentro del directorio actual) a volúmenes de Docker.

- ```docker run --rm``` → Crea un contenedor temporal (se borra al finalizar).
- ```-v $(pwd)/config:/old``` → Monta la carpeta local ```config``` como ```/old``` en el contenedor.
- ```-v gitlab_config:/new``` → Monta el volumen de Docker ```gitlab_config``` como ```/new```.
- ```busybox sh -c "cp -a /old/. /new/"``` → Copia todos los archivos desde ```/old/``` a ```/new/``` de forma recursiva y mantiene permisos.

- Se repite lo mismo para ```logs``` y ```data```.

Para hacer backup de los volumenes
```bash
docker run --rm -v gitlab_data:/data -v $(pwd):/backup busybox tar czf /backup/gitlab_data_backup.tar.gz -C /data .
docker run --rm -v gitlab_config:/data -v $(pwd):/backup busybox tar czf /backup/gitlab_config_backup.tar.gz -C /data .
docker run --rm -v gitlab_logs:/data -v $(pwd):/backup busybox tar czf /backup/gitlab_logs_backup.tar.gz -C /data .
```
📌 Explicación:
Estos comandos crean archivos comprimidos (.tar.gz) con los datos de los volúmenes.

- ```-v gitlab_data:/data``` → Monta el volumen ```gitlab_data``` como ```/data``` en el contenedor.
- ```-v $(pwd):/backup``` → Usa el directorio actual como ```/backup```.
- ```tar czf /backup/gitlab_data_backup.tar.gz -C /data .``` →
    - Crea un archivo comprimido ```.tar.gz``` en ```/backup/```
    - ```-C /data .``` → Comprime todo el contenido del volumen.

- Se hace lo mismo para ```gitlab_config``` y ```gitlab_logs```.

Para restaurar
```bash
docker run --rm -v gitlab_data:/data -v $(pwd):/backup busybox tar xzf /backup/gitlab_data_backup.tar.gz -C /data
docker run --rm -v gitlab_config:/data -v $(pwd):/backup busybox tar xzf /backup/gitlab_config_backup.tar.gz -C /data
docker run --rm -v gitlab_logs:/data -v $(pwd):/backup busybox tar xzf /backup/gitlab_logs_backup.tar.gz -C /data
```
📌 Explicación:
Estos comandos descomprimen los backups en sus volúmenes originales.

- ```tar xzf /backup/gitlab_data_backup.tar.gz -C /data ``` →
    - Extrae (```xzf```) el contenido del backup al volumen montado en ```/data```.

- Se hace lo mismo para ```gitlab_config``` y ```gitlab_logs```.