# Guía de estudio del backend de BarrioDigital

Todo el código de los 8 microservicios tiene comentarios en español: cada clase empieza con **QUÉ ES / QUÉ HACE** y cada bloque importante lleva un `//` que explica la línea. El código no cambió (solo comentarios), y está en ramas aparte con PR sin mergear, para que **no se redespliegue nada en EC2**.

## 1. Cómo traer el código comentado

Desde PowerShell, en la carpeta donde tengas tus repos (sustituye `<repo>` y `<rama>` según la tabla):

```powershell
git clone https://github.com/ManuelCaceresDuocUC/<repo>.git   # solo si aún no lo tienes
cd <repo>
git fetch origin
git checkout <rama>          # rama con el código comentado
```

| Servicio | Repositorio | Rama | PR |
|---|---|---|---|
| bff | barriodigital-ms-barriodigital-bff | `20-docs-codigo-comentado` | #21 |
| requests | barriodigital-ms-barriodigital-requests | `39-docs-codigo-comentado` | #40 |
| catalog | barriodigital-ms-barriodigital-catalog | `20-docs-codigo-comentado` | #21 |
| notify | barriodigital-ms-barriodigital-notify | `27-docs-codigo-comentado` | #28 |
| report | barriodigital-ms-barriodigital-report | `11-docs-codigo-comentado` | #12 |
| audit | barriodigital-ms-barriodigital-audit | `13-docs-codigo-comentado` | #14 |
| kafkaadmin | barriodigital-ms-barriodigital-kafkaadmin | `8-docs-codigo-comentado` | #9 |
| rabbitadmin | barriodigital-ms-barriodigital-rabbitadmin | `22-docs-codigo-comentado` | #23 |

Para volver al código real de producción: `git checkout main`.

## 2. Mapa general (léelo antes de abrir código)

```
Angular ──► API Gateway ──► BFF (8080) ──► requests (8081) ──► catalog (8082)   (HTTP síncrono)
                              │                  │
                              │                  ├─► rabbitadmin (8086) ─► RabbitMQ ─► notify (8083)
                              │                  └─► Kafka ─► report (8084)  y  audit (8085)
                              ├─► rabbitadmin (8086)  administra RabbitMQ
                              └─► kafkaadmin  (8087)  administra Kafka
```

- **Autenticación**: Azure Entra ID entrega un JWT; cada servicio lo valida como *resource server* y lee el claim `roles`.
- **Autorización**: por rol (Admin, Operador, Cliente, Auditor). Ver `SecurityConfig` de cada servicio.
- **RabbitMQ** = comandos (enviar correo, ticket de tripulación, certificado). **Kafka** = eventos (métricas y auditoría).
- Puertos reales en producción: los del diagrama (docker-compose los fija; los `application.yaml` traen otros por defecto).

## 3. Orden recomendado de estudio

1. **bff** (el más simple: un proxy) → entiendes cómo entra una petición.
2. **catalog** → CRUD con JPA/Oracle y el cupo diario atómico.
3. **requests** → el corazón: reglas de negocio, máquina de estados, eventos.
4. **notify** → consumir de RabbitMQ (ACK, reintentos, idempotencia, DLQ).
5. **rabbitadmin** → administrar y publicar en RabbitMQ.
6. **report** y **audit** → consumir de Kafka.
7. **kafkaadmin** → administrar Kafka.

## 4. Qué archivo abrir en cada servicio

Rutas dentro de `src/main/java/...` salvo que se indique.

### bff
| Orden | Archivo | Qué aprendes |
|---|---|---|
| 1 | `controller/ApiProxyController.java` | Cómo reenvía cualquier `/api/**` al microservicio correcto, pasando el token |
| 2 | `config/ServicesProperties.java` | Tabla prefijo → URL de cada servicio, validada al arrancar |
| 3 | `config/SecurityConfig.java` | Exige token válido |
| 4 | `config/ProxyConfig.java` | Cliente HTTP y timeouts |
| 5 | `resources/application.yaml` | Rutas configuradas |

