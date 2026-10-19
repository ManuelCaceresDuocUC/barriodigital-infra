# RabbitMQ (ec2-mq): clúster de 2 nodos

Clúster `rabbit@rabbitmq1` + `rabbit@rabbitmq2` con la topología del caso (sección 8):
exchanges `cmd.direct`, `cmd.topic`, `cmd.dead.dlx`; colas `q.cmd.email|crew|certificate` y sus DLQ.

## Puesta en marcha

```
copy .env.example .env      # (Linux: cp)  y edita los valores
docker compose up -d
docker compose ps -a        # rabbitmq-init debe quedar en Exited (0)
```

Management UI: `http://<host>:15672` (nodo 1) y `:15673` (nodo 2), con el usuario del `.env`.

## Cómo está resuelto

| Tema | Decisión |
|---|---|
| Formación del clúster | `classic_config` con ambos nodos; reintenta 60 s, así que pueden arrancar a la vez y en cualquier orden. |
| Topología | `definitions.json`, importado por la API desde `rabbitmq-init` (cargarlo con `load_definitions` impide que RabbitMQ cree el vhost y el usuario de `.env`). |
| Alta disponibilidad | Política `ha-cmd`: *mirroring* clásico (`ha-mode: all`, sincronización automática, `ha-promote-on-shutdown: always`) sobre las colas `q.cmd.*`. |
| Persistencia | Volúmenes nombrados y `restart: unless-stopped`: sobrevive al apagado de 4 h del Learner Lab. |
| Credenciales | `.env` (no versionado). El usuario solo se crea con el volumen vacío. |

### ¿Por qué mirroring y no quorum queues?

Con 2 nodos, una quorum queue necesita a ambos para tener mayoría: si uno cae, la cola deja de estar
disponible, es decir, no hay tolerancia a fallos. El mirroring clásico sí sobrevive a la caída de un nodo.
Limitación: el mirroring está marcado como obsoleto desde RabbitMQ 3.9 y se elimina en la 4.0; con la imagen
`rabbitmq:3.13` fijada funciona sin problema. Una migración futura implicaría pasar a 3 nodos y quorum queues.

## Verificación

```
docker exec rabbitmq1 rabbitmqctl cluster_status
docker exec rabbitmq1 rabbitmqctl list_policies
docker exec rabbitmq1 rabbitmqctl list_queues name policy messages
```

### ¿Por qué `ha-promote-on-shutdown: always`?

Con el valor por defecto (`when-synced`), al detener el nodo maestro de forma ordenada (`docker stop`, o el apagado
del Learner Lab) el espejo **no** asumía el rol y las colas quedaban caídas hasta que volvía el maestro; solo se
promovía ante un fallo brusco. Con `always` el espejo asume también en una parada ordenada. Contrapartida: si el
espejo no estuviera sincronizado, podrían perderse mensajes; con `ha-sync-mode: automatic` esa ventana es mínima.

### Prueba de resiliencia

1. En la UI, *Exchanges → cmd.direct → Publish message* con routing key `email.send`.
2. Detener de forma ordenada el nodo que es maestro (`docker exec rabbitmq1 rabbitmqctl list_queues name pid` indica cuál es), por ejemplo `docker stop rabbitmq1`.
3. `docker exec rabbitmq2 rabbitmqctl list_queues name messages pid`: `q.cmd.email` conserva el mensaje y su `pid` ya es del otro nodo.
4. `docker start rabbitmq1` y `docker exec rabbitmq1 rabbitmqctl cluster_status`: ambos nodos en *Running Nodes*.
5. Repetir deteniendo el otro nodo (que ahora es el maestro).
6. Reinicio completo: `docker restart rabbitmq1 rabbitmq2`; el clúster debe recomponerse solo.
