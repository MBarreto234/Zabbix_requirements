# Zabbix Monitoreo (Docker)

Monitoreo de servicios de red (DNS, NTP, etc.) con Zabbix 7.0 en contenedores.

- `server/`: Zabbix Server + PostgreSQL + Frontend (PC central, Windows con Docker Desktop).
- `agent/`: Zabbix Agent 2 (se replica en cada VM).
- `services/`: un servicio por carpeta (`dns`, `ntp`, `syslog`, `tftp`); se levanta solo el que necesite cada VM.

## Requisitos

- Docker y Docker Compose v2.
- Red solo-anfitrión de VirtualBox (`192.168.56.0/24`): el servidor es `192.168.56.1`.
- Las VMs necesitan IP fija y salida a internet (adaptador NAT) para descargar imágenes.

## 1. Servidor (PC central)

```bash
cd server
cp .env.example .env     # en Windows: copy .env.example .env
# editar .env y cambiar POSTGRES_PASSWORD
docker compose up -d
```

Frontend: <http://localhost> (usuario `Admin`, contraseña `zabbix`; cambiarla al ingresar).
Permitir en el firewall de Windows el puerto `10051/tcp` desde `192.168.56.0/24`.

## 2. Agente (cada VM)

```bash
cd agent
cp .env.example .env
nano .env                # ZBX_HOSTNAME único y ZBX_SERVER_IP
docker compose up -d
docker ps
```

Luego, en el frontend: **Data collection → Hosts → Create host** con el mismo
nombre de `ZBX_HOSTNAME`, plantilla *Linux by Zabbix agent* (y *Docker by Zabbix agent 2*),
interfaz *Agent* con la IP de la VM y puerto `10050`.

Firewall de la VM:

```bash
sudo ufw allow from 192.168.56.1 to any port 10050 proto tcp
```

## 3. Servicios (cada VM levanta solo los suyos)

| Carpeta | Imagen | Puerto | VLAN |
|---|---|---|---|
| `services/dns` | `francopro/dns-vlan81:1.0` | 53 tcp/udp | 81 |
| `services/ntp` | `francopro/ntp-vlan82:1.0` | 123/udp | 82 |
| `services/syslog` | `francopro/syslog-vlan83:1.0` | 514/udp | 83 |
| `services/tftp` | `francopro/tftp-vlan84:1.0` | 69/udp | 84 |

```bash
cd services/dns
cp .env.example .env     # DNS: poner SERVER_IP con la IP de la VM
docker compose up -d
docker logs dns-server --tail 20
```

Los demás servicios no necesitan `.env`. Si un contenedor no arranca con `cap_drop: [ALL]`,
el log indica qué permiso falta: añadirlo en `cap_add`.
Las imágenes ya están fijadas por digest (`image: francopro/xxx@sha256:...`), así todas las VMs
corren exactamente la misma versión. Para actualizar una: `docker pull francopro/xxx:TAG`,
obtener el nuevo digest con `docker inspect --format '{{index .RepoDigests 0}}' francopro/xxx:TAG`
y reemplazarlo en el `docker-compose.yml` del servicio.

## Ítems de monitoreo recomendados

| Servicio | Clave (tipo Zabbix agent) |
|---|---|
| NTP | `net.udp.service[ntp,127.0.0.1]` |
| DNS | `net.dns[<IP_VM>,<dominio>,A,2,1]` y `net.tcp.port[<IP_VM>,53]` |
| Syslog | `proc.num[rsyslogd]` |
| TFTP | `proc.num[in.tftpd]` |

Trigger de ejemplo: `last(/<host>/net.udp.service[ntp,127.0.0.1])=0`.
Usar el *Host name* técnico del host en las expresiones, no el nombre visible.

## Apagado ordenado

1. VMs: `docker compose stop` en cada carpeta y `sudo shutdown -h now`.
2. Windows: `docker compose stop` en `server/` (**nunca** `down -v`: borra la base de datos).

## Seguridad

- `.env` está en `.gitignore`: no subir contraseñas ni tokens (Telegram, SMTP).
- Guardar los tokens en **macros secretas** de Zabbix, no en archivos del repositorio.
