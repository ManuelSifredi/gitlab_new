# Licence generator

[Link de referencia](https://github.com/Lakr233/GitLab-License-Generator)

## Modo de uso

1. Buildear la imagen con

```docker build -t gitlab_license_generator .```

2. Generar archivos de licencia con

```docker run --rm -v "$(pwd)/license:/app/build" gitlab_license_generator```

Esto va a generar una carpeta llamada ```license``` en la carpeta desde donde se ejecutó el comando con 5 archivos
```features.json```
```license.json```
```private.key```
```public.key```
```result.gitlab-license```

3. Entrar al contenedor y reemplazar el contenido del archivo ```/opt/gitlab/embedded/service/gitlab-rails/.license_encryption_key.pub``` por el contenido del archivo generado ```/license/public.key```

4. Entrar a Gitlab y registrar la licencia
- Ir a la sección ```Admin```, se encuentra en la parte inferior derecha de la sidebar
- ```Settings``` -> ```General```
- ```Add License```
- Si no está elegida por defecto, elegir la opción ```Upload .gitlab-license file``` y seleccionar el archivo con el mismo nombre de la carpeta generada ```license```
- Checkear el input de la sección ```Terms of service```
- ```Add license```