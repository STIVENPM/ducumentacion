# ADR-004: Estrategia de versionamiento del repositorio con ramas develop, qa y main

- **Estado:** Propuesto
- **Fecha:** 2025-04-14
- **Autor:** Aprendiz SENA – ADSO

---

## Contexto

El proyecto está siendo construido en un repositorio Git. A medida que el equipo (o el aprendiz) avance en la estabilización del modelo de base de datos, será necesario gestionar cambios en paralelo: trabajo en nuevas funcionalidades, pruebas de integración y versiones estables del esquema. Sin una estrategia de ramas clara, los cambios pueden mezclarse, los entornos pueden quedar inconsistentes y es imposible garantizar que lo que se despliega en producción fue previamente validado.

## Problema

Un repositorio sin estructura de ramas definida presenta los siguientes riesgos:

- Cambios experimentales pueden afectar el esquema que otros están usando como base.
- No existe distinción entre código en desarrollo, código validado y código estable.
- No es posible hacer seguimiento de qué versión del DDL corresponde a cada entorno.
- Liquibase no tiene un punto de referencia claro para saber qué changelog aplicar en cada entorno.

## Decisión

Se adopta una estrategia de tres ramas permanentes basada en **GitFlow simplificado**, adaptada a las necesidades del proyecto de base de datos:

### Ramas permanentes

| Rama | Propósito | Entorno destino |
|---|---|---|
| `main` | Versión estable y validada del proyecto | Producción / entrega final |
| `qa` | Versión integrada lista para pruebas | Entorno de QA / validación |
| `develop` | Rama de integración del trabajo activo | Entorno local de desarrollo |

### Ramas de trabajo temporales

Adicionalmente se usan ramas temporales que nacen de `develop` y se fusionan de regreso a ella:

| Tipo de rama | Nomenclatura | Ejemplo |
|---|---|---|
| Historia de usuario | `feature/HU-XXX-descripcion` | `feature/HU-005-changelogs-por-dominio` |
| Corrección de error | `fix/descripcion-corta` | `fix/ruta-rota-changelog` |
| Documentación | `docs/descripcion` | `docs/adr-002-roles` |

### Flujo de trabajo estándar

```
[develop] → [feature/HU-XXX] → trabajar aquí
                             ↓
                         Pull Request → merge a [develop]
                                                ↓
                                       cuando está integrado → merge a [qa]
                                                                        ↓
                                                           pruebas OK → merge a [main]
```

Pasos detallados:

1. **Crear rama desde develop:**
   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b feature/HU-005-changelogs-dominio
   ```

2. **Trabajar en la rama de feature.** Commits frecuentes con mensajes descriptivos:
   ```bash
   git add liquibase/changelogs/01-geografia/
   git commit -m "feat: agrega changelog de dominio geografia con rollback"
   ```

3. **Merge a develop (vía Pull Request):**
   - Se revisa el trabajo antes de integrar.
   - Se resuelven conflictos en la rama de feature, no en `develop`.

4. **Promoción a qa:**
   ```bash
   git checkout qa
   git merge develop
   git push origin qa
   ```
   En QA se ejecuta Liquibase contra el entorno de pruebas y se validan los changelogs.

5. **Promoción a main (solo cuando QA aprueba):**
   ```bash
   git checkout main
   git merge qa
   git tag v1.0.0
   git push origin main --tags
   ```

### Reglas del flujo

- **Nadie hace commits directamente a `main` ni a `qa`.** Solo se llega a estas ramas mediante merge controlado.
- **`develop` es la rama de integración.** Siempre debe estar en un estado funcional (los changelogs aplicados deben levantar la base sin errores).
- **Cada HU tiene su propia rama.** Las ramas de feature no deben acumular cambios de múltiples historias.
- **Los tags en `main`** siguen versionado semántico: `v{MAJOR}.{MINOR}.{PATCH}`.
- **El archivo `changelog-master.xml`** solo se modifica cuando se agrega un nuevo dominio. Los cambios dentro de un dominio van en el changelog del dominio.

### Protecciones recomendadas en el repositorio

Si se usa GitHub o GitLab, configurar:
- **Branch protection en `main`:** Requiere Pull Request + aprobación antes de merge.
- **Branch protection en `qa`:** Requiere que el merge venga desde `develop` o feature ya integrado.
- **Checks requeridos:** Que Liquibase ejecute sin errores antes de autorizar el merge a `qa`.

## Justificación técnica

- Separar `develop`, `qa` y `main` garantiza que el código que llega a `main` ya pasó por al menos dos revisiones: la del desarrollador al integrar a `develop` y la de pruebas al promover a `qa`.
- Las ramas de feature por HU permiten trabajar en paralelo sin bloquear la línea principal.
- Los tags semánticos en `main` crean un historial claro de versiones del esquema, lo cual es especialmente importante para Liquibase: permite saber exactamente qué changesets corresponden a cada versión estable.
- Este modelo es compatible con pipelines de CI/CD que ejecuten Liquibase automáticamente al hacer push a `qa` o merge a `main`.

## Consecuencias e impacto esperado

| Aspecto | Impacto |
|---|---|
| Trazabilidad | Cada cambio al esquema está vinculado a una rama, HU y commit específico |
| Calidad | Los cambios pasan por validación antes de llegar a `main` |
| Colaboración | Varios desarrolladores pueden trabajar en paralelo sin conflictos en `develop` |
| Rollback de repositorio | Es posible regresar a cualquier tag estable si un cambio en `main` genera problemas |
| Disciplina de equipo | Requiere que todos respeten el flujo; sin eso, la estrategia pierde valor |
