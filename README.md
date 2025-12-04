# BookStack

Plataforma de documentación y wiki de código abierto. Organiza contenido en libros, capítulos y páginas con editor WYSIWYG y Markdown.

## Características

- 📚 **Organización jerárquica**: Libros → Capítulos → Páginas
- ✏️ **Editor dual**: WYSIWYG y Markdown
- 🔍 **Búsqueda potente**: Busca en todo el contenido
- 🔐 **Control de acceso**: Permisos granulares por rol
- 📝 **Historial de cambios**: Seguimiento completo de ediciones
- 🖼️ **Gestión de imágenes**: Biblioteca de medios integrada
- 🔗 **Integración**: LDAP, SAML, OAuth
- 🌍 **Multi-idioma**: Soporte para múltiples idiomas

## Requisitos Previos

- Docker Engine instalado
- Portainer configurado (recomendado)
- **Para Traefik o NPM**: Red Docker `proxy` creada
- **Dominio configurado**: Para acceso HTTPS
- **Contraseña generada**: DB_PASSWORD

⚠️ **IMPORTANTE**: BookStack requiere MariaDB. Este compose incluye el contenedor de base de datos.

## Generar Contraseña

**Antes de cualquier despliegue**, genera una contraseña segura:

```bash
# DB_PASSWORD (MariaDB)
openssl rand -base64 32
```

Guarda el resultado, lo necesitarás en el archivo `.env`.

> ⚠️ **Importante**: Usa comillas simples en el archivo `.env` si la contraseña contiene caracteres especiales.
> Ejemplo: `DB_PASSWORD='tu_password_generado'`

---

## Despliegue con Portainer

### Opción A: Git Repository (Recomendada)

Permite mantener la configuración actualizada automáticamente desde Git.

1. En Portainer, ve a **Stacks** → **Add stack**
2. Nombra el stack: `bookstack`
3. Selecciona **Git Repository**
4. Configura:
   - **Repository URL**: `https://git.ictiberia.com/groales/bookstack`
   - **Repository reference**: `refs/heads/main`
   - **Compose path**: `docker-compose.yml`
   - **GitOps updates**: Activado (opcional - auto-actualización)
5. **Solo para Traefik**: En **Additional paths**, añade:
   - `docker-compose.override.traefik.yml.example`
6. En **Environment variables**, añade:

```env
DB_PASSWORD=tu_password_generado
```

7. **Solo para Traefik**: Añade también `DOMAIN_HOST=bookstack.example.com`
8. Click en **Deploy the stack**

### Opción B: Web editor

Para personalización completa del compose.

1. En Portainer, ve a **Stacks** → **Add stack**
2. Nombra el stack: `bookstack`
3. Selecciona **Web editor**
4. Pega el contenido de `docker-compose.yml`
5. En **Environment variables**, añade las mismas variables que la Opción A
6. Click en **Deploy the stack**

---

## Modos de Despliegue

### Traefik (Proxy Inverso con SSL automático)

**Requisitos**:
- Stack de Traefik desplegado
- Red `proxy` creada
- DNS apuntando al servidor

Si desplegaste con **Opción A (Git Repository)**, ya configuraste todo en el paso 5 y 7. Simplemente accede a `https://bookstack.tudominio.com`

Si usas otra forma de despliegue:

1. Asegúrate de tener el archivo `docker-compose.override.traefik.yml.example` como `docker-compose.override.yml`

2. En las **Environment variables**, añade:
   ```env
   DOMAIN_HOST=bookstack.tudominio.com
   ```

3. Despliega el stack y accede a `https://bookstack.tudominio.com`

**Ejemplo de compose completo con Traefik**:

