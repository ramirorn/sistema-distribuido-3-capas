# 🖧 Sistema Distribuido de Registro de Alumnos
### Stack 3 VMs · Ubuntu Server 24.04 · MariaDB · Node.js · Apache

---

<div align="center">

| | |
|---|---|
| **Autor** | Pereyra Roman, Ramiro Nicolás |
| **Asignatura** | Seminario de Actualización II |
| **Año** | 2026 |

</div>

---

## 📋 Descripción del Trabajo Práctico

El objetivo del trabajo fue armar un stack de **3 servidores independientes** sobre máquinas virtuales Linux Ubuntu Server 24.04 en VirtualBox, para gestionar un **registro de alumnos** con separación de responsabilidades entre capas:

- **VM1** → Base de datos (MariaDB)
- **VM2** → Backend (Node.js)
- **VM3** → Frontend (Apache)

---

## 🏗️ Arquitectura General

```
┌──────────────────────────────────────────────────────────┐
│                    PC HOST (físico)                      │
│           Navegador → http://localhost:8080/Sistema      │
└──────────────────────┬───────────────────────────────────┘
                       │ NAT + Port Forwarding (VirtualBox)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                  RED NAT INTERNA · 10.0.2.0/24              │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐   │
│  │  VM3 Frontend│──▶│  VM2 Backend │──▶│  VM1 Base datos│   │
│  │  10.0.2.17   │   │  10.0.2.16   │   │  10.0.2.15     │   │
│  │  Apache:8080 │   │  Node.js:3454│   │  MariaDB:3306  │   │
│  └──────────────┘   └──────────────┘   └────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Tabla de componentes

| VM  | Rol         | IP         | Puerto | Tecnología        |
|-----|-------------|------------|--------|-------------------|
| VM1 | Base datos  | 10.0.2.15  | 3306   | MariaDB           |
| VM2 | Backend     | 10.0.2.16  | 3454   | Node.js + Express |
| VM3 | Frontend    | 10.0.2.17  | 8080   | Apache + HTML/JS  |

### Acceso SSH desde el host

| Puerto host   | Destino       | VM  |
|---------------|---------------|-----|
| localhost:2221 | VM1:22       | BD  |
| localhost:2222 | VM2:22       | Backend |
| localhost:2223 | VM3:22       | Frontend |

---

## ⚙️ Preparación de las Máquinas Virtuales

Se crearon tres VMs en VirtualBox con las siguientes características:

- **SO:** Ubuntu Server 24.04.4 LTS
- **RAM:** 1024 MB por VM
- **Disco:** 11 GB por VM
- **Red:** Adaptador NAT con reenvío de puertos

Los usuarios de cada VM quedaron definidos como:

| VM  | Usuario    |
|-----|------------|
| VM1 | `bd`       |
| VM2 | `backend`  |
| VM3 | `frontend` |

---

## 🌐 Configuración de Red NAT e IPs Estáticas

### Reenvío de puertos en VirtualBox

Para cada VM se configuraron reglas de reenvío en:
**Configuración → Red → Adaptador 1 → Avanzado → Reenvío de puertos**

### IPs estáticas con Netplan

En cada VM se editó `/etc/netplan/00-installer-config.yaml` y se aplicó la configuración.

> ⚠️ **Problema encontrado:** Ubuntu 24.04 instala un archivo `/etc/netplan/50-cloud-init.yaml` que activa DHCP y asigna `10.0.2.15` a todas las VMs, causando conflicto de IPs. Las VMs clonadas compartían la misma IP (`10.0.2.15`) en todas las interfaces.

**Solución aplicada:**

```bash
# Eliminar el archivo de cloud-init que pisaba la configuración
sudo rm /etc/netplan/50-cloud-init.yaml

# Evitar que cloud-init regenere configuración de red
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network.cfg'

# Corregir permisos del archivo Netplan
sudo chmod 600 /etc/netplan/00-installer-config.yaml

sudo netplan apply
sudo reboot
```

**Configuración Netplan de VM1 (`10.0.2.15`):**

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.0.2.15/24
      routes:
        - to: default
          via: 10.0.2.2
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

> Idem para VM2 con `10.0.2.16` y VM3 con `10.0.2.17`.

---

## 🗄️ VM1 — Configuración de Base de Datos (MariaDB)

### Instalación

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y mariadb-server
sudo mysql_secure_installation
```

### Configuración para acceso remoto

Se editó el archivo de configuración de MariaDB:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Se cambió la línea:

```ini
# Antes:
bind-address = 127.0.0.1

# Después:
bind-address = 0.0.0.0
```

```bash
sudo systemctl restart mariadb
```

### Creación de base de datos, tabla y usuario

