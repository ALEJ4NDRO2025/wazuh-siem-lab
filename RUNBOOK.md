# Runbook: montar el laboratorio Wazuh desde cero

Tiempo estimado: 60 a 90 minutos. Cambia `<IP_PUBLICA_SERVIDOR>`, `<IP_PRIVADA_SERVIDOR>` e `<IP_PUBLICA_ENDPOINT>` por las tuyas.

## 1. Red en la nube

1. Crear una VCN con una subred pública.
2. En la Security List (o Security Group), permitir entrada TCP solo desde el origen necesario:

| Puerto | Uso | Origen (Source CIDR) |
| --- | --- | --- |
| 22 | SSH | `0.0.0.0/0` (solo con llave) o tu IP `/32` |
| 443 | Dashboard de Wazuh | Tu IP pública `/32` (https://ifconfig.me) |
| 1514-1515 | Eventos y registro de agentes | La subred privada, por ejemplo `10.0.0.0/24` |

Crea primero las reglas nuevas y después borra las abiertas a `0.0.0.0/0`; así nunca te quedas sin acceso. Ojo con el botón Actions de la Security List: su opción Terminate borra la lista completa.

## 2. Servidor Wazuh

**VM:** Ubuntu 22.04, mínimo 2 CPU y 8 GB de RAM, IP pública, llave SSH pública cargada.

```bash
ssh -i ~/.ssh/<llave_privada> ubuntu@<IP_PUBLICA_SERVIDOR>

sudo apt update && sudo apt upgrade -y
sudo reboot
```

Abrir los puertos en el firewall interno de Ubuntu (en Oracle Cloud no basta con la Security List):

```bash
sudo iptables -I INPUT -p tcp -m multiport --dports 443,1514,1515 -j ACCEPT
sudo netfilter-persistent save
```

Instalar Wazuh todo en uno (manager, indexer y dashboard):

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

Al final imprime la contraseña de `admin`: guardarla en un gestor de contraseñas.

Verificar:

```bash
for s in wazuh-manager wazuh-indexer wazuh-dashboard filebeat; do echo "$s: $(systemctl is-active $s)"; done
```

Los cuatro deben decir `active`. El indexer puede tardar 1 a 2 minutos después de un reinicio.

Dashboard: `https://<IP_PUBLICA_SERVIDOR>` (certificado autofirmado), usuario `admin`.

Si hay que regenerar la contraseña de admin:

```bash
curl -so wazuh-passwords-tool.sh https://packages.wazuh.com/4.14/wazuh-passwords-tool.sh
sudo bash wazuh-passwords-tool.sh -u admin
sudo systemctl restart filebeat wazuh-dashboard
```

## 3. Endpoint con agente

**VM:** Ubuntu 22.04, 1 CPU y 2 GB, en la misma VCN y subred.

```bash
ssh -i ~/.ssh/<llave_privada> ubuntu@<IP_PUBLICA_ENDPOINT>

wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.10-1_amd64.deb
sudo WAZUH_MANAGER='<IP_PRIVADA_SERVIDOR>' WAZUH_AGENT_NAME='endpoint-ubuntu' dpkg -i ./wazuh-agent_4.14.10-1_amd64.deb

sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
sudo systemctl is-active wazuh-agent
```

En el **servidor**, confirmar que el agente está `Active`:

```bash
sudo /var/ossec/bin/agent_control -l
```

## 4. Pruebas

### Fuerza bruta SSH (desde otro equipo, solo contra tu propio servidor)

```powershell
1..10 | ForEach-Object { ssh -o BatchMode=yes -o StrictHostKeyChecking=no usuariofalso@<IP_PUBLICA_SERVIDOR> exit }
```

Dashboard → Threat Hunting → buscar `rule.id:5710 or rule.id:5712`.

### Integridad de archivos (FIM)

```bash
mkdir ~/confidencial
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak
sudo nano /var/ossec/etc/ossec.conf
```

Justo debajo de `<syscheck>`:

```xml
<directories realtime="yes" report_changes="yes" check_all="yes">/home/ubuntu/confidencial</directories>
```

```bash
sudo systemctl restart wazuh-manager
sudo grep -i "confidencial" /var/ossec/logs/ossec.log | tail -3

echo "password_db=Secreto123" > ~/confidencial/config.txt
echo "password_db=Hackeado999" > ~/confidencial/config.txt
rm ~/confidencial/config.txt
```

Dashboard → Threat Hunting → buscar `rule.id:554 or rule.id:550 or rule.id:553`. En la alerta 550, el campo `syscheck.diff` muestra el cambio.

### Vulnerabilidades

Esperar 10 a 15 minutos después de conectar el agente. Dashboard → Vulnerability Detection → buscar `agent.name:endpoint-ubuntu`.

Remediación en el endpoint:

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
sudo apt autoremove -y
```

Esperar otros 15 minutos y comparar. Verificar las críticas restantes en `https://ubuntu.com/security/<CVE>`.

## 5. Diagnóstico

| Síntoma | Qué revisar |
| --- | --- |
| Un servicio no arranca | `sudo systemctl status <servicio>` y `sudo tail -50 /var/ossec/logs/ossec.log` |
| Agente en `Pending` o `Disconnected` | Puertos 1514/1515 en la Security List y en `iptables`; `sudo tail /var/ossec/logs/ossec.log` en el endpoint |
| El dashboard no carga (timeout) | La regla del 443 solo permite una IP. Si te conectas desde otra red (casa, otra oficina), tu IP pública cambió: revísala en https://ifconfig.me y agrega o actualiza la regla en la Security List con `TU_IP/32` |
| Tablero de vulnerabilidades vacío | `curl -k -u admin "https://127.0.0.1:9200/_cat/indices/wazuh-states-vuln*?v"` y que haya un agente activo |
| Cambié `ossec.conf` y algo falló | Restaurar: `sudo cp /var/ossec/etc/ossec.conf.bak /var/ossec/etc/ossec.conf` y reiniciar el manager |

## 6. Al terminar

- Apagar las VMs (Stop, nunca Terminate si se quieren conservar).
- No ejecutar `do-release-upgrade`: un cambio de versión de Ubuntu rompe Wazuh.
