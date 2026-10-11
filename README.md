# AE3 — Galería con procesado de imágenes, módulos y HTTPS

## Objetivo

Desplegar una galería web en images.alumno.local utilizando Nginx,
el módulo `image_filter`, HTTPS con certificado autofirmado,
autenticación básica y análisis de logs con GoAccess.

## Estado del proyecto

Proyecto en desarrollo. En los siguientes apartados se documentarán
la instalación, la configuración, las comprobaciones y los problemas
encontrados durante la práctica.

## Instalación y activación del módulo `image_filter`

La práctica se realiza dentro del contenedor Docker `dpl-lab`, que
dispone de Nginx. El módulo `image_filter` permite redimensionar y
procesar imágenes cuando Nginx las sirve, sin generar miniaturas
intermedias.

### Instalación

Actualizar los índices de paquetes e instalar el módulo:

```bash
docker exec dpl-lab apt-get update
docker exec dpl-lab apt-get install -y libnginx-mod-http-image-filter
```

### Activación

En Ubuntu, la instalación del paquete habilita el módulo dinámico
mediante un archivo en `/etc/nginx/modules-enabled/`.

Comprobar que existe:

```bash
docker exec dpl-lab ls -l /etc/nginx/modules-enabled/
```

### Validación

Comprobar que Nginx acepta su configuración:

```bash
docker exec dpl-lab nginx -t
```

### Confirmar que Nginx reconoce el módulo

```bash
docker exec dpl-lab nginx -V 2>&1 | grep image_filter
docker exec dpl-lab nginx -T 2>&1 | grep -E "load_module|image_filter"
```

## Generación de imágenes de prueba

Las 20 imágenes originales se generan dentro del contenedor `dpl-lab`
con ImageMagick. Todas tienen dimensiones de 400x400 píxeles y nombres
desde `image01.jpg` hasta `image20.jpg`.

Comando utilizado:

```bash
docker exec dpl-lab bash -c 'for i in $(seq -w 1 20); do convert -size 400x400 gradient:"#2f6fed"-"#8bc34a" -fill white -pointsize 64 -gravity center -annotate 0 "IMG $i" "/var/www/images.alumno.local/img/image$i.jpg"; done'
```

Comprueba la primera imagen:

```bash
docker exec dpl-lab identify /var/www/images.alumno.local/img/image01.jpg
```

## Crear la aplicación de la galería

La aplicación web está formada por tres archivos:

- `web/index.html`: contiene el formulario con los cinco controles y el contenedor de la galería.
- `web/app.js`: crea las 20 imágenes y actualiza sus URL cuando el usuario pulsa el botón «Generar».
- `web/estilos.css`: define la presentación de la galería.

El formulario permite modificar el tamaño de las imágenes, el ancho del borde, su color, el enfoque y el desenfoque.

### Procesamiento de imágenes

La aplicación distribuye el trabajo entre el servidor y el cliente:

- **Servidor (Nginx):** procesa el redimensionado y el enfoque mediante el módulo `image_filter`. Los valores se envían en los parámetros `size` y `sharpen` de la URL.
- **Cliente (navegador):** aplica el ancho y el color del borde mediante CSS, así como el desenfoque con `filter: blur()`.

### Copiar la aplicación al DocumentRoot

Los archivos de la aplicación deben estar disponibles en el directorio servido por Nginx:

```bash
docker exec dpl-lab cp /home/alumno/dpl/ae3/web/index.html /var/www/images.alumno.local/
docker exec dpl-lab cp /home/alumno/dpl/ae3/web/app.js /var/www/images.alumno.local/
docker exec dpl-lab cp /home/alumno/dpl/ae3/web/estilos.css /var/www/images.alumno.local/
docker exec dpl-lab chmod -R a+rX /var/www/images.alumno.local
```