### catalog
| Orden | Archivo | Qué aprendes |
|---|---|---|
| 1 | `model/ProcedureType.java`, `model/QuotaUsage.java` | Entidades JPA (tablas Oracle) |
| 2 | `repository/*` | Consultas; **`QuotaUsageRepository`: UPDATE nativo atómico** del cupo |
| 3 | `service/QuotaService.java` | Consumir cupo sin condiciones de carrera |
| 4 | `service/ProcedureService.java` | CRUD de trámites |
| 5 | `controller/*` | Endpoints REST y roles |
| 6 | `exception/GlobalExceptionHandler.java` | Errores → HTTP |
| 7 | `config/SecurityConfig.java`, `RolesAuthoritiesConverter.java` | JWT y roles |

### requests
| Orden | Archivo | Qué aprendes |
|---|---|---|
| 1 | `model/RequestStatus.java` | **Máquina de estados** (qué cambios de estado son válidos) |
| 2 | `model/RequestEntity.java`, `RequestStatusHistoryEntity.java` | Tablas y relación con el historial |
| 3 | `service/RequestService.java` | **Archivo clave**: crear solicitud, cambiar estado, `SELECT … FOR UPDATE`, publicar eventos |
| 4 | `repository/RequestRepository.java` | Bloqueo pesimista |
| 5 | `controller/RequestController.java` | Endpoints y reglas por rol |
| 6 | `client/CatalogClient.java` | Llamada HTTP a catalog |
| 7 | `messaging/RequestNotificationPublisher.java` | `@TransactionalEventListener(AFTER_COMMIT)`: publica solo si la transacción se confirmó |
| 8 | `messaging/RabbitAdminMessageGateway.java`, `RequestMessages.java` | Cómo se arma el comando para RabbitMQ |
| 9 | `streaming/RequestStreamPublisher.java`, `KafkaEventStream.java`, `KafkaStreamConfig.java` | Publicar en Kafka (productor idempotente, `acks=all`) |
| 10 | `exception/GlobalExceptionHandler.java`, `security/Requester.java`, `config/*` | Errores y usuario autenticado |

### notify
| Orden | Archivo | Qué aprendes |
|---|---|---|
| 1 | `config/RabbitMQConfig.java` | **Topología**: exchanges, colas, DLQ, ACK manual |
| 2 | `listener/EmailListener.java` (y `CrewTicketListener`, `CertificateListener`) | Consumidor: valida, procesa, ACK/NACK |
| 3 | `listener/AckHandler.java` | Decisión de ACK, reintento con backoff o envío a DLQ |
| 4 | `listener/ProcessedEventStore.java` | Idempotencia (no procesar dos veces el mismo `eventId`) |
| 5 | `listener/DlqReporter.java` | Qué se registra cuando un mensaje muere |
| 6 | `service/*` | Lógica de cada comando |
| 7 | `dto/EventEnvelope.java` | Sobre común de eventos |
| 8 | `logging/SensitiveDataMasker.java` | Enmascarar correo y RUT en logs |
| 9 | `config/*Properties.java`, `resources/application.yaml` | Reintentos, idempotencia, nombres de colas |

### rabbitadmin
| Orden | Archivo | Qué aprendes |
|---|---|---|
| 1 | `controller/QueueController.java` (y Exchange/Binding) | API de administración |
| 2 | `service/RabbitAdminServiceImpl.java` | Lógica con `AmqpAdmin` + validaciones + recursos protegidos |
| 3 | `client/RabbitManagementClient.java` | Llamadas al Management API HTTP (listar, leer mensajes) |
| 4 | `service/MessagePublisherService.java`, `controller/MessagePublisherController.java` | Publicación para otros servicios (`/internal/messages`) |
| 5 | `service/MessageInspectionService.java`, `util/SensitiveDataMasker.java` | Leer DLQ sin consumir, con datos enmascarados |
| 6 | `config/SecurityConfig.java` | Todo exige rol Admin salvo health, Swagger e `/internal/messages` |
| 7 | `exception/GlobalExceptionHandler.java` | Mapa de códigos HTTP |

