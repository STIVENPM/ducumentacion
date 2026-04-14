# ADR-005: Estrategia de contenedorización con Docker Compose para PostgreSQL y Liquibase

- **Estado:** Propuesto
- **Fecha:** 2025-04-14
- **Autor:** Aprendiz SENA – ADSO

---

## Contexto

El proyecto requiere que cualquier desarrollador pueda levantar el entorno de base de datos de forma reproducible, sin depender de instalaciones locales de PostgreSQL ni configuraciones manuales. Adicionalmente, Liquibase debe poder ejecutarse en el mismo entorno de manera integrada para aplicar los changelogs automáticamente al iniciar el sistema. El equipo de SENA trabaja en distintos sistemas operativos (Windows, Linux, macOS), lo que hace crítica la portabilidad del entorno.

## Problema

Sin contenedorización:
- Cada desarrollador debe instalar y configurar PostgreSQL manualmente en su máquina.
- Las versiones de PostgreSQL pueden diferir entre desarrolladores, generando inconsistencias.
- Liquibase requiere instalación y configuración local adicional.
- No hay garantía de que el entorno de desarrollo sea equivalente al entorno de QA o producción.
- Compartir el proyecto entre miembros del equipo implica documentar pasos de instalación complejos que con frecuencia quedan desactualizados.

## Decisión

Se adopta **Docker y Docker Compose** como estrategia oficial de contenedorización del entorno de base de datos. El proyecto incluirá un archivo `docker-compose.yml` que levanta dos servicios coordinados:

1. **PostgreSQL** – servidor de base de datos.
2. **Liquibase** – servicio que aplica los changelogs al iniciar, esperando a que PostgreSQL esté listo.

### Archivo `docker-compose.yml`

```yaml
version: '3.9'

services:

  postgres:
    image: postgres:16-alpine
    container_name: airline_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: airline
      POSTGRES_USER: airline_admin
      POSTGRES_PASSWORD: changeme_local
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U airline_admin -d airline"]
      interval: 10s
      timeout: 5s
      retries: 5

  liquibase:
    image: liquibase/liquibase:4.27
    container_name: airline_liquibase
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./liquibase:/liquibase/changelog
    command: >
      --url=jdbc:postgresql://postgres:5432/airline
      --username=airline_admin
      --password=changeme_local
      --changeLogFile=changelog/changelog-master.xml
      update

volumes:
  postgres_data:
    driver: local
```

### Archivo `docker/README.md`

Instrucciones para el equipo:

```markdown
## Levantar el entorno de base de datos

### Prerrequisitos
- Docker Desktop instalado (Windows/macOS) o Docker Engine (Linux)
- Docker Compose v2+

### Comandos

# Levantar PostgreSQL y aplicar changelogs
docker compose up -d

# Ver logs de Liquibase para verificar que los cambios se aplicaron
docker compose logs liquibase

# Detener los contenedores (los datos persisten en el volumen)
docker compose down

# Detener y eliminar todos los datos (reset completo)
docker compose down -v

# Conectarse a la base de datos
docker exec -it airline_db psql -U airline_admin -d airline
```

### Variables de entorno por entorno

Se usan archivos `.env` por entorno para no hardcodear credenciales:

```
# .env.local (ignorado por .gitignore)
POSTGRES_USER=airline_admin
POSTGRES_PASSWORD=changeme_local
POSTGRES_DB=airline

# .env.qa (ignorado por .gitignore, distribuido de forma segura)
POSTGRES_USER=airline_qa_user
POSTGRES_PASSWORD=qa_secure_password
POSTGRES_DB=airline_qa
```

El `docker-compose.yml` referencia estas variables con `${POSTGRES_USER}` en lugar de valores directos. Los archivos `.env.*` **nunca se suben al repositorio**.

### `.gitignore` recomendado

```gitignore
# Variables de entorno con credenciales
.env
.env.local
.env.qa
.env.prod

# Datos generados por Docker localmente
docker/data/

# Archivos de sistema
.DS_Store
Thumbs.db
```

## Justificación técnica

- **Docker Compose** es el estándar de la industria para definir entornos de desarrollo multi-servicio. Su curva de aprendizaje es baja y la documentación es extensa.
- Usar `depends_on` con `condition: service_healthy` garantiza que Liquibase no intenta conectarse a PostgreSQL antes de que este esté listo, eliminando condiciones de carrera frecuentes en entornos de contenedores.
- El volumen `postgres_data` persiste los datos entre reinicios del contenedor, simulando el comportamiento de un servidor real. El comando `down -v` permite un reset limpio cuando se necesita.
- Usar `postgres:16-alpine` reduce el tamaño de la imagen base sin perder funcionalidad. Alpine es la variante más usada en entornos de contenedores de producción.
- Mantener Liquibase como servicio separado (no como parte del contenedor de PostgreSQL) respeta el principio de responsabilidad única: cada contenedor hace una sola cosa.
- Los archivos `.env` por entorno evitan que las credenciales queden en el historial de Git, lo cual es un requisito básico de seguridad.

## Consecuencias e impacto esperado

| Aspecto | Impacto |
|---|---|
| Reproducibilidad | Cualquier desarrollador levanta el entorno con `docker compose up -d` en menos de 2 minutos |
| Portabilidad | Funciona igual en Windows, macOS y Linux |
| Integración con Liquibase | Los changelogs se aplican automáticamente al levantar el entorno |
| Seguridad | Las credenciales no van al repositorio; se gestionan con archivos `.env` |
| Aislamiento | El entorno de desarrollo no afecta otras instalaciones locales de PostgreSQL |
| Deuda técnica | Requiere que todos los miembros del equipo instalen Docker. En equipos sin acceso a Docker, se debe evaluar alternativa con `podman` |
