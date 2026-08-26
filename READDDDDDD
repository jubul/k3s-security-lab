# 📧 latampass-notify — Diseño y plan de la Lambda de notificación diaria

> Documento de trabajo (no va al repo). El README del repo `oca-int-app-eop-lambda-latampass-notify` se escribe después, cuando el código esté hecho.

## Qué es

Lambda en Python que corre todos los días a las **07:00 (America/Montevideo)** disparada por **EventBridge Scheduler**, consulta los registros del **día anterior** en `[latampass].[dbo].[NonAirAccrualPayLoad ]` (**Amazon RDS for SQL Server Standard Edition** — confirmado 2026-08-25 contra la consola RDS), agrupa los errores por `errorCode` y publica un resumen en un **tópico SNS** que lo distribuye por email a los suscriptores.

Reemplaza el envío vía APIGetter que estaba en el diagrama original — la decisión actual es **SNS**.

---

## 🏗️ Arquitectura

![Arquitectura](oca-int-app-eop-lambda-latampass-notify-develop/docs/architecture.drawio.png)

```
EventBridge Scheduler (07:00 MVD)
        │ 1. invoca
        ▼
AWS Lambda latampass-notify (Python, VPC privada)
        │ 2. credenciales ← Secrets Manager (rds_secret*)
        │ 3. SELECT día anterior + control idempotencia → RDS SQL Server
        │ 4. agrupa por errorCode y arma resumen
        ▼
SNS Topic (cifrado KMS) ──► suscriptores email (equipo LatamPass/Operaciones)

CloudWatch: logs, métricas y alarma de fallo diario
```

Diagrama editable: `oca-int-app-eop-lambda-latampass-notify-develop/docs/architecture.drawio` (el PNG tiene el XML embebido — se abre directo en draw.io).

---

## 📊 Lógica de negocio (de `SQLQueryAleNotify.sql`)

1. Calcular la ventana del día anterior en America/Montevideo: `@desde` = ayer 00:00, `@hasta` = hoy 00:00.
2. **¿Hubo actividad ayer?** Si `COUNT(*) = 0` en esa ventana → no se envía nada y termina.
3. Traer los registros con error y agrupar:

```sql
SELECT [ffn], [status], [errorCode], [errorMessage],
       [errorInstructions], [errorIssuedDateTime], [errorDetails]
FROM [latampass].[dbo].[NonAirAccrualPayLoad ]
WHERE receivedDateTime >= @desde AND receivedDateTime < @hasta
  AND errorCode IS NOT NULL
ORDER BY errorCode, ffn ASC;
```

Correcciones respecto al SQL original de Ale:
- `LIKE '%2026-07-15%'` sobre datetime → **rango sargable** con parámetros (más rápido y no depende del formato de conversión).
- `errorCode <> ''` sobre una columna `int` → **`errorCode IS NOT NULL`** (el original funcionaba de casualidad por conversión implícita).
- ⚠️ El nombre de la tabla tiene un **espacio al final** (`NonAirAccrualPayLoad `, así está en el DDL) — las queries deben respetarlo.

### Formato del correo

SNS entrega **texto plano** (no soporta HTML — si algún día se quiere HTML hay que migrar a SES). Ejemplo con los datos reales del CSV:

```
Asunto: [LatamPass] Resumen de errores de acreditación - 2026-07-15

Resumen del 2026-07-15
Registros procesados: 152 | Con error: 6

════════════════════════════════════════════
Error 111 - Invalid country of residence (1 caso)
Instrucciones: Invalid program. Please contact our Latam team...
  - FFN 38854287768  (21:52:58)

Error 125 - Member status is CLOSED. Miles accrual is not allowed. (2 casos)
Instrucciones: The accrual of purchase NOT done
  - FFN 598077766866 (21:53:15)
  - FFN 59859380281K (22:17:22)

Error 128 - Document Type Invalid (3 casos)
Instrucciones: Validate your sending data
  - FFN 598476010996 (22:16:40)
  - FFN 598526363143 (22:17:08)
  - FFN 598666982099 (22:17:40)
```

