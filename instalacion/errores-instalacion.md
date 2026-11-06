# Errores encontrados en la instalación

Instalación de **Odoo 19.0 Community con Docker Compose** en Windows 11. El proceso completo está en la diapositiva [`instalacion-odoo.pdf`](instalacion-odoo.pdf).

## Comprobaciones previas

| Comprobación | Comando | Resultado |
|---|---|---|
| Docker funciona | `docker version` | Cliente y servidor 29.9.2 |
| Compose disponible | `docker compose version` | v5.6.1 |
| Puerto libre | `Get-NetTCPConnection -LocalPort 8069` | Sin conexiones |
| Archivo válido | `docker compose config --quiet` | Sin errores |
| No hay contenedores previos | `docker ps -a` | Lista vacía |

## Tabla de errores

Los errores 1 y 5 aparecieron durante el trabajo normal. El 2 y el 4 se reprodujeron a propósito. El 3 no estaba previsto: apareció al reproducir el 2.

| # | Error o mensaje | Causa | Solución aplicada | Fuente consultada |
|---|---|---|---|---|
| 1 | `option addons_path, invalid addons directory '/mnt/extra-addons', skipped` | La carpeta `addons` estaba vacía. Odoo solo acepta una carpeta si contiene al menos un módulo con `__init__.py` y `__manifest__.py`. | Es un aviso sin consecuencias. La carpeta se comprueba al arrancar, así que hay que ejecutar `docker restart odoo-web` después de añadir el primer módulo. | Código de la imagen: `odoo/tools/config.py`, función `_is_addons_path`. [Imagen oficial de Odoo](https://hub.docker.com/_/odoo) |
| 2 | `Bind for 0.0.0.0:8069 failed: port is already allocated` | **Reproducido:** otro contenedor ya publicaba el puerto 8069. | Localizar al ocupante con `docker ps --filter "publish=8069"`, pararlo con `docker stop` y volver a ejecutar `docker compose up -d`. | [`docker container ls`](https://docs.docker.com/reference/cli/docker/container/ls/) y [`docker compose up`](https://docs.docker.com/reference/cli/docker/compose/up/) |
| 3 | `Database connection failure: could not translate host name "db" to address` y puerto mostrado como `8069/tcp`, sin publicar | **Consecuencia del error 2:** tras el arranque fallido, `odoo-web` se quedó sin red. `docker inspect` mostraba `Networks: {}` y 8 reinicios seguidos. Sin red no resuelve `db` ni publica el puerto. | `docker network connect --alias odoo --alias odoo-web documentos_default odoo-web` y después `docker restart odoo-web`. Resultado: HTTP 200, 0 reinicios y datos intactos. | Diagnóstico con `docker inspect`. [`docker network connect`](https://docs.docker.com/reference/cli/docker/network/connect/) |
| 4 | Con PostgreSQL parado no sale «Internal Server Error»: `/web/login` **redirige a `/web/database/selector`** con HTTP 200, como si no hubiera bases de datos. En el log: `psycopg2.OperationalError: connection to server at "db" ... FATAL: the database system is shutting down` | **Reproducido** con `docker stop odoo-db`. Odoo no puede listar las bases de datos y muestra el gestor de bases de datos. | No crear nada en ese formulario. Mirar `docker ps`, arrancar con `docker start odoo-db` y volver a `/web/login`. Los datos seguían intactos. | Log de `odoo-web`. [Imagen oficial de PostgreSQL](https://hub.docker.com/_/postgres) |
| 5 | «Validar» abría el asistente de SMS `confirm.stock.sms` y la entrega seguía en estado *Listo* | La confirmación por SMS estaba activa (`stock_text_confirmation`) y la validación no termina hasta responder al asistente. | Se desactivó el SMS en la compañía y se validó de nuevo. **Efecto en la venta:** se facturó antes de entregar y salió una factura parcial; ver [`proceso-venta.md`](../venta/proceso-venta.md#incidencia-factura-parcial). | Código de la imagen: `stock_sms/models/stock_picking.py` y `stock/models/res_company.py` |

## Riesgos detectados al revisar la instalación

| Hallazgo | Riesgo | Estado |
|---|---|---|
| El registro libre estaba activo (`auth_signup.invitation_scope = b2c`) y el login mostraba «¿No tiene una cuenta?» | Cualquiera podía crearse una cuenta de portal | **Resuelto:** valor `b2b`. El enlace desaparece y `/web/signup` responde 404. |
| La contraseña maestra se guarda en `/etc/odoo/odoo.conf`, dentro del contenedor y fuera de cualquier volumen | Recrear el contenedor la perdería y el gestor de bases de datos volvería al valor por defecto. Por eso el error 3 no se resolvió con `--force-recreate`. | **Pendiente:** antes de recrear el contenedor, guardar la configuración en un volumen. |
| `docker-compose.yml` conserva la contraseña de ejemplo de PostgreSQL | El puerto 5432 no está publicado y solo se llega desde los contenedores, así que el riesgo es bajo en local | **Pendiente:** cambiarla si el sistema sale del entorno local. |
| El puerto 8069 está publicado en todas las interfaces | Otros equipos de la red podrían entrar si el firewall lo permite | **Aceptado** para este ejercicio local. |

## Fuentes de requisitos

- [Instalación desde código fuente de Odoo 19](https://www.odoo.com/documentation/19.0/administration/on_premise/source.html): Python 3.10 o superior y PostgreSQL 13 o superior.
- [Despliegue de Odoo 19](https://www.odoo.com/documentation/19.0/administration/on_premise/deploy.html): 1 worker por cada 6 usuarios concurrentes.
- [Imagen oficial de Odoo](https://hub.docker.com/_/odoo): variables `HOST`, `USER` y `PASSWORD`, y volúmenes `/var/lib/odoo` y `/mnt/extra-addons`.