```yaml
services:
  bookstack:
    container_name: bookstack
    image: lscr.io/linuxserver/bookstack:latest
    restart: unless-stopped
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Europe/Madrid
      APP_URL: https://${DOMAIN_HOST:-bookstack.example.com}
      DB_HOST: bookstack-db
      DB_PORT: 3306
      DB_DATABASE: ${DB_NAME:-bookstack}
      DB_USERNAME: ${DB_USER:-bookstack}
      DB_PASSWORD: ${DB_PASSWORD}
    volumes:
      - bookstack_config:/config
    networks:
      - proxy
      - bookstack-internal
    depends_on:
      - bookstack-db
    labels:
      - traefik.enable=true
      - traefik.http.routers.bookstack-http.rule=Host(`${DOMAIN_HOST}`)
      - traefik.http.routers.bookstack-http.entrypoints=web
      - traefik.http.routers.bookstack-http.middlewares=redirect-to-https
      - traefik.http.routers.bookstack.rule=Host(`${DOMAIN_HOST}`)
      - traefik.http.routers.bookstack.entrypoints=websecure
      - traefik.http.routers.bookstack.tls=true
      - traefik.http.routers.bookstack.tls.certresolver=letsencrypt
      - traefik.http.routers.bookstack.service=bookstack-svc
      - traefik.http.services.bookstack-svc.loadbalancer.server.port=80
      - traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
      - traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true

  bookstack-db:
    container_name: bookstack-db
    image: mariadb:11-alpine
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_NAME:-bookstack}
      MYSQL_USER: ${DB_USER:-bookstack}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
    volumes:
      - bookstack_db:/var/lib/mysql
    networks:
      - bookstack-internal

volumes:
  bookstack_config:
    name: bookstack_config
  bookstack_db:
    name: bookstack_db

networks:
  proxy:
    external: true
  bookstack-internal:
    name: bookstack-internal
```

### Nginx Proxy Manager (NPM)

**Requisitos**:
- NPM desplegado y accesible
- Red `proxy` creada
- DNS apuntando al servidor

**Pasos**:

1. Despliega el stack con el `docker-compose.yml` base (sin override)

2. En NPM, crea un nuevo **Proxy Host**:
   - **Domain Names**: `bookstack.tudominio.com`
   - **Scheme**: `http`
   - **Forward Hostname / IP**: `bookstack`
   - **Forward Port**: `80`
   - **Cache Assets**: ✅ Activado
   - **Block Common Exploits**: ✅ Activado
   - **Websockets Support**: ✅ Activado

