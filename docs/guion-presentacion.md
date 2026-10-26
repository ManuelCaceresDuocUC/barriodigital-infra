# Guion de la presentación EP4 (5 a 10 minutos)

Hilo conductor: **una petición de un vecino recorre el sistema de punta a punta.** Primero se identifica (IDaaS y OAuth 2.0 / OIDC), después entra por la puerta única (API Manager) con un JWT que se valida, llega al backend desplegado en la nube y, por último, los efectos secundarios (correo, ticket de cuadrilla, certificado) viajan por RabbitMQ sin bloquear al usuario.

Dirección del frontend: `https://barriodigitalcloud.duckdns.org`
API Gateway: `https://lkbz5es695.execute-api.us-east-1.amazonaws.com`

## 1. Orden y tiempos

La pauta de EP4 calcula el logro de la matriz sobre lo que se **demuestra en funcionamiento**, así que cada fila de [matriz-funcionalidades.md](matriz-funcionalidades.md) (M01 a M16) se muestra en algún paso. Varias salen de un mismo recorrido: crear una solicitud muestra login, roles, solicitudes, cupo y cola a la vez.

| # | Tiempo | Qué se muestra | Matriz | Punto de la pauta |
|---|--------|----------------|--------|-------------------|
| 0 | 0:00–0:30 | Diagrama de arquitectura (README de infra): JWT → API Gateway → BFF → microservicios, y RabbitMQ al costado. | — | Introducción |
| 1 | 0:30–1:20 | Portal de Azure (Entra ID): tenant, usuarios `admin.test`, `operador.test`, `cliente.test` y roles de la app. | M01 | Tenant y usuarios en el IDaaS |
| 2 | 1:20–2:00 | Frontend con la sesión cerrada: login con Microsoft y token decodificado (`iss`, `aud`, `roles`, `exp`). | M02 | Login OAuth 2.0 / OIDC |
| 3 | 2:00–3:10 | Consola de AWS → API Gateway: API, ruta `/{proxy+}`, integración con el BFF y el JWT Authorizer. Luego PowerShell: sin token → 401, token alterado → 401, token válido → 200. | M03 | Instancia y configuración del API Manager; JWT inválido/válido |
| 4 | 3:10–5:10 | **Recorrido integrado en el frontend.** Como `cliente.test`: crear una solicitud con correo propio y filtrarla (M06); ver que el menú solo ofrece lo permitido (M05); DevTools → Network con las llamadas a `execute-api…` y el Bearer (M04). Como `admin.test`: abrir el catálogo (M08), anotar el cupo del trámite, admitir la solicitud, cambiar su estado y probar una transición inválida (M07); ver que el cupo bajó (M09). | M04–M09 | Frontend consumiendo por el API Manager |
| 5 | 5:10–5:40 | SSH a ec2-apps: `docker compose ps` (BFF, requests, catalog, notify, rabbitadmin, frontend) y el candado HTTPS del frontend. | M16 | Backend y frontend desplegados en la nube |
| 6 | 5:40–6:40 | SSH a ec2-mq: `docker compose ps` y `cluster_status`. Management UI: los dos nodos, los exchanges `cmd.direct` (direct), `cmd.topic` (topic) y `cmd.dead.dlx`, y las seis colas con la política `ha-cmd`. | M14 | Compose de RabbitMQ; dos nodos en clúster; exchanges direct y topic |
| 7 | 6:40–7:40 | `docker compose logs --tail 30 ms-barriodigital-notify`: el evento de la solicitud del paso 4 fue consumido y confirmado (ACK); el correo llegó a la bandeja. | M10, M11 | Colas funcionando |
| 8 | 7:40–8:40 | Publicar un mensaje inválido desde la Management UI y mostrar 1 mensaje en `q.cmd.crew.dlq` con su cabecera `x-death`; mostrar el registro en los logs de notify. | M13 | Tres colas con sus DLQ funcionando |
| 9 | 8:40–9:30 | PowerShell con token de Admin: crear y eliminar una cola de prueba por el microservicio administrador (a través del BFF). | M15 | Administrador de RabbitMQ |
| 10 | 9:30–9:50 | Cierre: matriz de funcionalidades (16 funcionalidades, 15 demostradas; el PDF del certificado está declarado como parcial). | — | Sistematización |

Total estimado: unos 9 minutos y 50 segundos. Está justo en el límite de 10 minutos: si hace falta ganar tiempo, acorta el paso 0 y el 5.

## 2. Lista de pre-chequeo (30 minutos antes)

**Infraestructura (Learner Lab)**
- [ ] Sesión del laboratorio iniciada y con tiempo suficiente (la sesión se corta sola).
- [ ] ec2-mq encendida **primero**, ec2-apps después. Contenedores arriba: `docker compose ps` en `~/barriodigital/barriodigital-infra/mq` y en `.../apps`.
- [ ] La IP pública de ec2-mq cambió al reiniciar: anotar la nueva y confirmar que el Security Group deja entrar **tu IP actual** al puerto 15672.
- [ ] ec2-apps conserva su Elastic IP; el dominio DuckDNS apunta a ella (`nslookup barriodigitalcloud.duckdns.org`).
- [ ] `docker compose logs --tail 20 ms-barriodigital-notify` sin errores y con los listeners iniciados.

