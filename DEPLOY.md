# Deploy de este fork

Este fork de [karakeep-app/karakeep](https://github.com/karakeep-app/karakeep) está desplegado en **links.24h.cloud** mediante Coolify (panel en `cool.24h.cloud`).

Este archivo NO existe en upstream — vive sólo en este fork, así no genera conflictos al sincronizar.

## Estrategia de ramas

| Rama | Propósito | Coolify la observa |
|---|---|---|
| `main` | Espejo del upstream. Aquí mergeamos `karakeep-app/karakeep:main`. | No |
| `production` | Lo que está en links.24h.cloud. Sólo recibe merges desde `main` o desde feature branches estables. | **Sí (auto-deploy on push)** |
| `feature/*` | Cambios propios en curso. Se mergean a `production` cuando estén listos. | No |

Regla de oro: **nunca pushear directamente a `production` algo no probado**. Si algo se rompe, los datos están a salvo en volúmenes Docker named, pero la app cae hasta el siguiente deploy bueno.

## Sincronizar con upstream

Cuando karakeep saque una nueva versión y quieras incorporarla:

```bash
# Estás en local
git checkout main
git fetch upstream
git merge upstream/main          # o `git rebase upstream/main` si prefieres historia lineal
git push origin main

# Llevar los cambios a producción:
git checkout production
git merge main                   # esto puede generar conflictos si tenías cambios propios
git push origin production       # dispara auto-deploy en Coolify
```

Si no quieres exponerte a inestabilidad de `main`, sincroniza desde un tag estable:

```bash
git fetch upstream --tags
git checkout main
git merge v0.X.Y                 # tag específico
```

## Añadir una feature propia

```bash
git checkout main                # base desde upstream actualizado
git pull
git checkout -b feature/mi-cambio
# ...trabaja, commitea...
git push origin feature/mi-cambio
# Cuando esté listo:
git checkout production
git merge feature/mi-cambio
git push origin production       # auto-deploy
```

Para contribuir al upstream: abre PR desde `feature/mi-cambio` (en tu fork) hacia `karakeep-app/karakeep:main`.

## Variables de entorno y secretos

NO se commiten al repo. Viven en Coolify → recurso Karakeep → **Environment Variables**:

- `NEXTAUTH_URL`, `NEXTAUTH_SECRET`, `MEILI_MASTER_KEY` — definidos al instalar
- `OPENAI_API_KEY` — añadirla en Coolify cuando quieras activar IA
- `DISABLE_SIGNUPS=true` después de crear el primer usuario admin

Para añadir/cambiar variables: Coolify UI → recurso → Environment Variables → editar → "Apply" (reinicia el contenedor `web` en ~30s, sin perder datos).

## Volúmenes persistentes (DATOS)

Los volúmenes Docker named contienen TODO lo que es valioso:

| Volumen | Contenido |
|---|---|
| `mf8r638h59oavvtraz160pnc_data` | SQLite db + assets (screenshots, páginas archivadas) |
| `mf8r638h59oavvtraz160pnc_meilisearch` | Índice de búsqueda (regenerable pero lento) |

**No borrar nunca.** Si haces algún `docker compose down -v` o equivalente, perderás todo. Backups: ver sección al final.

## Endurecimiento de red

Por defecto Docker bypasses UFW y expone los puertos publicados a Internet. En este VPS hay reglas iptables en `DOCKER-USER` que bloquean externamente:

| Puerto | Servicio | Por qué bloquearlo |
|---|---|---|
| 3000 | Karakeep web | Sólo accesible vía Traefik en 443 |
| 6001-6002 | Coolify realtime | Sólo necesario internamente |
| 8000 | Coolify UI | Sólo accesible vía Traefik en 443 |

Estas reglas se aplican automáticamente al boot mediante:

- `/usr/local/sbin/karakeep-docker-firewall.sh` — script idempotente con las reglas
- `/etc/systemd/system/karakeep-docker-firewall.service` — systemd oneshot que corre after docker.service

Si añades un nuevo servicio en Coolify que publica un puerto que no quieres exponer al mundo, edita el script para incluirlo y reinicia con `systemctl restart karakeep-docker-firewall.service`.

## Coexistencia con GitHub Actions self-hosted runner

En este VPS también corre un runner self-hosted bajo el servicio `actions.runner.RodolfoMiguelLopez-edukids-Cloud.edukids-ci-runner-01.service`. No hay conflicto con Karakeep ni Coolify porque:

- El runner usa long-polling saliente a GitHub (egress, no inbound)
- Sus jobs corren bajo el usuario `actions`, no como `root`
- No comparten puertos ni volúmenes Docker

Si en el futuro el runner empieza a usar Docker (por ej. para integration tests), valida que no hayan conflictos con la red `coolify-proxy` o con las reglas iptables.

## Backups

Pendiente de implementar. Cuando los datos sean valiosos, considera:

1. **Snapshot diario del VPS** desde el panel del proveedor (lo más rápido)
2. **restic** + bucket S3-compatible para los volúmenes Docker named

La DB es SQLite — para snapshot consistente, parar el contenedor `web` o usar `sqlite3 .backup`.

## Acceso a logs

Coolify UI tiene logs en tiempo real, pero para casos de debug:

```bash
ssh root@<vps>
docker logs web-mf8r638h59oavvtraz160pnc-XXX --tail 200 -f
docker logs meilisearch-mf8r638h59oavvtraz160pnc-XXX --tail 100
```

Los IDs con sufijo cambian en cada deploy. Listado actual: `docker ps --format '{{.Names}}'`.