3. En la pestaña **SSL**:
   - **SSL Certificate**: Request a new SSL Certificate (Let's Encrypt)
   - **Force SSL**: ✅ Activado
   - **HTTP/2 Support**: ✅ Activado
   - **HSTS Enabled**: ✅ Activado (opcional)

4. Guarda y accede a `https://bookstack.tudominio.com`

---

## Configuración Inicial

### Primer Acceso

1. Accede a BookStack usando tu dominio configurado
2. Login con credenciales por defecto:
   - **Email**: `admin@admin.com`
   - **Password**: `password`

3. **⚠️ CRÍTICO**: Cambia la contraseña inmediatamente:
   - Click en tu avatar (esquina superior derecha)
   - **Edit Profile** → **Change Password**

### Panel de Administración

Accede al panel: **Settings** (engranaje superior derecho)

#### Configuración Básica

**Settings → Settings**:
- **Application Name**: Nombre de tu wiki
- **Application Description**: Descripción breve
- **Application Logo**: Sube tu logo personalizado
- **Default Language**: Español u otro idioma

#### Registro de Usuarios

**Settings → Registration Settings**:
- **Allow public registration**: Desactivar (❌)
- **Allow public viewing**: Activar si quieres que usuarios no autenticados puedan leer
- **Default role for new users**: Seleccionar rol apropiado

#### Roles y Permisos

**Settings → Roles**:

1. **Admin**: Control total
2. **Editor**: Puede crear y editar contenido
3. **Viewer**: Solo lectura

Crea roles personalizados según necesites:
- **Permissions**: Granular por libros, capítulos, páginas
- **System Permissions**: Gestión de imágenes, ajustes

#### Personalización de Tema

**Settings → Customization**:
- **Custom HTML Head Content**: Añadir CSS personalizado
- **Color Scheme**: Claro/Oscuro
- **Custom Styles**: CSS para personalización avanzada

---

## Personalización

### Integración con LDAP

**Settings → Authentication → LDAP**:

```env
# Añadir al .env
LDAP_SERVER=ldap.example.com:389
LDAP_BASE_DN=dc=example,dc=com
LDAP_DN=cn=bookstack,ou=Services,dc=example,dc=com
LDAP_PASS=password
LDAP_USER_FILTER=(&(uid=${user}))
LDAP_VERSION=3
```

Reinicia el contenedor: `docker restart bookstack`

### SAML / OAuth

**Settings → Authentication**:
- **SAML 2.0**: Para Active Directory con ADFS
- **OAuth**: Google, GitHub, GitLab, etc.

Consulta la [documentación oficial](https://www.bookstackapp.com/docs/admin/third-party-auth/) para configuración específica.

### Comandos Personalizados

Acceso al contenedor:

```bash
# Shell de BookStack
docker exec -it bookstack bash

# Comandos artisan disponibles
docker exec bookstack php artisan list

# Limpiar caché
docker exec bookstack php artisan cache:clear

# Regenerar permisos de búsqueda
docker exec bookstack php artisan bookstack:regenerate-search
```

---

## Backup y Restauración

### Backup Manual

```bash
# Backup de MariaDB
docker exec bookstack-db mariadb-dump -u bookstack -p${DB_PASSWORD} bookstack > bookstack-backup-$(date +%Y%m%d).sql

# Backup de configuración y uploads
docker run --rm -v bookstack_config:/backup -v $(pwd):/target alpine tar czf /target/bookstack-config-$(date +%Y%m%d).tar.gz -C /backup .
```

### Backup Automático

Crea un script en `/root/backup-bookstack.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/backups/bookstack"
DATE=$(date +%Y%m%d-%H%M%S)
DB_PASSWORD="tu_password_aqui"

mkdir -p $BACKUP_DIR

# MariaDB
docker exec bookstack-db mariadb-dump -u bookstack -p${DB_PASSWORD} bookstack | gzip > $BACKUP_DIR/bookstack-db-$DATE.sql.gz

# Configuración
docker run --rm -v bookstack_config:/backup -v $BACKUP_DIR:/target alpine tar czf /target/bookstack-config-$DATE.tar.gz -C /backup .

# Limpiar backups antiguos (mantener 7 días)
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completado: $DATE"
```

Programa con cron:

```bash
chmod +x /root/backup-bookstack.sh
crontab -e

# Backup diario a las 2 AM
0 2 * * * /root/backup-bookstack.sh
```

### Restauración

```bash
# Detener BookStack
docker stop bookstack

# Restaurar MariaDB
gunzip < bookstack-db-20250101.sql.gz | docker exec -i bookstack-db mariadb -u bookstack -p${DB_PASSWORD} bookstack

# Restaurar configuración
docker run --rm -v bookstack_config:/restore -v $(pwd):/source alpine tar xzf /source/bookstack-config-20250101.tar.gz -C /restore

# Iniciar BookStack
docker start bookstack
```

---

## Actualización

### Actualizar BookStack

```bash
# 1. Backup ANTES de actualizar
docker exec bookstack-db mariadb-dump -u bookstack -p${DB_PASSWORD} bookstack > bookstack-pre-update-$(date +%Y%m%d).sql

# 2. Detener stack
docker stop bookstack bookstack-db

# 3. Actualizar imágenes
docker pull lscr.io/linuxserver/bookstack:latest
docker pull mariadb:11-alpine

# 4. Iniciar stack
docker start bookstack-db
sleep 10
docker start bookstack

# 5. Verificar logs
docker logs -f bookstack

# 6. Verificar versión en Settings → About
```

### Actualizar MariaDB

Si necesitas actualizar de MariaDB 10 a 11 (ya está en 11):

```bash
# 1. Backup
docker exec bookstack-db mariadb-dump -u bookstack -p${DB_PASSWORD} bookstack > bookstack-db-migration.sql

# 2. Detener y eliminar contenedor antiguo
docker stop bookstack-db
docker rm bookstack-db

# 3. Eliminar volumen antiguo
docker volume rm bookstack_db

# 4. Recrear con MariaDB 11
docker compose up -d bookstack-db

# 5. Esperar inicialización
sleep 15

# 6. Restaurar datos
cat bookstack-db-migration.sql | docker exec -i bookstack-db mariadb -u bookstack -p${DB_PASSWORD} bookstack

# 7. Iniciar BookStack
docker start bookstack
```

---

## Solución de Problemas

### BookStack no inicia

**Síntomas**: Contenedor se reinicia constantemente

**Diagnóstico**:
```bash
docker logs bookstack
```

**Soluciones**:
- Verificar que MariaDB esté funcionando: `docker logs bookstack-db`
- Comprobar contraseña en `.env`
- Verificar permisos del volumen: `docker exec bookstack ls -la /config`

### Error de conexión a base de datos

**Síntomas**: `SQLSTATE[HY000] [2002] Connection refused`

**Solución**:
```bash
# Verificar que MariaDB esté lista
docker exec bookstack-db mariadb -u bookstack -p${DB_PASSWORD} -e "SELECT 1"

# Reiniciar servicios en orden
docker restart bookstack-db
sleep 10
docker restart bookstack
```

### Error de permisos

**Síntomas**: `PermissionError: [Errno 13] Permission denied`

**Solución**:
```bash
docker run --rm -v bookstack_config:/data alpine chown -R 1000:1000 /data
docker restart bookstack
```

### BookStack lento o no responde

**Diagnóstico**:
```bash
# Ver uso de recursos
docker stats bookstack bookstack-db

# Ver queries lentas en MariaDB
docker exec bookstack-db mariadb -u bookstack -p${DB_PASSWORD} -e "SHOW FULL PROCESSLIST;"
```

**Soluciones**:
- Limpiar caché: `docker exec bookstack php artisan cache:clear`
- Optimizar BD: `docker exec bookstack-db mariadb-optimize -u bookstack -p${DB_PASSWORD} bookstack`
- Regenerar índices de búsqueda: `docker exec bookstack php artisan bookstack:regenerate-search`

### Comandos de emergencia

```bash
# Reiniciar todo el stack
docker restart bookstack bookstack-db

# Ver logs en tiempo real
docker logs -f --tail 100 bookstack

# Acceder a shell de BookStack
docker exec -it bookstack bash

# Ejecutar comandos artisan
docker exec bookstack php artisan tinker

# Limpiar todo el caché
docker exec bookstack php artisan cache:clear
docker exec bookstack php artisan config:clear
docker exec bookstack php artisan view:clear
docker restart bookstack
```

---

## Variables de Entorno

### Requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `DB_PASSWORD` | Contraseña de MariaDB | `generada_con_openssl` |

### Opcionales

| Variable | Descripción | Valor por defecto |
|----------|-------------|-------------------|
| `DB_NAME` | Nombre de la base de datos | `bookstack` |
| `DB_USER` | Usuario de MariaDB | `bookstack` |
| `DOMAIN_HOST` | Dominio (solo Traefik) | `bookstack.example.com` |
| `PUID` / `PGID` | UID/GID del usuario | `1000` |
| `TZ` | Zona horaria | `Europe/Madrid` |

---

## Recursos

- [Documentación oficial de BookStack](https://www.bookstackapp.com/docs/)
- [LinuxServer BookStack Image](https://docs.linuxserver.io/images/docker-bookstack)
- [BookStack GitHub](https://github.com/BookStackApp/BookStack)
- [BookStack Demo](https://demo.bookstackapp.com/)
- [API Documentation](https://demo.bookstackapp.com/api/docs)

---

## Licencia

BookStack es software de código abierto bajo licencia MIT.