---

## 🔁 Idempotencia (punto clave del diagrama original)

EventBridge puede reintentar y puede haber ejecuciones superpuestas. Propuesta: **tabla de control en la misma base** (no requiere infra AWS nueva, la Lambda ya tiene conectividad):

```sql
CREATE TABLE [dbo].[NotifyControl](
    [summaryDate]  date         NOT NULL,  -- día resumido (PK)
    [status]       varchar(20)  NOT NULL,  -- CLAIMED | SENT | NO_DATA | NO_ERRORS | FAILED
    [snsMessageId] varchar(100) NULL,
    [updatedAt]    datetime2(7) NOT NULL,
    CONSTRAINT PK_NOTIFYCONTROL PRIMARY KEY ([summaryDate])
);
```

Flujo: `INSERT` atómico de la fecha (claim) antes de enviar — si la PK ya existe en `SENT`/`NO_DATA`/`NO_ERRORS`, sale sin hacer nada. Tras publicar, actualiza a `SENT` con el `MessageId`. Si falla, queda `FAILED` y un reintento la retoma; un `CLAIMED` de más de 30 minutos se considera de una ejecución muerta y también se retoma.

**Reproceso manual**: invocar la Lambda con `{"summary_date": "YYYY-MM-DD"}` re-procesa ese día (si quedó `FAILED`; si ya está `SENT` no re-envía — para forzar, borrar la fila de `NotifyControl`).

Alternativa si no quieren tocar la base: DynamoDB con conditional put (requiere infra) — la tabla SQL es lo más simple dado que "no tocamos infra".

Requiere pedirle al DBA la creación de la tabla (o permiso de DDL una única vez).

---

## 🐍 Implementación (app/, lo único que tocamos)

- **Driver DB**: `python-tds` (pytds) — puro Python, se empaqueta en el zip sin dependencias nativas (pymssql/pyodbc necesitan binarios). TLS habilitado. (RDS Data API no aplica: es solo para Aurora.)
- **Estructura del handler**:
  1. Resolver ventana de fechas (zoneinfo, `SUMMARY_TIMEZONE`)
  2. Leer secreto de Secrets Manager (`DB_SECRET_NAME`) — cachear entre invocaciones
  3. Claim de idempotencia en `NotifyControl`
  4. Count de actividad → si 0, marcar `NO_DATA` y salir
  5. Query de errores → agrupar por `(errorCode, errorMessage, errorInstructions)`
  6. Armar asunto + body de texto plano
  7. `sns.publish(TopicArn=SNS_TOPIC_ARN, ...)`
  8. Marcar `SENT`; ante excepción marcar `FAILED` y relanzar (para que dispare la alarma)
- **Logs**: estructurados, con `ffn` enmascarado (últimos 4). Nunca loguear credenciales ni el body completo.
- **requirements.txt**: `boto3`, `python-tds` (sacar `requests`, ya no se usa).
- **Config por env vars** (ya existen en `locals-config.tf`, se agregan las que faltan): `SQL_SOURCE`, `SQL_CATALOG`, `DB_SECRET_NAME`*, `SNS_TOPIC_ARN`*, `SUMMARY_TIMEZONE`* (* = pendientes en infra).
- Estándares del repo s3-main: misma estructura `app/src/lambda_function.py`, `env_vars.yaml`, CHANGELOG con Keep a Changelog, CI por GitHub Actions (`app-flow.yml`), zip a S3 + SSM.

---

## 🚧 Gaps de infraestructura detectados (NO los tocamos — para el equipo de infra)

