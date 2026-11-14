# Kafka (ec2-kafka): 3 Zookeeper + 3 brokers + Kafka UI

Clúster de streaming de BarrioDigital. Complementa a RabbitMQ (`mq/`) sin reemplazarlo: RabbitMQ entrega
**comandos** (enviar un correo, crear un ticket) y Kafka transporta **hechos** (una solicitud cambió de estado)
que `report` y `audit` leen cada uno a su ritmo.

## Arquitectura

```
ec2-apps  (producers/consumers)                       ec2-kafka (t3.medium)
  requests ──publica──►  IP_PRIVADA:9092,9093,9094 ──►  kafka-1  kafka-2  kafka-3   (3 brokers)
  report   ◄──consume──                                    └────────┬────────┘
  audit    ◄──consume──                              zookeeper-1  zookeeper-2  zookeeper-3
  kafkaadmin ──administra tópicos y grupos──►                 kafka-ui :8088 (solo administrador)
```

| Pieza | Para qué sirve |
|---|---|
| 3 Zookeeper | Coordinan el clúster (elección del controlador, metadatos). Con 3 nodos hay mayoría aunque caiga 1. |
| 3 brokers | Guardan las particiones. Con réplicas en los 3 y `min.insync.replicas=2`, el clúster sigue publicando con 1 broker caído sin perder datos. |
| Kafka UI | Ver brokers, tópicos, particiones, mensajes, consumer groups, offsets y lag. |

Listeners: `INTERNAL` (`kafka-N:29092`) para brokers y Kafka UI dentro de Docker; `EXTERNAL`
(`IP_PRIVADA:9092|9093|9094`) para los microservicios de `ec2-apps`. Por eso los brokers anuncian la **IP privada**
de la EC2 (estable entre reinicios), no `localhost` ni la IP pública.

## Instancia ec2-kafka

- Amazon Linux 2023, VPC por defecto del Learner Lab, **t3.medium (4 GiB), 30 GB**, sin IP elástica.
- Si se queda sin memoria se puede subir a t3.large: detener la instancia, *Acciones > Configuración de la
  instancia > Cambiar el tipo de instancia*, iniciar. Los volúmenes se conservan.
- Security group (`ec2-kafka`):

| Puerto | Origen | Uso |
|---|---|---|
| 22 | tu IP | SSH |
| 9092, 9093, 9094 | security group de `ec2-apps` | Brokers (microservicios) |
| 8088 | tu IP | Kafka UI |

No se publican los puertos de Zookeeper fuera de la red Docker.

## Instalación (una sola vez)

```bash
sudo dnf install -y docker git
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/download/v2.29.7/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# Swap de 2 GiB: red de seguridad para los 7 contenedores JVM en 4 GiB
sudo dd if=/dev/zero of=/swapfile bs=128M count=16
sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Cierra y vuelve a abrir la sesión SSH para que el grupo `docker` surta efecto.

## Puesta en marcha

```bash
git clone https://github.com/ManuelCaceresDuocUC/barriodigital-infra.git
cd barriodigital-infra/kafka
cp .env.example .env
nano .env                     # KAFKA_HOST_IP = IP PRIVADA de esta EC2
chmod 600 .env
docker compose up -d
docker compose ps             # los 3 kafka deben pasar a (healthy) en ~1-2 minutos
```

Kafka UI: `http://<IP_PUBLICA>:8088` (la IP pública cambia al reiniciar el lab; la privada no).

## Verificación

```bash
# Los 3 brokers registrados en Zookeeper
docker exec kafka-1 kafka-broker-api-versions --bootstrap-server kafka-1:29092 | grep -c "id:"

# Prueba de un tópico con 3 particiones y 3 réplicas (los reales los crea kafkaadmin)
docker exec kafka-1 kafka-topics --bootstrap-server kafka-1:29092 --create --topic prueba \
  --partitions 3 --replication-factor 3
docker exec kafka-1 kafka-topics --bootstrap-server kafka-1:29092 --describe --topic prueba
docker exec kafka-1 kafka-topics --bootstrap-server kafka-1:29092 --delete --topic prueba
```

El `--describe` debe mostrar 3 particiones, cada una con `Replicas: x,y,z` e `Isr` con los 3 brokers.

### Resiliencia

```bash
docker stop kafka-2           # la publicación y el consumo siguen (ISR 2 de 3)
docker exec kafka-1 kafka-topics --bootstrap-server kafka-1:29092 --describe --topic prueba
docker start kafka-2          # se reincorpora y se vuelve a sincronizar
```

## Decisiones

| Tema | Decisión |
|---|---|
| Auto-creación de tópicos | Desactivada. Los tópicos se crean con `ms-barriodigital-kafkaadmin` (3 particiones, 3 réplicas, retención por tópico). |
| Persistencia | Volúmenes nombrados y `restart: unless-stopped`: sobrevive al apagado del Learner Lab. |
| Memoria | Heap limitado por JVM (Zookeeper 128 MB, brokers 320 MB, Kafka UI 256 MB) para caber en una t3.medium. |
| Seguridad | Sin TLS ni autenticación entre servicios: solo accesible desde el security group de `ec2-apps`. Aceptable para el alcance del curso; en producción se usarían SASL/TLS. |
| Versiones | Confluent Platform 7.6.0 (Kafka 3.6 con Zookeeper) y Kafka UI 0.7.2 fijadas. |

## Conexión desde los microservicios (ec2-apps)

```
SPRING_KAFKA_BOOTSTRAP_SERVERS=<IP_PRIVADA_EC2_KAFKA>:9092,<IP_PRIVADA_EC2_KAFKA>:9093,<IP_PRIVADA_EC2_KAFKA>:9094
```

Se define en `apps/.env` de `ec2-apps` cuando se despliegue el productor y los consumidores.