**Sesiones abiertas (dejar las pestañas listas)**
- [ ] Consola de AWS: API Gateway (`BarrioDigital-API`) y EC2.
- [ ] Portal de Azure: Entra ID → Users y App registrations.
- [ ] Frontend: `https://barriodigitalcloud.duckdns.org` con sesión **cerrada** para mostrar el login en vivo.
- [ ] Management UI: `http://<IP-PÚBLICA-EC2-MQ>:15672`, ya autenticada.
- [ ] Dos terminales: PowerShell (pruebas con `curl.exe`) y SSH a ec2-apps.
- [ ] Un visor de tokens (por ejemplo `https://jwt.ms`) abierto en otra pestaña.

**Datos y tokens**
- [ ] Un token válido de `admin.test` guardado en `$TOKEN` (dura aproximadamente una hora: obtenerlo **justo antes** de empezar).
- [ ] Un token alterado preparado en `$BAD`.
- [ ] Colas limpias: en Queues, **Purge** de `q.cmd.crew.dlq`, `q.cmd.email.dlq` y `q.cmd.certificate.dlq`, para que el contador parta de 0.
- [ ] Un trámite del catálogo con cupo disponible y un correo propio para el solicitante.
- [ ] Probar el recorrido completo una vez y cerrar sesión de nuevo.
- [ ] Confirmar en la consola de API Gateway que el JWT Authorizer está asignado a la ruta `/{proxy+}` (la pauta exige mostrar que el API Manager valida el JWT).
- [ ] Tener a mano un trámite con cupo y anotar su valor; tener probados los dos comandos del paso 9.

## 3. Comandos listos

### Paso 3: JWT inválido y válido (PowerShell)

```powershell
$API = 'https://lkbz5es695.execute-api.us-east-1.amazonaws.com'

# Cómo sacar el token: en el frontend, F12 → Network → una llamada a /api/... →
# Request Headers → Authorization → copiar lo que va después de "Bearer ".
$TOKEN = '<pegar el token aquí>'
$BAD   = $TOKEN.Substring(0, $TOKEN.Length - 4) + 'abcd'   # firma alterada

# 1) Sin token -> 401
curl.exe -s -o NUL -w "sin token:      %{http_code}`n" "$API/api/catalog/procedures"

# 2) Token alterado -> 401
curl.exe -s -o NUL -w "token alterado: %{http_code}`n" -H "Authorization: Bearer $BAD" "$API/api/catalog/procedures"

# 3) Token válido -> 200
curl.exe -s -o NUL -w "token valido:   %{http_code}`n" -H "Authorization: Bearer $TOKEN" "$API/api/catalog/procedures"
```

Qué decir: la petición llega al BFF a través de API Gateway; el BFF actúa como *resource server* y valida firma, emisor y expiración del JWT contra las claves públicas del tenant. Un token manipulado falla la verificación de firma.

### Paso 5: despliegue (SSH en ec2-apps)

```bash
cd ~/barriodigital/barriodigital-infra/apps
docker compose ps
```

### Paso 6: RabbitMQ con Compose (SSH en ec2-mq)

```bash
cd ~/barriodigital/barriodigital-infra/mq
docker compose ps
docker exec rabbitmq1 rabbitmqctl cluster_status | head -20
```

### Pasos 7 y 8: consumo y DLQ

**A) Flujo real (paso 7).** Con la solicitud creada en el paso 4:

```bash
docker compose logs --tail 30 ms-barriodigital-notify
```

Debe verse el consumo de `q.cmd.email` y el ACK. En la Management UI, `q.cmd.email` queda en 0 mensajes listos (se consumió y confirmó).

**B) Mensaje inválido que termina en la DLQ (paso 8).** Management UI → Exchanges → `cmd.direct` → *Publish message*:

- Routing key: `crew.ticket`
- Properties: `content_type` = `application/json`
- Payload:

```json
{"type":"CREW_TICKET","eventId":"demo-dlq-001","timestamp":"2026-10-24T18:00:00Z","traceId":"demo","correlationId":"demo","payload":{"ticketId":"","category":"BACHE","location":"Plaza central","priority":"ALTA"}}
```

Resultado esperado: el listener rechaza el ticket porque no trae identificador (`IllegalArgumentException`, error no recuperable), hace **NACK sin reencolar** y RabbitMQ lo enruta a `q.cmd.crew.dlq` por el dead-letter exchange `cmd.dead.dlx`. En Queues, `q.cmd.crew.dlq` pasa de 0 a 1. Abrirla con *Get messages* (modo *Nack message requeue true*, para no destruirlo) y mostrar la cabecera `x-death`. Los logs de notify registran el motivo.

Para volver a dejarlo limpio: *Purge Messages* en `q.cmd.crew.dlq`.

### Paso 9: administrador de RabbitMQ (PowerShell)

Crear, consultar y eliminar una cola de prueba con el token de `admin.test`, a través del BFF. El cuerpo va en un archivo porque PowerShell 5 altera las comillas dobles al pasárselas a `curl.exe`.

```powershell
'{"name":"q.demo.eval"}' | Set-Content -Encoding ascii body.json