```sql
CREATE DATABASE registro_alumnos CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE registro_alumnos;

CREATE TABLE alumnos (
    id        INT AUTO_INCREMENT PRIMARY KEY,
    apellidos VARCHAR(100),
    nombres   VARCHAR(100),
    dni       VARCHAR(20) UNIQUE
);

CREATE USER 'usuario_consulta'@'%' IDENTIFIED BY 'ClaveSegura2024!';
GRANT SELECT, INSERT ON registro_alumnos.* TO 'usuario_consulta'@'%';
FLUSH PRIVILEGES;
```

### Firewall

```bash
sudo ufw allow from 10.0.2.16 to any port 3306
sudo ufw enable
```

---

## ⚙️ VM2 — Configuración del Backend (Node.js)

### Instalación de Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### Inicialización del proyecto

```bash
mkdir -p ~/backend-alumnos
cd ~/backend-alumnos
npm init -y
npm install express mysql2 cors
```

### Archivo `server.js`

```javascript
const express = require('express');
const mysql   = require('mysql2');
const cors    = require('cors');

const app  = express();
const PORT = 3454;

app.use(express.json());
app.use(cors());

const db = mysql.createConnection({
  host:     '10.0.2.15',
  user:     'usuario_consulta',
  password: 'ClaveSegura2024!',
  database: 'registro_alumnos'
});

db.connect((err) => {
  if (err) { console.error('[ERROR]', err.message); process.exit(1); }
  console.log('[OK] Conectado a MariaDB');
});

app.post('/grabaAlumnos', (req, res) => {
  const { apellidos, nombres, dni } = req.body;
  if (!apellidos || !nombres || !dni)
    return res.json({ resultado: 0, mensaje: 'Campos incompletos' });

  db.query(
    'INSERT INTO alumnos (apellidos, nombres, dni) VALUES (?, ?, ?)',
    [apellidos, nombres, dni],
    (err, result) => {
      if (err) {
        if (err.code === 'ER_DUP_ENTRY')
          return res.json({ resultado: 0, mensaje: 'DNI ya registrado' });
        return res.json({ resultado: 0, mensaje: 'Error interno' });
      }
      res.json({ resultado: 1, mensaje: 'Alumno registrado correctamente' });
    }
  );
});

app.get('/consultarAlumnos', (req, res) => {
  db.query(
    'SELECT id, apellidos, nombres, dni FROM alumnos ORDER BY apellidos, nombres',
    (err, rows) => {
      if (err) return res.status(500).json({ resultado: 0 });
      res.json({ resultado: 1, alumnos: rows });
    }
  );
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`[OK] Servidor escuchando en puerto ${PORT}`);
});
```

### Servicio systemd

Se creó `/etc/systemd/system/backend-alumnos.service`:

```ini
[Unit]
Description=Backend Registro Alumnos
After=network.target

[Service]
Type=simple
User=backend
WorkingDirectory=/home/backend/backend-alumnos
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable backend-alumnos
sudo systemctl start backend-alumnos
```

> ⚠️ **Problema encontrado:** El servicio fallaba con `status=217/USER` porque el archivo tenía `User=alumno` pero el usuario real de la VM era `backend`. Se corrigió la línea y también `WorkingDirectory` que apuntaba a `/home/alumno/` en vez de `/home/backend/`.

**Verificación:**

```bash
sudo systemctl status backend-alumnos
# Active: active (running)

curl http://localhost:3454/consultarAlumnos
# {"resultado":1,"alumnos":[]}
```

---

## 🌐 VM3 — Configuración del Frontend (Apache)

### Instalación de Apache

> ⚠️ **Problema encontrado:** La VM3 no tenía acceso a internet debido a que la red institucional bloquea el tráfico DNS desde VirtualBox NAT. Se resolvió transfiriendo los paquetes `.deb` desde el host Windows mediante SCP:

```powershell
# Desde PowerShell en Windows
scp -P 2223 apache2.deb frontend@localhost:/home/frontend/
```

```bash
# Instalación desde el .deb descargado manualmente
sudo dpkg -i *.deb
sudo apt install -f
```

### Configuración del puerto 8080

```bash
sudo nano /etc/apache2/ports.conf
```

```apache
Listen 8080
```

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

```apache
<VirtualHost *:8080>
    DocumentRoot /var/www/html
</VirtualHost>
```

```bash
sudo systemctl restart apache2
```

### Creación del sitio

```bash
sudo mkdir -p /var/www/html/Sistema
```

Se creó `/var/www/html/Sistema/index.html` con formulario HTML, JavaScript y Fetch API consumiendo los endpoints de VM2.

La URL de acceso desde el host:

```
http://localhost:8080/Sistema
```

---

## 🔧 Problemas Encontrados y Soluciones

### 1. SSH con `kex_exchange_identification: Connection reset`

**Causa:** Ubuntu 24.04 usa `ssh.socket` (socket activation) que en combinación con el NAT de VirtualBox causa resets en el handshake SSH desde Windows.

**Solución:**