### report
| Orden | Archivo | Qué aprendes |
|---|---|---|
| 1 | `config/KafkaConsumerConfig.java` | Consumidor, `DefaultErrorHandler`, reintentos |
| 2 | `listener/KafkaReportEventListener.java` | Recibe evento de `requests.events` |
| 3 | `service/MetricsService.java` | Actualiza métricas; idempotencia por `eventId` |
| 4 | `entity/*`, `repository/*` | Tablas de métricas y eventos procesados |
| 5 | `controller/ReportController.java` | Endpoints de lectura (sin chequeo de rol en backend: lo exige el frontend + BFF autenticado) |

### audit
| Orden | Archivo | Qué aprendes |
|---|---|---|
| 1 | `config/KafkaConsumerConfig.java` | Consumidor Kafka |
| 2 | `listener/KafkaAuditEventListener.java` | Recibe `audit.timeline` |
| 3 | `service/AuditService.java` | Guarda la línea de tiempo; idempotencia por topic+partición+offset |
| 4 | `entity/AuditEvent.java`, `repository/*` | Tabla de auditoría |
| 5 | `controller/AuditController.java` | Consulta |

### kafkaadmin
| Orden | Archivo | Qué aprendes |
|---|---|---|
| 1 | `util/KafkaCalls.java` | Cómo se espera a Kafka y se traducen sus errores a HTTP |
| 2 | `service/TopicService.java` | Crear, configurar, ampliar y borrar tópicos |
| 3 | `service/ConsumerGroupService.java` | Grupos, **lag**, reseteo de offsets |
| 4 | `service/TopicProvisioner.java`, `ProtectedTopics.java` | Crea los tópicos del sistema al arrancar; protege los borrables |
| 5 | `controller/*` | Endpoints |
| 6 | `exception/GlobalExceptionHandler.java`, `util/TopicConfigRules.java` | Errores y configuraciones permitidas |
| 7 | `config/*`, `resources/application.yaml` | Conexión, seguridad, tópicos definidos |

## 5. Conceptos transversales (dónde verlos)

| Concepto | Dónde |
|---|---|
| JWT + roles de Azure | `config/SecurityConfig.java` y `RolesAuthoritiesConverter.java` (requests, catalog), `SecurityConfig` en cada servicio |
| Errores uniformes (`ApiError`) | `GlobalExceptionHandler` de cada servicio |
| Fail-fast de configuración | Records `@ConfigurationProperties` (`*Properties.java`) |
| Concurrencia: UPDATE atómico | `catalog` → `QuotaUsageRepository` |
| Concurrencia: bloqueo pesimista | `requests` → `RequestRepository` (`SELECT … FOR UPDATE`) |
| Evento solo tras commit | `requests` → `RequestNotificationPublisher`, `RequestStreamPublisher` |
| ACK manual / DLQ / reintentos | `notify` → `RabbitMQConfig`, `AckHandler` |
| Idempotencia | `notify` (memoria), `report` (eventId), `audit` (topic+partición+offset) |
| Enmascarar datos personales | `notify/logging` y `rabbitadmin/util` `SensitiveDataMasker` |

## 6. Flujo completo para seguir con el debugger o los logs

1. El cliente crea una solicitud: Angular → BFF `ApiProxyController` → `RequestController.create`.
2. `RequestService` valida, pide cupo a **catalog** (`CatalogClient` → `QuotaService`), guarda en Oracle.
3. Al confirmar la transacción se publican: comando a RabbitMQ (vía rabbitadmin) y evento a Kafka.
4. **notify** consume el comando, "envía" el correo y hace ACK; si falla, reintenta y luego va a la DLQ.
5. **report** suma métricas y **audit** guarda la línea de tiempo, ambos desde Kafka.
6. El administrador revisa colas en rabbitadmin y tópicos/lag en kafkaadmin.