# Crear -> 201
curl.exe -i -X POST "$API/api/rabbit/queues" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "@body.json"

# Consultar -> 200 (también aparece en la Management UI)
curl.exe -s "$API/api/rabbit/queues/q.demo.eval" -H "Authorization: Bearer $TOKEN"

# Eliminar -> 204
curl.exe -i -X DELETE "$API/api/rabbit/queues/q.demo.eval" -H "Authorization: Bearer $TOKEN"
```

Qué decir: solo el rol Admin puede usarlo (con otro rol responde 403) y las colas `q.cmd.*` y los exchanges `cmd.*` están protegidos (responden 409); por eso la demo usa una cola propia. Más ejemplos en `docs/rabbitadmin.http` del repositorio `ms-barriodigital-rabbitadmin`.

## 4. Preguntas probables del docente

**¿Por qué colas y no llamadas síncronas entre servicios?**
Porque el correo, el ticket de cuadrilla y el certificado son efectos secundarios que no deben retrasar ni hacer fallar la respuesta al vecino. Con una cola, `requests` responde apenas guarda la solicitud y publica el evento; `notify` lo procesa a su ritmo. Se desacoplan los servicios (requests no conoce a notify), se absorben picos y se pueden reintentar los fallos sin que el usuario espere. Con una llamada HTTP directa, si notify estuviera caído o lento, la creación de la solicitud fallaría o se demoraría.

**¿Qué pasa si notify cae?**
Los mensajes **no se pierden**: quedan en las colas `q.cmd.*`, que son durables y están replicadas en los dos nodos (política `ha-cmd`). Cuando notify vuelve, consume lo acumulado. Mientras tanto, `requests` sigue creando solicitudes con normalidad. Si es RabbitMQ el que está inaccesible al momento de publicar, hoy el fallo solo se registra en el log; la mejora identificada es el **patrón outbox** (guardar el evento en Oracle dentro de la misma transacción y publicarlo después), que queda documentada como trabajo futuro.

**¿Qué hace una DLQ?**
Es la cola donde van a parar los mensajes que no se pudieron procesar: los rechazados con NACK sin reencolar o los que agotaron los reintentos (3 intentos con espera de 1 s y 2 s). Evita que un mensaje defectuoso se reintente en bucle y bloquee la cola principal, y deja el mensaje guardado, con su cabecera `x-death`, para analizarlo o reprocesarlo. En el proyecto hay una DLQ por dominio: email, crew y certificate.

**¿Qué pasa si llega el mismo mensaje dos veces?**
El listener lleva un registro de `eventId` ya procesados y confirma con ACK sin repetir el efecto (idempotencia). Limitación honesta: el registro está en memoria y se pierde al reiniciar notify.

**¿Para qué sirve el exchange topic si hoy nadie publica en él?**
`cmd.direct` enruta por clave exacta y es el que usa el sistema hoy. `cmd.topic` queda declarado, con bindings `email.*`, `crew.#` y `certificate.*`, para enrutar por patrones cuando haya consumidores que necesiten, por ejemplo, todos los eventos de cuadrillas.

**¿Cómo se valida el JWT y quién lo valida?**
Se valida en dos capas. Primero, el API Gateway con su JWT Authorizer rechaza lo que no trae un token con firma, emisor, audiencia y expiración válidos. Segundo, el BFF y los microservicios, como *resource server* de Spring Security, vuelven a verificar firma, emisor y expiración y autorizan según el claim `roles` (`Admin`, `Operador`, `Auditor`). Limitación honesta: los servicios internos no comprueban la audiencia (`aud`); esa comprobación solo la hace el API Gateway.

**¿Por qué dos instancias EC2?**
Separar RabbitMQ de las aplicaciones permite reiniciar o escalar cada parte por separado, y mantener la IP privada del broker estable. El clúster de dos nodos con colas replicadas permite que el servicio de mensajería siga disponible si un nodo cae.

## 5. Si algo falla en vivo

| Síntoma | Qué hacer |
|---------|-----------|
| El token dio 401 con el válido | Expiró (dura ~1 h): copiar uno nuevo desde DevTools. |
| La Management UI no abre | La IP pública de ec2-mq cambió o tu IP no está en el Security Group (puerto 15672). |
| El frontend no carga | `docker compose ps` en ec2-apps; revisar que `frontend-barriodigital` esté `Up`. |
| El login de Microsoft da error de redirect | Falta la URI en Azure (SPA): `https://barriodigitalcloud.duckdns.org/dashboard`. |
| No llega el correo | Mostrar igualmente el consumo y el ACK en los logs de notify; el envío real depende de `MAIL_ENABLED` y de las credenciales SMTP. |
| Las pantallas de Auditoría o Reportes fallan | Dependen de Kafka (pendiente): no abrirlas en la demo. |
