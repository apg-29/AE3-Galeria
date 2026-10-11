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

## Generar el certificado HTTPS

Para proteger la galería se utiliza un certificado autofirmado generado
con OpenSSL dentro del contenedor Docker `dpl-lab`.

El certificado se utiliza para el sitio `images.alumno.local` y contiene
también el nombre alternativo `www.images.alumno.local` mediante la
extensión SAN (Subject Alternative Name).

### Generación del certificado

Primero se crea el directorio destinado a los certificados:

```bash
docker exec dpl-lab mkdir -p /etc/ssl/dpl
```

A continuación, se genera el certificado autofirmado y su clave privada:

```bash
docker exec dpl-lab openssl req -x509 -nodes -newkey rsa:2048 -days 365 -keyout /etc/ssl/dpl/images.alumno.local.key -out /etc/ssl/dpl/images.alumno.local.crt -subj "/CN=images.alumno.local" -addext "subjectAltName=DNS:images.alumno.local,DNS:www.images.alumno.local"
```

La clave privada se protege con permisos restrictivos:

```bash
docker exec dpl-lab chmod 400 /etc/ssl/dpl/images.alumno.local.key
```

### Verificación

Se comprueban el sujeto, la validez y los nombres DNS del certificado:

```bash
docker exec dpl-lab openssl x509 -in /etc/ssl/dpl/images.alumno.local.crt -noout -subject -dates -ext subjectAltName
```

### Certificado público y clave privada

El certificado público se copia al directorio `certs/` del proyecto:

```bash
docker cp dpl-lab:/etc/ssl/dpl/images.alumno.local.crt ~/dpl/ae3/certs/images.alumno.local.crt
```

El archivo `certs/LEEME.md` contiene las instrucciones para generar
el certificado.

La clave privada permanece en `/etc/ssl/dpl/` dentro del contenedor
y no se incluye en el repositorio Git.

Al ser autofirmado, el navegador puede mostrar una advertencia de
confianza. Esto es esperado en el laboratorio. En producción se
utilizaría un certificado emitido por una autoridad de certificación
de confianza.

## Configurar la autenticación básica

La galería utiliza la autenticación HTTP Basic para proteger el acceso a las imágenes. Las credenciales son el usuario `alumno` y la contraseña `1234qwerty`.

### Crear el fichero de usuarios

Se genera el hash de la contraseña con OpenSSL y se guarda fuera del
DocumentRoot, en `/etc/nginx/htpasswd-images`:

```bash
docker exec dpl-lab bash -c 'echo "alumno:$(openssl passwd -apr1 1234qwerty)" > /etc/nginx/htpasswd-images'
```

### Proteger el fichero

Se asignan el propietario y el grupo necesarios para que Nginx pueda
leer el fichero:

```bash
docker exec dpl-lab chown root:www-data /etc/nginx/htpasswd-images docker exec dpl-lab chmod 640 /etc/nginx/htpasswd-images
```

### Comprobación

```bash
docker exec dpl-lab ls -l /etc/nginx/htpasswd-images
docker exec dpl-lab cat /etc/nginx/htpasswd-images
```

El fichero contiene el usuario y el hash de su contraseña, no la
contraseña en texto plano.

## Configuración del sitio en Nginx

Se ha creado el archivo de configuración `/etc/nginx/sites-available/004-images.alumno.local` dentro del contenedor `dpl-lab`.

### Bloques map

Se han definido tres directivas `map` para controlar los parámetros que recibe la aplicación mediante la URL:

- `$img_size`: permite los tamaños 150, 300 y 600 píxeles. El valor predeterminado es 300.
- `$img_quality`: permite las calidades JPEG 60, 80 y 100. El valor predeterminado es 80.
- `$img_sharpen`: permite los niveles de enfoque 0, 30, 60 y 100. El valor predeterminado es 0.

Estas listas blancas evitan que se utilicen valores arbitrarios
en los parámetros de procesamiento de imágenes.

Las directivas `map` se declaran en el contexto `http`, fuera de los bloques `server`, porque Nginx no permite definirlas dentro de un servidor o una ubicación.

### Servidor HTTPS

El bloque principal configura el sitio `images.alumno.local`:

- Escucha las conexiones HTTPS por el puerto 443.
- Utiliza `/var/www/images.alumno.local` como DocumentRoot.
- Define `index.html` como página de inicio.
- Utiliza el certificado y la clave privada configurados para el sitio.

La ubicación `/` sirve los archivos de la aplicación y utiliza `try_files` para comprobar que los recursos solicitados existen.

### Procesamiento y autenticación de imágenes

La ubicación `/img/` contiene las directivas de procesamiento del módulo `image_filter`:

- `image_filter resize`: redimensiona las imágenes utilizando el tamaño permitido.
- `image_filter_sharpen`: aplica el nivel de enfoque solicitado.
- `image_filter_jpeg_quality`: establece la calidad JPEG.
- `image_filter_buffer 5M`: establece un búfer máximo de 5 MB para leer las imágenes.

Esta ubicación también utiliza autenticación básica mediante `auth_basic` y el archivo de usuarios `/etc/nginx/htpasswd-images`.

La directiva `error_page 401 @sin_autorizacion` permite gestionar la respuesta de autenticación sin que el filtro de imágenes transforme la respuesta de error. La ubicación con nombre `@sin_autorizacion` devuelve el código HTTP 401.

### Redirecciones

Se han definido dos bloques adicionales:

- Las peticiones HTTP a `images.alumno.local` se redirigen a HTTPS.
- Las peticiones HTTP a `www.images.alumno.local` se redirigen al dominio `https://images.alumno.local`.

### Registros

Se han configurado archivos específicos para registrar la actividad del sitio:

- `/var/log/nginx/images-alumno.access.log`
- `/var/log/nginx/images-alumno.error.log`

### Comprobación de la configuración

Antes de activar el sitio, se ha comprobado la sintaxis de Nginx:

```bash
docker exec dpl-lab nginx -t
```

## Activación y validación del sitio

Se activa el sitio creando un enlace simbólico desde `sites-available` hacia `sites-enabled`.

```bash
docker exec dpl-lab ln -s /etc/nginx/sites-available/004-images.alumno.local /etc/nginx/sites-enabled/
```

Se comprueba la sintaxis de Nginx mediante `nginx -t` y, si la validación es correcta, se recarga el servicio para aplicar los cambios.

Finalmente, se verifica la configuración cargada mediante `nginx -T`:

```bash
docker exec dpl-lab nginx -T | grep -E "server_name images|image_filter|load_module" | head
```

Salida:

```text
load_module modules/ngx_http_image_filter_module.so;
    server_name images.alumno.local;
        # Sin esto, image_filter convierte la pagina de error del 401 en un
        image_filter resize $img_size $img_size;
        image_filter_sharpen $img_sharpen;
        image_filter_jpeg_quality $img_quality;
        image_filter_buffer 5M;
    # Respuesta de autenticacion sin image_filter de por medio
    server_name images.alumno.local;
```