1. **No existe el schedule de EventBridge** ni el permiso de invocación sobre la Lambda → cron `0 7 * * ? *` con timezone `America/Montevideo` (EventBridge Scheduler soporta timezone; una Rule clásica sería `cron(0 10 * * ? *)` UTC).
2. **No existe el tópico SNS** ni las suscripciones email, ni `sns:Publish` en el rol IAM.
3. **La Lambda no tiene Security Group** (`main.tf` pasa `subnet_ids` pero no `security_group_ids`, a diferencia de s3-main) → egress restringido a SQL Server :1433 + VPC endpoints. Además, el SG de la instancia RDS debe permitir ingress :1433 desde el SG de la Lambda.
4. **Bugs en `locals-config.tf`**: `SQL_PASS = var.sql_user` (bug real) y `SQL_CATALOG = var.sqs_catalog` (typo). Ideal: eliminar credenciales de env vars y usar solo Secrets Manager.
5. `prd.tfvars` tiene `vpc_id` y subnets **vacíos** y valores de prueba (`test_sql_source`, `test@test.com`).
6. Falta **alarma CloudWatch** sobre fallos de la función (con 1 ejecución/día, un fallo = no llegó el resumen) y opcionalmente un destination `on_failure`.
7. Runtime declarado `python3.14` en locals vs `3.11` en `env_vars.yaml` — alinear.

---

## 🏛️ Well-Architected (6 pilares)

| Pilar | Cómo se aplica |
|---|---|
| **Excelencia operativa** | IaC Terraform + módulos OCA; CI/CD GitHub Actions; logs estructurados con correlation id; CHANGELOG; runbook en el README del repo |
| **Seguridad** | Credenciales solo en Secrets Manager; IAM mínimo privilegio (`rds_secret*`, logs propios, `sns:Publish` a un solo tópico); subnets privadas; TLS a DB y SNS; KMS at-rest en tópico y logs |
| **Fiabilidad** | Idempotencia por tabla de control; reintentos con backoff en query/publish; timeout Lambda (300 s) >> timeout DB; alarma si falla la corrida diaria |
| **Eficiencia de rendimiento** | Serverless; query sargable por rango; 512 MB de sobra para cientos de filas |
| **Optimización de costos** | 1 invocación/día ≈ costo ~cero; logs 14 días; sin NAT si hay VPC endpoints |
| **Sostenibilidad** | Ejecución bajo demanda 1 vez/día, sin recursos ociosos, runtime liviano |

## 🛡️ PCI DSS

- **Alcance**: el flujo **no maneja datos de tarjeta** (PAN/track/CVV). El `ffn` es número de socio de fidelidad; igual se trata como dato personal.
- **Minimización**: el mail lleva solo lo operativo (ffn, error, instrucciones); `errorDetails` no se incluye por defecto; en logs el `ffn` va enmascarado.
- **Cifrado (req. 3/4)**: TLS en tránsito (DB y SNS), KMS at-rest (tópico, logs).
- **Mínimo privilegio (req. 7)** y **credenciales (req. 8)**: sin secretos hardcodeados ni en tfvars; rotación en Secrets Manager.
- **Auditoría (req. 10)**: CloudWatch Logs + `NotifyControl` dejan traza de cada envío (fecha, estado, MessageId).
- **Red**: sin ingress, egress restringido, sin exposición pública.

---

## ✅ Plan de trabajo

1. ~~Diagrama de arquitectura draw.io~~ ✔ (`docs/architecture.drawio` + PNG)
2. ~~Escribir el código de la Lambda~~ ✔ (2026-08-25): `app/src/lambda_function.py` (handler y orquestación), `db.py` (pytds, queries, idempotencia), `summary.py` (agrupado + cuerpo del mail), `requirements.txt` (`boto3`, `python-tds`). Smoke test del armado del correo con el CSV real: OK.
3. Tests unitarios con datos del CSV real (`resultadosQuery1.csv`) como fixture
4. Pedir al DBA la tabla `NotifyControl`
5. Pasar al equipo de infra la lista de gaps (sección 🚧)
6. README del repo + CHANGELOG 1.0.0
7. Prueba end-to-end en dev (tópico SNS de prueba, suscripción a tu mail)

## ⏳ Pendientes (decisiones diferidas — 2026-08-25, el código usa estos defaults)

