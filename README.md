# Gitlab CE

## Modo de implementación:

### 1. Levantar contenedor

```bash
docker-compose up -d
```

### 2. Ingresar a http://localhost:8929 y esperar que esté listo (lleva a la pantalla de login)

### 3. Conseguir contraseña del usuario root

```bash
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

### 4. Iniciar sesión

Usuario: root
Contraseña: La conseguida en el punto 3

## Configuración del primer runner

### 1. Crear runner en gitlab

- Ir a http://localhost:8929/admin/runners -> Click en el botón ```New instance runner```
- Llenar con la configuración deseada

### 2. Registrar runner

- Siguiendo la guía que nos da en la misma página, hay que entrar en el contenedor del gitlab-runner y ejecutar

```bash
gitlab-runner register  --url http://gitlab  --token <token>
```

- **Ingresar URL de gitlab:** Se puede dejar vacío porque toma por defecto la escrita en el comando
- **Ingresar nombre del runner:** Elegir un nombre representativo, no es importante para la funcionalidad
- **Ingresar executor:** Se elige sobre una lista, ```Docker``` para mayor compatibilidad y simplicidad
- **Ingresar imagen de Docker por defecto:**
| Tecnología | Imagen recomendada |
|--|--|
| **Python** | `python:3.11` |
| **Node.js** | `node:20` |
| **Java** | `openjdk:17` |
| **Go** | `golang:1.21` |
| **PHP** | `php:8.2` |
| **Ruby** | `ruby:3.2` |
| **.NET** | `mcr.microsoft.com/dotnet/sdk:8.0` |
| **Docker dentro de Docker** | `docker:latest` |
| **CI/CD genérico** | `alpine:latest` (liviano) o `ubuntu:latest` (más completo) |

**Tener en cuenta que las versiones tienen que ser la misma o compatible con la utilizada en el proyecto**

### 3. Configurar network del runner

**Problema:** Al elegir tipo "Docker", cuando se corre un job, se va a crear un contenedor nuevo por lo que queda sin acceso a la red global.
**Solución:** Hay que configurar el ```.toml``` del runner para que los cree con la red global del stack.

Para lograr la solución se agrega la siguiente propiedad debajo de la clausula ```[runners.docker]```
```bash
network_mode = "gitlab_new_gitlab-network"
```

## (Opcional) Configuraciones para reducir el consumo de RAM

- Ir al archivo ```/etc/gitlab/gitlab.rb```

### 1. Reducir la cantidad de procesos de ```puma```
- Modificar y descomentar todas las lineas relacionadas con ```puma``` de la primer sección para que queden así
```ruby
puma['enable'] = true # Puma es el servidor HTTP. Al dejar en true se usa puma en vez de unicorn para que sea más liviano
puma['ha'] = false # High Availability para balanceo de carga
puma['worker_timeout'] = 30 # Si un worker no responde en x segundos, Puma lo reinicia
puma['worker_processes'] = 2 # Numero de procesos de Puma (workers). Se podría bajar a 1 si hay menos de 10 usuarios recurrentes
puma['min_threads'] = 1 # Numero mínimo de threads por worker. Se puede dejar en 1 mientras se tenga menos de 20 usuarios recurrentes
puma['max_threads'] = 4 # Numero máximo de threads por worker. Pequeña = 2, Media = 4, Grande = 8
```

### 2. Reducir la cantidad de procesos de ```sidekiq```
- Modificar y descomentar todas las lineas relacionadas con ```sidekiq``` de la primer sección para que queden así
```ruby
sidekiq['enable'] = true # Habilita sidekiq. Tiene que estar habilitado para que Gitlab esté totalmente funcional
# sidekiq['log_directory'] = "/var/log/gitlab/sidekiq" # Directorio donde se guardan los logs de Sidekiq, queda comentado porque no lo queremos modificar
sidekiq['log_format'] = "json" # Formato de logs. Queda en json aunque sean más pesado que "plain" para tener mejores logs
sidekiq['shutdown_timeout'] = 4 # Tiempo que espera Sidekiq antes de forzar cierre de procesos. 4 es un valor relativamente bajo y ahorra recursos
# sidekiq['interval'] = nil # Intervalo de chequeo de tareas. nil es el valor por defecto. Queda así para no forzar chequeos más frecuentes
sidekiq['concurrency'] = 10 # Numero de threads simultáneos en Sidekiq. Pequeña = 5, Media = 10, Grande = 20
```

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