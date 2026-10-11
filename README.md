# AE3 — Galería con procesado de imágenes, módulos y HTTPS

## Objetivo

Desplegar una galería web en images.alumno.local utilizando Nginx,
el módulo image_filter, HTTPS con certificado autofirmado,
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