- ~~Motor de la base~~ ✔ Resuelto (2026-08-25): **RDS for SQL Server Standard Edition** (`aws-sqlsrv-standard-prd-patronimicos...rds.amazonaws.com`) — no es Aurora. Driver: `python-tds`.
- **Actividad con cero errores**: por ahora NO se manda mail (spec de Ale); queda registrado como `NO_ERRORS` en `NotifyControl`. Si se quiere mail "sin errores", agregar flag `SEND_OK_SUMMARY`.
- **Nombre definitivo del tópico SNS** y lista de suscriptores por ambiente → definir con infra.
- **`errorDetails`**: por ahora NO va en el mail (puede ser largo y sensible); la query directamente no lo trae.

---

# 🎓 Explicación del código, pieza por pieza (y el porqué de cada decisión)

## Por qué 3 módulos y no un solo archivo

```
app/src/
├── lambda_function.py   # QUÉ hace el proceso (orquestación, decisiones)
├── db.py                # CÓMO se habla con SQL Server (conexión, queries, idempotencia)
└── summary.py           # CÓMO se ve el mail (agrupado y formato, sin tocar red ni DB)
```

Es **separación de responsabilidades**: cada módulo tiene un motivo distinto para cambiar. Si mañana el mail pasa a HTML, tocás solo `summary.py`. Si migran la base, tocás solo `db.py`. El handler queda como un guion legible de arriba a abajo.

El beneficio inmediato es la **testeabilidad**: `summary.py` es una *función pura* — recibe datos, devuelve un string, no toca red ni base ni disco. Por eso pudimos probarlo hoy mismo con el CSV real sin tener acceso a la base ni a AWS. Regla práctica: cuanta más lógica puedas empujar hacia funciones puras, más barato es testear.

## `lambda_function.py` — el orquestador

### Inicialización a nivel de módulo (fuera del handler)

```python
sns = boto3.client("sns")
if not SNS_TOPIC_ARN:
    raise ValueError(...)
```

Lambda reutiliza el proceso entre invocaciones (**container reuse**): el código a nivel de módulo corre una sola vez en el *cold start*, y las invocaciones siguientes (*warm*) ya encuentran el cliente SNS creado. Crear clientes boto3 adentro del handler funcionaría, pero pagarías ese costo en cada invocación.

El `raise` si falta `SNS_TOPIC_ARN` es **fail fast**: si la config está rota, la Lambda muere al importar, con un error obvio, en vez de fallar a mitad del proceso con algo críptico. Es el mismo patrón que usa la Lambda de s3-main con `API_URL` — consistencia entre repos.

### La fecha con `zoneinfo`

```python
hoy = datetime.now(ZoneInfo("America/Montevideo")).date()
```

Lambda corre en UTC. Si hicieras `datetime.now().date()` a las 07:00 de Montevideo, en UTC ya son las 10:00 — todavía es el mismo día, pero un schedule a las 22:00 MVD (01:00 UTC del día siguiente) calcularía "ayer" mal. Fijar la zona de negocio explícitamente hace que el cálculo sea correcto sin importar a qué hora corra ni dónde. `zoneinfo` es stdlib desde Python 3.9 — no suma dependencias.

### El flujo como máquina de estados

El handler es una cadena de decisiones donde cada salida temprana deja rastro en `NotifyControl`:

```
reclamar fecha ──no──► "already-processed" (otro ya lo hizo/lo está haciendo)
      │sí
  ¿actividad? ──no──► NO_DATA   (no hubo movimientos → no hay nada que contar)
      │sí
  ¿errores?   ──no──► NO_ERRORS (hubo movimientos, todos OK → no molestamos)
      │sí
  publicar SNS ─────► SENT (+ MessageId)
      │excepción
      └────────────► FAILED + re-raise
```

Dos detalles importantes:

- **`try/finally` con `conn.close()`**: la conexión se cierra pase lo que pase. Las conexiones de base son un recurso del servidor; si las dejás colgadas, SQL Server las acumula hasta que expiran.
- **Marcar `FAILED` y RE-LANZAR la excepción** (`raise`): esto es deliberado y es una lección aprendida de la Lambda s3-main, que ante un error devolvía 200 igual y el fallo quedaba invisible. Acá, si algo explota, la Lambda **falla de verdad**: la métrica `Errors` de CloudWatch se incrementa, la alarma (cuando exista) se dispara, y el estado `FAILED` en la tabla permite que un reintento retome el trabajo. Regla: *nunca te tragues un error que alguien necesita ver*.

