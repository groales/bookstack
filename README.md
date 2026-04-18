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
- Docker Compose instalado
- Red Docker externa `proxy` creada si vas a usar un proxy inverso genérico
- APP_KEY y DB_PASSWORD generadas

⚠️ **IMPORTANTE**: BookStack requiere MariaDB. Este compose incluye el contenedor de base de datos.

## Generar Claves y Contraseñas

**Antes de cualquier despliegue**, genera las claves necesarias:

```bash
# APP_KEY (BookStack)
docker run -it --rm --entrypoint /bin/bash lscr.io/linuxserver/bookstack:latest appkey

# DB_PASSWORD (MariaDB)
openssl rand -base64 32
```

Guarda los resultados, los necesitarás en el archivo `.env`.

> ⚠️ **Importante**: Usa comillas simples en el archivo `.env` si la contraseña contiene caracteres especiales.
> Ejemplo: `DB_PASSWORD='tu_password_generado'`

---

## Despliegue con Docker Compose

### 1. Crear Directorio y Archivos

```bash
# Crear directorio
mkdir bookstack
cd bookstack
```

### 2. Crear docker-compose.yml

Crea el archivo `docker-compose.yml`:

```yaml
services:
  bookstack:
    container_name: bookstack
    image: lscr.io/linuxserver/bookstack:latest
    restart: unless-stopped
    ports:
      - 6875:80
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Europe/Madrid
      APP_URL: ${APP_URL}
      APP_KEY: ${APP_KEY}
      DB_HOST: bookstack-db
      DB_PORT: 3306
      DB_DATABASE: ${DB_NAME:-bookstack}
      DB_USERNAME: ${DB_USER:-bookstack}
      DB_PASSWORD: ${DB_PASSWORD}
    volumes:
      - bookstack_config:/config
    networks:
      - proxy
      - default
    depends_on:
      - bookstack-db

  bookstack-db:
    container_name: bookstack-db
    image: mariadb:12
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_NAME:-bookstack}
      MYSQL_USER: ${DB_USER:-bookstack}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
    volumes:
      - bookstack_db:/var/lib/mysql

volumes:
  bookstack_config:
    name: bookstack_config
  bookstack_db:
    name: bookstack_db

networks:
  default:
  proxy:
    external: true
```

### 3. Generar APP_KEY y Contraseña

**APP_KEY** (requerido por BookStack):

```bash
docker run --rm lscr.io/linuxserver/bookstack:latest php /app/www/artisan key:generate --show
```

Salida esperada: `base64:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=`

**DB_PASSWORD** (contraseña de MariaDB):

```bash
openssl rand -base64 32
```

### 4. Configurar Variables de Entorno

Crea el archivo `.env`:

```env
# URL final de acceso a BookStack
# Ejemplo local:  http://localhost:6875
# Ejemplo con proxy: https://bookstack.midominio.com
APP_URL=http://localhost:6875

# Claves de Seguridad (GENERAR NUEVAS)
APP_KEY=base64:tu_clave_generada
DB_PASSWORD=tu_password_generado

# Base de datos (valores por defecto)
DB_NAME=bookstack
DB_USER=bookstack
```

### 5. Desplegar

```bash
# Crear red proxy si no existe
docker network create proxy

# Iniciar servicios
docker compose up -d

# Ver logs
docker compose logs -f bookstack
```

La inicialización puede tardar **30-60 segundos** (MariaDB + BookStack migrations).

---

## Método Alternativo: Clonar desde Git

Si prefieres usar Git para mantener la configuración actualizada:

```bash
# Clonar repositorio
git clone https://github.com/groales/bookstack.git
cd bookstack

# Copiar ejemplo de variables
cp .env.example .env
nano .env  # Editar con tus claves

# Crear red proxy si no existe
docker network create proxy

# Desplegar
docker compose up -d
```

---

## Acceso y Credenciales Iniciales

**URL de acceso**:
- Local: `http://localhost:6875`
- Con proxy inverso genérico: la URL que hayas definido en `APP_URL`

**Credenciales por defecto** (⚠️ **CAMBIAR INMEDIATAMENTE**):
- Email: `admin@admin.com`
- Contraseña: `password`

**Tras el primer login**:
1. Click en avatar → **Edit Profile**
2. **Change Password** → Establecer contraseña segura
3. Cambiar email si es necesario

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
docker pull mariadb:12

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

# 4. Recrear con MariaDB 12
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
| `APP_URL` | URL final de acceso a BookStack | `http://localhost:6875` |
| `APP_KEY` | Clave de encriptación Laravel | `base64:generada_con_docker` |
| `DB_PASSWORD` | Contraseña de MariaDB | `generada_con_openssl` |

### Opcionales

| Variable | Descripción | Valor por defecto |
|----------|-------------|-------------------|
| `DB_NAME` | Nombre de la base de datos | `bookstack` |
| `DB_USER` | Usuario de MariaDB | `bookstack` |
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
