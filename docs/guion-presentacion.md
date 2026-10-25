# Guion de la presentación EP4 (5 a 10 minutos)

Hilo conductor: **una petición de un vecino recorre el sistema de punta a punta.** Primero se identifica (IDaaS y OAuth 2.0 / OIDC), después entra por la puerta única (API Manager) con un JWT que se valida, llega al backend desplegado en la nube y, por último, los efectos secundarios (correo, ticket de cuadrilla, certificado) viajan por RabbitMQ sin bloquear al usuario.

Dirección del frontend: `https://barriodigitalcloud.duckdns.org`
API Gateway: `https://lkbz5es695.execute-api.us-east-1.amazonaws.com`

## 1. Orden y tiempos

| # | Tiempo | Qué se muestra | Punto de la pauta que cubre |
|---|--------|----------------|-----------------------------|
| 0 | 0:00–0:40 | Diagrama de arquitectura (README de infra): JWT → API Gateway → BFF → microservicios, y RabbitMQ al costado. | Introducción |
| 1 | 0:40–1:30 | Portal de Azure (Entra ID): tenant, usuarios `admin.test`, `operador.test`, `cliente.test`, roles de la app. | 4. Tenant y usuarios en el IDaaS |
| 2 | 1:30–2:20 | Abrir el frontend, iniciar sesión con Microsoft y mostrar el token decodificado (claims `roles`, `aud`, `iss`, `exp`). | 5. Login OAuth 2.0 / OpenID Connect |
| 3 | 2:20–3:10 | Consola de AWS → API Gateway: API `BarrioDigital-API`, ruta `/{proxy+}`, integración con el BFF, stage. | 1. Instancia y configuración del API Manager |
| 4 | 3:10–3:50 | Frontend → Solicitudes con DevTools (pestaña Network): las llamadas van a `execute-api…amazonaws.com` con `Authorization: Bearer`. | 2. Frontend consumiendo por el API Manager |
| 5 | 3:50–4:50 | PowerShell: sin token → 401; token alterado → 401; token válido → 200. | 3. JWT inválido rechazado y válido aceptado |
| 6 | 4:50–5:30 | SSH a ec2-apps: `docker compose ps` (BFF, requests, catalog, notify, rabbitadmin, frontend) y candado HTTPS del frontend. | 6. Backend y frontend desplegados en la nube |
| 7 | 5:30–6:10 | SSH a ec2-mq: `docker compose ps` y `mq/compose.yml` (dos nodos + init que importa `definitions.json`). | 10. `docker compose` levantando RabbitMQ en la nube |
| 8 | 6:10–6:50 | Management UI → Overview: dos nodos en el clúster y política `ha-cmd`. | 7. Dos nodos RabbitMQ en clúster |
| 9 | 6:50–7:40 | Management UI → Exchanges: `cmd.direct` (direct), `cmd.topic` (topic) y sus bindings. | 8. Exchange direct y 9. Exchange topic |
| 10 | 7:40–9:20 | Queues: las tres colas con su DLQ. Crear una solicitud real (llega el correo / log de notify) y publicar un mensaje inválido para que termine en la DLQ. | 11. Tres colas con sus DLQ funcionando |
| 11 | 9:20–10:00 | Cierre y preguntas. | — |

Total estimado: 9 a 10 minutos. Si se acorta, se recortan los pasos 3 y 7 (basta nombrarlos).

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

## 3. Comandos listos

### Paso 5: JWT inválido y válido (PowerShell)

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

### Paso 6: despliegue (SSH en ec2-apps)

```bash
cd ~/barriodigital/barriodigital-infra/apps
docker compose ps
```

### Paso 7: RabbitMQ con Compose (SSH en ec2-mq)

```bash
cd ~/barriodigital/barriodigital-infra/mq
docker compose ps
docker exec rabbitmq1 rabbitmqctl cluster_status | head -20
```

### Paso 10: DLQ funcionando

**A) Flujo real.** En el frontend, crear una solicitud con correo. Luego:

```bash
docker compose logs --tail 30 ms-barriodigital-notify
```

Debe verse el consumo de `q.cmd.email` y el ACK. En la Management UI, `q.cmd.email` queda en 0 mensajes listos (se consumió y confirmó).

**B) Mensaje inválido que termina en la DLQ.** Management UI → Exchanges → `cmd.direct` → *Publish message*:

- Routing key: `crew.ticket`
- Properties: `content_type` = `application/json`
- Payload:

```json
{"type":"CREW_TICKET","eventId":"demo-dlq-001","timestamp":"2026-10-24T18:00:00Z","traceId":"demo","correlationId":"demo","payload":{"ticketId":"","category":"BACHE","location":"Plaza central","priority":"ALTA"}}
```

Resultado esperado: el listener rechaza el ticket porque no trae identificador (`IllegalArgumentException`, error no recuperable), hace **NACK sin reencolar** y RabbitMQ lo enruta a `q.cmd.crew.dlq` por el dead-letter exchange `cmd.dead.dlx`. En Queues, `q.cmd.crew.dlq` pasa de 0 a 1. Abrirla con *Get messages* (modo *Nack message requeue true*, para no destruirlo) y mostrar la cabecera `x-death`. Los logs de notify registran el motivo.

Para volver a dejarlo limpio: *Purge Messages* en `q.cmd.crew.dlq`.

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
El BFF, como *resource server* de Spring Security, descarga las claves públicas del tenant y verifica firma, emisor y expiración. Los roles (`Admin`, `Operador`, `Auditor`) vienen en el claim `roles` y controlan el acceso a cada ruta. Limitación honesta: aún no se valida la audiencia (`aud`) del token.

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
