# ec2-mq: instancia de RabbitMQ

- Instancia EC2 separada de `ec2-apps`, Amazon Linux 2023, VPC por defecto del Learner Lab.
- IP privada: `172.31.16.143` (estable entre reinicios). La IP pública cambia al reiniciar.
- Security group `ec2-mq-rabbitmq`:
  - 22 (SSH): acceso administrativo
  - 5672 (AMQP): origen = security group de `ec2-apps`
  - 15672 (Management UI): origen = IP del administrador
- Instalación: `sudo dnf install -y docker git`, `systemctl enable --now docker`,
  plugin Docker Compose v2 en `/usr/local/lib/docker/cli-plugins`.
- Despliegue: `cd mq && cp .env.example .env` (con clave y cookie propias, `chmod 600`)
  y `docker compose up -d`.

## Verificación realizada
- Clúster de 2 nodos (`rabbit@rabbitmq1`, `rabbit@rabbitmq2`) en Running Nodes.
- Política `ha-cmd` aplicada a las 6 colas `q.cmd.*`.
- `ec2-apps` alcanza `172.31.16.143:5672`.
- Resiliencia: con el master detenido, el otro nodo asumió las colas y conservó el mensaje.
  Al volver el nodo caído, se reincorporó al clúster sin pérdida de datos.