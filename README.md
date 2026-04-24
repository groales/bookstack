# BookStack

Plataforma de documentación y wiki de código abierto para organizar contenido en libros, capítulos y páginas con editor WYSIWYG y Markdown.

Referencia oficial de instalación: https://docs.linuxserver.io/images/docker-bookstack

## Características

- 📚 **Organización jerárquica**: Libros, capítulos y páginas para estructurar documentación
- ✏️ **Editor dual**: WYSIWYG y Markdown en la misma plataforma
- 🔍 **Búsqueda integrada**: Localiza contenido rápidamente
- 🔐 **Control de acceso**: Roles y permisos por usuario o equipo
- 📝 **Historial de cambios**: Registro completo de revisiones
- 🔗 **Integraciones**: LDAP, SAML y OAuth disponibles
- 🌍 **Multiidioma**: Soporte para distintos idiomas y localizaciones

## Requisitos Previos

- Docker Engine instalado
- Docker Compose instalado
- Red Docker externa `proxy` creada si vas a publicar BookStack detrás de un proxy inverso genérico
- Variables `APP_KEY` y `DB_PASSWORD` generadas antes del despliegue

> ⚠️ **Importante**: Este `compose.yaml` ya incluye MariaDB, por lo que BookStack y la base de datos se despliegan juntos.

## Archivos de este Repositorio

Este repositorio contiene:

- `compose.yaml` - Stack base de BookStack y MariaDB
- `.env.example` - Plantilla de variables de entorno
- `README.md` - Esta documentación
- `config/` - Persistencia local de BookStack (se crea al arrancar)

---

## Generar Claves Seguras

Antes del primer arranque, genera los valores necesarios:

```bash
# APP_KEY de BookStack
docker run --rm lscr.io/linuxserver/bookstack:latest php /app/www/artisan key:generate --show

# Contraseña de MariaDB
openssl rand -base64 32
```

Guarda ambos valores para usarlos en `.env`.

> 💡 **Tip**: Si `DB_PASSWORD` contiene caracteres especiales, mantenla entre comillas simples en el archivo `.env`.

---

## Despliegue con Docker Compose

### 1. Clonar el repositorio

```bash
git clone https://github.com/groales/bookstack.git
cd bookstack
```

### 2. Preparar el archivo `.env`

```bash
cp .env.example .env
```

Contenido esperado:

```env
APP_URL=http://localhost:6875
APP_KEY='base64:tu_clave_generada'
DB_PASSWORD='tu_password_generado'

DB_NAME=bookstack
DB_USER=bookstack
```

### 3. Crear la red externa si vas a usar proxy

```bash
docker network create proxy
```

Si no la necesitas, elimina o comenta la red `proxy` en `compose.yaml` antes de arrancar.

### 4. Desplegar

```bash
docker compose up -d
```

Para seguir el arranque:

```bash
docker compose logs -f bookstack
```

La inicialización puede tardar entre 30 y 60 segundos porque BookStack espera a que MariaDB quede operativa.

---

## Acceso Inicial

Una vez desplegado, accede a BookStack usando la URL definida en `APP_URL`.

```text
http://localhost:6875
# o
https://bookstack.midominio.com
```

### Credenciales Iniciales

Credenciales por defecto de BookStack:

- Email: `admin@admin.com`
- Contraseña: `password`

### Primera Configuración

1. Inicia sesión con las credenciales por defecto.
2. Cambia la contraseña del administrador inmediatamente.
3. Ajusta nombre, idioma y branding desde `Settings`.
4. Revisa registro público, roles y permisos antes de abrir acceso a otros usuarios.

---

## Comandos Útiles

### Ver logs

```bash
docker compose logs -f bookstack
docker compose logs -f bookstack-db
```

### Reiniciar servicios

```bash
docker compose restart bookstack
docker compose restart bookstack-db
```

### Ejecutar comandos de mantenimiento

```bash
docker exec bookstack php artisan list
docker exec bookstack php artisan cache:clear
docker exec bookstack php artisan bookstack:regenerate-search
```

### Actualizar contenedores

```bash
docker compose pull
docker compose up -d
```

---

## Estructura de Persistencia

```text
Persistencia:
├── ./config      -> /config
└── bookstack_db  -> /var/lib/mysql
```

---

## Configuración Avanzada

### Variables de Entorno

| Variable | Descripción | Valor por defecto |
|----------|-------------|-------------------|
| `APP_URL` | URL final de acceso a BookStack | `http://localhost:6875` |
| `APP_KEY` | Clave de aplicación Laravel | Sin valor por defecto |
| `DB_PASSWORD` | Contraseña de MariaDB | Sin valor por defecto |
| `DB_NAME` | Nombre de la base de datos | `bookstack` |
| `DB_USER` | Usuario de base de datos | `bookstack` |

### LDAP, SAML y OAuth

BookStack soporta integración con proveedores externos de autenticación. Si vas a usar LDAP, añade variables como estas al entorno:

```env
LDAP_SERVER=ldap.example.com:389
LDAP_BASE_DN=dc=example,dc=com
LDAP_DN=cn=bookstack,ou=Services,dc=example,dc=com
LDAP_PASS=password
LDAP_USER_FILTER=(&(uid=${user}))
LDAP_VERSION=3
```

Después reinicia el contenedor:

```bash
docker compose restart bookstack
```

Para SAML u OAuth, consulta la documentación oficial de administración:
https://www.bookstackapp.com/docs/admin/third-party-auth/

---

## Solución de Problemas

### BookStack no inicia

```bash
docker logs bookstack
docker logs bookstack-db
```

Verifica especialmente:

- que MariaDB esté arrancando correctamente
- que `DB_PASSWORD` coincida entre contenedor y `.env`
- que los volúmenes tengan permisos correctos

### Error de conexión a base de datos

```bash
docker exec bookstack-db mariadb -u bookstack -p${DB_PASSWORD} -e "SELECT 1"
```

Si la base responde, reinicia BookStack:

```bash
docker compose restart bookstack
```

### Problemas de permisos

```bash
docker run --rm -v $(pwd)/config:/data alpine chown -R 1000:1000 /data
docker compose restart bookstack
```

---

## Seguridad

### Recomendaciones

1. Cambia las credenciales administrativas en el primer acceso.
2. Publica el servicio detrás de HTTPS si habrá acceso remoto.
3. Desactiva el registro público si no es estrictamente necesario.
4. Haz backups periódicos de `./config` y `bookstack_db`.
5. Mantén las imágenes actualizadas.

---

## Backup y Restauración

### Backup

```bash
docker exec bookstack-db mariadb-dump -u bookstack -p${DB_PASSWORD} bookstack > bookstack-backup-$(date +%Y%m%d).sql

docker run --rm -v $(pwd)/config:/backup -v $(pwd):/target alpine \
  tar czf /target/bookstack-config-$(date +%Y%m%d).tar.gz -C /backup .
```

### Restauración

```bash
docker compose stop bookstack

cat bookstack-backup-YYYYMMDD.sql | docker exec -i bookstack-db \
  mariadb -u bookstack -p${DB_PASSWORD} bookstack

docker run --rm -v $(pwd)/config:/restore -v $(pwd):/source alpine \
  tar xzf /source/bookstack-config-YYYYMMDD.tar.gz -C /restore

docker compose start bookstack
```

---

## Recursos

- Documentación oficial LinuxServer: https://docs.linuxserver.io/images/docker-bookstack
- Documentación oficial BookStack: https://www.bookstackapp.com/docs/
- Autenticación externa: https://www.bookstackapp.com/docs/admin/third-party-auth/