### Reproceso manual

```python
event.get("summary_date")  # {"summary_date": "2026-07-15"}
```

Costó 4 líneas y te salva un sábado: si el envío del martes falló, invocás la Lambda a mano (consola o CLI) con esa fecha y listo, sin tocar código ni esperar al schedule. Pensar en "¿cómo re-ejecuto esto cuando falle?" *antes* de que falle es parte de la excelencia operativa.

## `db.py` — el acceso a datos

### Queries parametrizadas (los `%s`)

```python
cur.execute("... WHERE receivedDateTime >= %s AND receivedDateTime < %s", (desde, hasta))
```

Los valores viajan **separados** del SQL y el driver se encarga de tiparlos. Tres razones:
1. **Inyección SQL**: acá el riesgo es bajo (las fechas las calculamos nosotros), pero es un hábito no negociable — el día que un valor venga del `event`, ya estás protegido.
2. **Tipos correctos**: el driver convierte `date`/`datetime` de Python al tipo SQL exacto, sin peleas de formato de fecha (¿`15/07` o `07/15`?).
3. **Plan cache**: SQL Server cachea el plan de ejecución de la query parametrizada y lo reutiliza cada día, en vez de recompilar por cada fecha distinta pegada como string.

### Rango de fechas en vez de `LIKE` (sargabilidad)

El SQL original de Ale hacía `receivedDateTime LIKE '%2026-07-15%'`. Eso obliga a SQL Server a **convertir cada fila a texto** para comparar → *table scan* completo, y de paso depende del formato regional. `>= @desde AND < @hasta` compara datetimes nativos y puede usar índices. A esto se le dice consulta **sargable** (*Search ARGument-able*). Nota el `< @hasta` en vez de `<= 23:59:59`: el intervalo semiabierto `[desde, hasta)` no deja huecos (un registro de las 23:59:59.9999999 caería afuera con `<=`).

Lo mismo con `errorCode <> ''`: la columna es `int`, comparar contra `''` fuerza una conversión implícita que *casualmente* funciona (`'' → 0`). `errorCode IS NOT NULL` dice lo que realmente queremos: "las filas que tienen error".

### La idempotencia en serio: el claim atómico

El problema real: EventBridge puede reintentar, alguien puede invocar a mano, dos ejecuciones pueden solaparse. ¿Cómo garantizás **un solo mail por día** sin un "coordinador" central? Usando la base como árbitro:

```python
try:
    cur.execute("INSERT INTO NotifyControl (summaryDate, 'CLAIMED', ...)")
    return True                      # gané la carrera: yo proceso este día
except pytds.IntegrityError:         # PK duplicada: otro llegó primero
    ...
```

La jugada es que el **PRIMARY KEY sobre `summaryDate`** convierte al INSERT en un *lock distribuido*: si dos Lambdas insertan la misma fecha al mismo tiempo, la base garantiza que **exactamente una** gana y la otra recibe violación de PK. No hay ventana de carrera, porque no hacemos "SELECT para ver si existe y después INSERT" (eso sí tendría una carrera entre el SELECT y el INSERT — el clásico *check-then-act*). La verificación y la acción son **una sola operación atómica**.

¿Y si la ganadora se murió a mitad de camino (timeout, OOM) y dejó el `CLAIMED` colgado? Para eso está el *takeover*:

```sql
UPDATE ... SET status='CLAIMED' WHERE summaryDate=%s
  AND (status='FAILED' OR (status='CLAIMED' AND updatedAt < hace 30 minutos))
```

Un `CLAIMED` de más de 30 minutos no puede ser una ejecución viva (el timeout de la Lambda es de 5), así que se considera huérfano y se retoma. El `UPDATE` también es atómico: `cur.rowcount == 1` te dice si fuiste vos quien lo retomó. `SENT`, `NO_DATA` y `NO_ERRORS` son estados terminales — nadie los retoma, por eso jamás sale un mail duplicado.

