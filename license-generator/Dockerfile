FROM ruby:alpine

# Instalar dependencias necesarias
RUN apk add --no-cache --virtual .build-deps \
    build-base \
    git \
    && apk add --no-cache \
    bash \
    curl \
    ruby-dev \
    && gem install bundler

# Configurar el directorio de trabajo
WORKDIR /app

# Clonar el repositorio
RUN git clone --depth 1 https://github.com/Lakr233/GitLab-License-Generator.git .

# Dar permisos de ejecución a los scripts necesarios
RUN chmod +x make.sh ./src/scan.features.rb

# Usar ENTRYPOINT para ejecutar el script y permitir override de parámetros en `docker run`
ENTRYPOINT ["./make.sh"]