```bash
sudo systemctl stop ssh.socket
sudo systemctl disable ssh.socket
sudo systemctl mask ssh.socket
sudo systemctl restart ssh
```

---

### 2. Conflicto de IPs — todas las VMs con `10.0.2.15`

**Causa:** El archivo `/etc/netplan/50-cloud-init.yaml` generado por cloud-init activa DHCP y VirtualBox asigna la misma IP a todas las VMs clonadas.

**Solución:**

```bash
sudo rm /etc/netplan/50-cloud-init.yaml
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network.cfg'
sudo netplan apply
sudo reboot
```

---

### 3. Servicio systemd con `status=217/USER` y `status=200/CHDIR`

**Causa:** El archivo `.service` tenía `User=alumno` y `WorkingDirectory=/home/alumno/` pero el usuario real de VM2 era `backend`.

**Solución:** Editar el archivo de servicio con los valores correctos:

```ini
User=backend
WorkingDirectory=/home/backend/backend-alumnos
```

---

### 4. Sin acceso a internet en VM3 para instalar Apache

**Causa:** La red institucional bloquea el tráfico DNS y ICMP desde VirtualBox NAT hacia el exterior.

**Solución:** Descarga de paquetes `.deb` en el host Windows y transferencia a la VM por SCP:

```powershell
scp -P 2223 apache2.deb frontend@localhost:/home/frontend/
```

---

### 5. `resolv.conf` sobreescrito automáticamente

**Causa:** `systemd-resolved` y cloud-init regeneran `/etc/resolv.conf` en cada arranque.

**Solución:** Bloquear el archivo con `chattr`:

```bash
sudo chattr -i /etc/resolv.conf
sudo bash -c 'echo "nameserver 10.254.196.1" > /etc/resolv.conf'
sudo chattr +i /etc/resolv.conf
```

---

## ✅ Verificación Final

### Backend responde correctamente

```bash
# Registrar alumno
curl -X POST http://localhost:3454/grabaAlumnos \
  -H "Content-Type: application/json" \
  -d '{"apellidos":"García","nombres":"Luis","dni":"30123456"}'
# → {"resultado":1,"mensaje":"Alumno registrado correctamente"}

# DNI duplicado
curl -X POST http://localhost:3454/grabaAlumnos \
  -H "Content-Type: application/json" \
  -d '{"apellidos":"Otro","nombres":"Test","dni":"30123456"}'
# → {"resultado":0,"mensaje":"DNI ya registrado"}

# Consultar alumnos
curl http://localhost:3454/consultarAlumnos
# → {"resultado":1,"alumnos":[...]}
```

### Checklist de componentes

| Componente | Estado |
|---|---|
| MariaDB escuchando en `0.0.0.0:3306` | ✅ |
| Usuario `usuario_consulta` con permisos correctos | ✅ |
| Backend Node.js corriendo en puerto 3454 | ✅ |
| Endpoint `POST /grabaAlumnos` funcional | ✅ |
| Endpoint `GET /consultarAlumnos` funcional | ✅ |
| Validación de DNI duplicado | ✅ |
| Apache sirviendo en puerto 8080 | ✅ |
| Interfaz web en `/Sistema` | ✅ |
| Servicio `backend-alumnos` con systemd | ✅ |

---

## 🚀 Desactivación de cloud-init

Para evitar arranques lentos en Ubuntu Server 24.04:

```bash
sudo touch /etc/cloud/cloud-init.disabled

sudo systemctl stop cloud-init
sudo systemctl disable cloud-init
sudo systemctl mask cloud-init

sudo systemctl disable cloud-config
sudo systemctl disable cloud-final
sudo systemctl disable cloud-init-local

sudo systemctl disable systemd-networkd-wait-online.service
sudo systemctl mask systemd-networkd-wait-online.service
```

**Verificación:**

```bash
dpkg -l | grep cloud-init
ls -l /etc/cloud/cloud-init.disabled
systemctl status cloud-init
systemctl status cloud-config
systemctl status cloud-final
systemctl status cloud-init-local
```

---

## 🎓 Conclusión

Se implementó exitosamente un sistema distribuido de tres capas sobre VirtualBox con Ubuntu Server 24.04. El proceso implicó resolver varios problemas reales de infraestructura que no están documentados en la mayoría de las guías:

- El comportamiento de `ssh.socket` en Ubuntu 24.04 con clientes Windows
- El conflicto de IPs causado por `cloud-init` en VMs clonadas
- Las restricciones de red institucional que impiden acceso a repositorios desde VMs NAT
- La gestión correcta de servicios systemd con usuarios y rutas específicas por VM

Cada problema fue diagnosticado, comprendido y resuelto de forma fundamentada, aplicando los principios de separación de responsabilidades, mínimo privilegio y reproducibilidad que caracterizan una arquitectura distribuida bien diseñada.

---

*Seminario de Actualización II · 2026 · Pereyra Roman, Ramiro Nicolás*
