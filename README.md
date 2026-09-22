# aurex-oracle-launcher

Reintenta, cada 5 minutos vía GitHub Actions (sin depender de ningún computador
encendido), lanzar una instancia ARM Always Free (`VM.Standard.A1.Flex`, 1
OCPU / 6 GB) en Oracle Cloud, región `sa-bogota-1`. Corre **en paralelo** con el
script local `~/.oci/aurex_retry_launch.sh` (a propósito, no como reemplazo):
el script local reintenta con mucha menos distancia entre intentos (90 s vs.
las corridas reales de Actions, que llegan más espaciadas por el throttling de
GitHub — ver abajo), y este workflow sirve de respaldo para cuando el Mac está
apagado o sin red.

## Cómo funciona

- `on.schedule` dispara el workflow cada 5 minutos (mínimo práctico de GitHub
  Actions — aunque el cron admitiera algo más frecuente, GitHub trata los
  schedules de alta frecuencia como "mejor esfuerzo" y tiende a saltarlos bajo
  carga).
- Cada corrida hace **un solo intento** de `oci compute instance launch`
  (a diferencia del script local, que hacía un loop de reintentos cada 90 s
  dentro de un mismo proceso).
- Si Oracle responde "sin capacidad" (o cualquier error transitorio), la
  corrida termina en verde (no es una falla real) y el siguiente cron vuelve
  a intentar.
- Si Oracle responde con un error de autenticación/configuración real (no de
  capacidad), la corrida falla a propósito — para que se note que algo hay
  que corregir, en vez de reintentar por siempre contra un error que nunca se
  va a resolver solo.
- Si se consigue lanzar la instancia: se abre un **Issue** en este repo con
  los datos (incluida la IP pública) y el workflow **se desactiva solo**
  (`gh workflow disable`) para no seguir intentando ni gastar minutos.

### Rotación de fault domain

`sa-bogota-1` tiene un solo dominio de disponibilidad (AD), así que no hay
forma de repartir los intentos por AD — y cambiar de región perdería la
elegibilidad Always Free (atada a la región *home* del tenant). Lo único
diversificable dentro de esa restricción es el **fault domain**: cada AD
tiene 3 (`FAULT-DOMAIN-1/2/3`, agrupaciones de hardware distintas dentro del
mismo AD). Cada corrida usa `GITHUB_RUN_NUMBER` (que GitHub incrementa 1 por
corrida, automático) para rotar: corrida 1 → `FAULT-DOMAIN-1`, corrida 2 →
`FAULT-DOMAIN-2`, corrida 3 → `FAULT-DOMAIN-3`, corrida 4 → de vuelta a
`FAULT-DOMAIN-1`, etc. El script local hace la misma rotación por su propio
número de intento — así entre los dos se cubren los 3 fault domains sin
coordinarse entre sí. No hay garantía de que la escasez sea por fault domain
y no por el AD completo, pero no cuesta nada intentarlo.

## Por qué un repo nuevo y público

Los repos públicos tienen minutos de GitHub Actions **ilimitados y gratis**;
los privados solo dan 2.000 min/mes gratis, que un cron cada 5 min agotaría
en pocos días. Este repo no contiene nada del código de negocio de Aurex,
solo el script de reintento — no hay nada sensible en el código en sí (las
credenciales viven como *secrets*, nunca en el repo).

## Configuración necesaria (secrets del repo)

`Settings → Secrets and variables → Actions → New repository secret`:

| Secret | De dónde sale |
|---|---|
| `OCI_USER_OCID` | `user` en `~/.oci/config` |
| `OCI_FINGERPRINT` | `fingerprint` en `~/.oci/config` |
| `OCI_TENANCY_OCID` | `tenancy` en `~/.oci/config` |
| `OCI_REGION` | `region` en `~/.oci/config` (`sa-bogota-1`) |
| `OCI_PRIVATE_KEY` | contenido completo de `~/.oci/oci_api_key.pem` |
| `OCI_COMPARTMENT_ID` | compartment (en este caso, el mismo tenancy) |
| `OCI_SUBNET_ID` | subnet donde se lanza la instancia |
| `OCI_IMAGE_ID` | imagen del sistema operativo |
| `OCI_AD` | dominio de disponibilidad, ej. `xxxx:SA-BOGOTA-1-AD-1` |
| `SSH_PUBLIC_KEY` | contenido de `~/.ssh/aurex_oracle.pub` |

No hace falta tocar `GITHUB_TOKEN` — GitHub lo inyecta solo, con los permisos
declarados en `permissions:` del workflow (`issues: write`, `actions: write`).

## Probarlo manualmente

`Actions → Reintento instancia Oracle ARM → Run workflow` (dispara sin
esperar al cron). Revisa el log de la corrida: si dice "sin capacidad
todavía", todo está bien configurado y solo falta que Oracle libere un hueco.

## Si se consigue la instancia

1. Llega un Issue nuevo en este repo con la IP pública y el JSON completo de
   la respuesta de OCI.
2. El workflow queda desactivado (no se borra, por si se necesita reactivar
   para lanzar OTRA instancia en el futuro:
   `Actions → Reintento instancia Oracle ARM → ⋯ → Enable workflow`).
3. Conectarse por SSH: `ssh -i ~/.ssh/aurex_oracle ubuntu@<IP_PUBLICA>`
   (o `opc@` según la imagen usada).

## Si algo se rompe

Una corrida que falla "de verdad" (rojo, no solo "sin capacidad") significa
un error de configuración/autenticación — revisar el log de esa corrida
específica antes de asumir que es solo falta de capacidad.

## Notas de mantenimiento

- 2026-09-22: el workflow se apagó solo por la política de GitHub de
  desactivar cron jobs tras 60 días sin actividad en el repo (las corridas
  programadas no cuentan como "actividad" para ese contador, solo lo hacen
  los pushes). Reactivado manualmente. Si vuelve a pasar, basta con
  `gh workflow enable retry.yml` o un commit cualquiera a este repo.