### Secrets Manager con cache y fallback

```python
_secret_cache = {}   # a nivel de módulo → sobrevive entre invocaciones warm
```

El secreto se pide **una vez por container**, no una vez por invocación: menos latencia y menos costo de API. El fallback a `SQL_USER`/`SQL_PASS` existe solo porque la infra actual todavía pasa credenciales por env vars — cuando infra defina `DB_SECRET_NAME`, el fallback queda muerto sin tocar código. Es una **migración sin fricción**: el código nuevo ya está listo para el mundo ideal pero funciona en el actual.

### ¿Por qué `python-tds` (pytds)?

Los drivers de SQL Server para Python son: `pyodbc` (necesita el ODBC driver de Microsoft instalado — en Lambda implica una layer custom), `pymssql` (extensión C compilada — el wheel tiene que coincidir con la plataforma de Lambda) y `python-tds` (**Python puro**: habla el protocolo TDS directamente, se mete en el zip con `pip install -t` y funciona en cualquier runtime). Con ~1 query al día, la performance del driver es irrelevante; la simplicidad del empaquetado gana.

### El espacio en el nombre de la tabla

`[NonAirAccrualPayLoad ]` — con espacio final, así está en el DDL de producción. Los corchetes de T-SQL existen justamente para nombres con caracteres raros. Está definido **una sola vez** como constante (`TABLA_PAYLOAD`) para que el día que alguien arregle el nombre en la base, se cambie en un solo lugar.

## `summary.py` — el formato del mail

### Restricciones de SNS que moldearon el diseño

1. **Solo texto plano**: SNS entrega a los suscriptores email el `Message` tal cual — sin HTML. Por eso el formato con separadores `====` (si se quiere HTML lindo, el camino es SES, otro servicio).
2. **Subject solo ASCII, máx. 100 chars**: por eso "acreditacion" sin tilde — con tilde, SNS rechaza el publish con `InvalidParameter`. Un acento te puede tirar el envío del día.
3. **Mensaje máx. 256 KB**: con ~6 errores/día no pasa nunca, pero un día catastrófico con miles de errores reventaría el publish y no saldría **ningún** mail. El truncado a 250 KB es un *seguro*: mejor un mail incompleto avisando que se truncó, que ningún mail el día que más lo necesitás.

### El agrupado

```python
clave = (errorCode, errorMessage, errorInstructions)
grupos.setdefault(clave, []).append(fila)
```

Una pasada por las filas, agrupando por la tupla completa (no solo el código — si un mismo código viniera con dos mensajes distintos, se verían separados). Como la query ya viene con `ORDER BY errorCode, ffn`, los grupos y sus FFN salen ordenados solos: la base ordena, Python agrupa, cada uno hace lo que mejor hace.

## Conceptos para llevarte (glosario)

| Concepto | En este proyecto |
|---|---|
| **Idempotencia** | Ejecutar N veces = ejecutar 1 vez. Lograda con claim atómico por PK en `NotifyControl`. |
| **Check-then-act (antipatrón)** | "SELECT para ver si existe, después INSERT" tiene una carrera. Se evita con INSERT directo + catch de PK duplicada. |
| **Sargable** | Query que puede usar índices. Rangos nativos sí; `LIKE` sobre datetime, no. |
| **Intervalo semiabierto `[desde, hasta)`** | `>= 00:00 de ayer AND < 00:00 de hoy` — sin huecos ni solapamientos entre días. |
| **Fail fast** | Config inválida → explotar al arrancar, no a mitad del proceso. |
| **Container reuse / warm start** | Lo caro (clientes boto3, secretos) se crea a nivel de módulo, una vez. |
| **Función pura** | `summary.py` no tiene efectos secundarios → se testea sin AWS ni base. |
| **Errores visibles** | Marcar `FAILED` y re-lanzar para que CloudWatch cuente el error — nunca devolver 200 ante un fallo. |
| **Estados terminales vs retomables** | `SENT`/`NO_DATA`/`NO_ERRORS` nadie los toca; `FAILED` y `CLAIMED` viejo se retoman. |
