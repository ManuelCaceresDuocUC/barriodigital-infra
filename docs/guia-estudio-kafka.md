# Guía de estudio: Kafka en BarrioDigital

Para la presentación individual (EP6). Cada sección termina con lo que hay que poder explicar en voz alta.

## 1. Idea central

Kafka es un **log de eventos distribuido y replicado**. Los productores agregan eventos al final de un tópico; los consumidores los leen a su ritmo y recuerdan hasta dónde leyeron (el *offset*). El evento **no se borra al leerlo**: se conserva según la retención configurada.

| | RabbitMQ (en el proyecto) | Kafka (en el proyecto) |
|---|---|---|
| Qué transporta | Comandos: «envía el correo» | Hechos: «la solicitud 22 pasó a RESUELTO» |
| Quién lo consume | Un solo consumidor por cola | Varios grupos independientes leen lo mismo |
| Tras leerlo | Se confirma (ACK) y desaparece | Sigue en el log; se puede releer |
| Orden | Por cola | Por partición |
| Uso aquí | `notify` (correo, ticket, certificado) | `report` (KPIs) y `audit` (trazabilidad) |

## 2. Vocabulario

- **Tópico**: categoría de eventos (`requests.events`, `audit.timeline`).
- **Partición**: subdivisión del tópico; el orden solo se garantiza dentro de una partición. Aquí, 3 por tópico.
- **Offset**: posición de un evento dentro de una partición.
- **Broker**: servidor Kafka. Aquí, 3.
- **Réplica / factor de replicación (RF)**: copias de cada partición en brokers distintos. Aquí RF = 3.
- **ISR (in-sync replicas)**: réplicas al día. Con `min.insync.replicas = 2` y `acks=all`, un evento confirmado está en al menos 2 brokers.
- **Líder**: la réplica que atiende lecturas y escrituras de una partición; las otras la copian.
- **Consumer group**: conjunto de consumidores que se reparten las particiones. Dos grupos distintos reciben **todos** los eventos cada uno.
- **Lag**: eventos publicados que el grupo aún no procesó (en Kafka UI, columna *Lag*; 0 = al día).
- **Zookeeper**: coordina el clúster (elección de líder, metadatos). Aquí, 3 nodos.

## 3. Arquitectura desplegada

```
requests ──(después del commit, asíncrono)──► Kafka (ec2-kafka)
                                               ├─ requests.events ──► report (KPIs)   ──► Oracle
                                               │                  └─► audit  (log)    ──► Oracle
                                               └─ audit.timeline  ──► audit
kafkaadmin ◄── BFF ◄── API Gateway ◄── frontend (administra tópicos y grupos)
```

- Clúster: 3 Zookeeper + 3 brokers (Confluent 7.6.0) + Kafka UI (:8088) en ec2-kafka.
- Tópicos: `requests.events` y `audit.timeline`, 3 particiones, RF 3, creados de forma idempotente por `kafkaadmin` al arrancar.
- Clave del mensaje: `requestId` → todos los eventos de una solicitud van a la misma partición, en orden.

## 4. Mensajes

`requests.events` (sin datos personales): `eventId`, `eventType` (`REQUEST_CREATED` / `REQUEST_STATUS_CHANGED`), `occurredAt`, `requestId`, `procedureType`, `previousStatus`, `status`.

`audit.timeline`: `eventId`, `occurredAt`, `requestId`, `action` (`STATUS_CHANGE`), `fromStatus`, `toStatus`, `reason`, `actorId`, `actorName`, `actorRole` y la cuadrilla (`crew`) cuando aplica.

## 5. Productor (`requests`)

- Se publica con `@TransactionalEventListener(AFTER_COMMIT)`: **solo si la transacción en Oracle se confirmó**. Si hay rollback, no sale ningún evento.
- Envío asíncrono con callback que registra tópico, partición y offset. `max.block.ms` ≤ 2 s para no colgar al usuario si Kafka no responde.
- Idempotente (`enable.idempotence=true`, `acks=all`): los reintentos del cliente no duplican eventos.
- Si Kafka falla, la solicitud se guarda igual y el error queda en el log. Mejora identificada: patrón outbox.

## 6. Consumidores

| | `report` | `audit` |
|---|---|---|
| Lee | `requests.events` | `requests.events` y `audit.timeline` |
| Grupo | `ms-report-consumer-group-v2` | grupo propio |
| Guarda | Métricas por tipo de trámite | Un registro por evento (con partición y offset) |
| Idempotencia | Tabla `PROCESSED_EVENTS` por `eventId` | Restricción única `UK_AUDIT_KAFKA_COORDS` (tópico + partición + offset) |
| Endpoints | `/api/report/kpis`, `/api/report/top-procedures` | `/api/audit/events` (Auditor y Admin) |

Errores: `DefaultErrorHandler` con 3 reintentos y 1 s de espera. Entrega *at-least-once* + idempotencia ≈ efecto único.

## 7. `kafkaadmin`

Administración vía API (a través del BFF, solo Admin): tópicos (crear, listar, cambiar configuración, aumentar particiones, eliminar), consumer groups (ver, reiniciar offsets solo si el grupo está vacío, eliminar) y estado del clúster. Los tópicos del sistema están protegidos contra borrado. Las particiones solo se pueden aumentar, nunca reducir.

## 8. Demostración sugerida (hilo)

1. Kafka UI → Brokers: 3 en línea. Topics: las particiones y réplicas de cada tópico.
2. Cambiar el estado de una solicitud en el portal.
3. Kafka UI → Topics → `requests.events` → Messages: el evento nuevo (clave = `requestId`).
4. Logs: `requests` («Evento en Kafka»), `report` («Métricas actualizadas»), `audit` («auditado con ID»).
5. Portal: Reportería (KPIs suben) y Auditoría (evento con actor y motivo).
6. Kafka UI → Consumers: lag 0 en ambos grupos.
7. Opcional: `curl.exe` a `/api/kafka/topics` con token de Admin.

## 9. Preguntas de autoevaluación

1. ¿Por qué Kafka y no solo RabbitMQ? *(hechos vs comandos; varios lectores; releer)*
2. ¿Qué garantiza `acks=all` con `min.insync.replicas=2`?
3. ¿Qué pasa si cae un broker? *(el líder se reelige entre las réplicas sincronizadas; RF 3 tolera 1 caída sin perder escrituras confirmadas)*
4. ¿Qué pasa si caen dos? *(con min ISR 2, las escrituras con `acks=all` fallan; las lecturas pueden seguir)*
5. ¿Por qué la clave es `requestId`? *(orden por solicitud)*
6. ¿Cómo evitan duplicados `report` y `audit`? ¿Por qué hacen falta si el productor es idempotente? *(el consumidor puede reprocesar tras un fallo antes de confirmar el offset)*
7. ¿Qué es un consumer group y por qué `report` y `audit` ven los mismos eventos?
8. ¿Qué significa lag y cómo se ve?
9. ¿Qué pasa con la solicitud si Kafka está caído? ¿Qué se pierde? *(se guarda igual; se pierde el evento; outbox)*
10. ¿Por qué se publica después del commit y no dentro de la transacción? *(no emitir hechos que luego se revierten; Kafka y Oracle no comparten transacción)*
11. ¿Por qué el evento no lleva datos personales?
12. ¿Cuántos consumidores útiles puede tener un grupo con 3 particiones? *(3; el cuarto queda ocioso)*
