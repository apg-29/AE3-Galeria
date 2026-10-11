# Certificado autofirmado de images.alumno.local

El archivo `images.alumno.local.crt` contiene el certificado público
utilizado por Nginx para servir la galería mediante HTTPS.

El certificado incluye los nombres DNS:
- `images.alumno.local`
- `www.images.alumno.local`

La clave privada `images.alumno.local.key` se almacena exclusivamente
en `/etc/ssl/dpl/` dentro del contenedor `dpl-lab`. Nunca debe
copiarse al repositorio.

## Generación

Comandos ejecutados dentro del contenedor:

```bash
mkdir -p /etc/ssl/dpl

openssl req -x509 -nodes -newkey rsa:2048 -days 365 -keyout /etc/ssl/dpl/images.alumno.local.key -out /etc/ssl/dpl/images.alumno.local.crt -subj "/CN=images.alumno.local" -addext "subjectAltName=DNS:images.alumno.local,DNS:www.images.alumno.local"

chmod 400 /etc/ssl/dpl/images.alumno.local.key
```

## Verificación

```bash
openssl x509 -in /etc/ssl/dpl/images.alumno.local.crt -noout -subject -dates -ext subjectAltName
```

Al ser autofirmado, el navegador no confiará automáticamente en
el certificado. Esto es normal en el entorno de prácticas.
En producción se utilizaría un certificado emitido por una autoridad
de certificación de confianza.
