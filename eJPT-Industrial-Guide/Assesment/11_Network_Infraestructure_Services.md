---
creado: 2026-02-16 18:11
tags:

  - tipo/apunte
  - ejpt
  - infraestructure
  - services
  - network
  - enum
  - recon
asignatura:
---

Markdown

````

---

## 1. Metasploit Port Scanning (Alternative to Nmap)
Si Nmap no está disponible o necesitas integrar los resultados directamente en la base de datos de Metasploit (`db_nmap`).

**Setup:**
```bash
service postgresql start
msfconsole
workspace -a infrastructure_enum
````

**TCP Scan Module:** Realiza un handshake completo (SYN-SYN/ACK-ACK). Más preciso pero más ruidoso que SYN Scan.

Bash

```
use auxiliary/scanner/portscan/tcp
set RHOSTS <TARGET_IP>
set PORTS 1-1000
set THREADS 10
set CONCURRENCY 10
run
```

---

## 2. SSH Enumeration (Port 22)

_El estándar de oro para la administración remota._

### Version Fingerprinting

Identificar la versión exacta (ej. OpenSSH 7.2p2) es vital para buscar exploits de corrupción de memoria o bypass de autenticación.

Bash

```
use auxiliary/scanner/ssh/ssh_version
set RHOSTS <TARGET_IP>
run
```

### SSH Brute Force (Credential Access)

> [!CAUTION] **Account Lockout** En entornos reales, esto bloqueará las cuentas después de 3-5 intentos. Usar solo si se confirma que no hay políticas de bloqueo o en fase final.

Bash

```
use auxiliary/scanner/ssh/ssh_login
set RHOSTS <TARGET_IP>
set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt
set PASS_FILE /usr/share/wordlists/rockyou.txt
set VERBOSE true
run
```

---

## 3. SMTP Enumeration (Port 25/465/587)

_Simple Mail Transfer Protocol. A menudo olvidado, pero una mina de oro para enumerar usuarios válidos del sistema._

### User Enumeration (VRFY / EXPN)

Abusa de los comandos `VRFY` (Verify) y `EXPN` (Expand) para confirmar si un usuario existe en el sistema sin autenticarse.

Bash

```
use auxiliary/scanner/smtp/smtp_enum
set RHOSTS <TARGET_IP>
set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt
run
```

### Version Scanning

Bash

```
use auxiliary/scanner/smtp/smtp_version
set RHOSTS <TARGET_IP>
run
```

---

## 🏭 OT/ICS Impact: The Industrial Context

- **SSH en OT:** Común en Gateways de Comunicaciones, RTUs modernas y servidores SCADA/Historian.
    
    - _Riesgo:_ Un ataque de fuerza bruta SSH consume mucha CPU (encriptación). En un PLC embebido o un Gateway serial-to-ethernet antiguo, esto **causará latencia** en el proceso industrial.
        
- **SMTP en OT:** Usado por PLCs y HMIs para enviar alarmas críticas por correo.
    
    - _Riesgo:_ Saturar el servicio SMTP puede impedir que el operador reciba una alerta de "Temperatura Crítica".
        

````

---

### Archivo 2: `04_Database_Hunting_MySQL.md`
*Fusión de toda tu enumeración de MySQL. Enfoque: Extracción de datos.*


---

## 1. Discovery & Fingerprinting
Antes de atacar, identifica la versión. MySQL < 5.x suele ser vulnerable a exploits de buffer overflow o UDF injection.

```bash
use auxiliary/scanner/mysql/mysql_version
set RHOSTS <TARGET_IP>
run
````

---

## 2. Access: Brute Force & Login

Validar credenciales por defecto o débiles.

Bash

```
use auxiliary/scanner/mysql/mysql_login
set RHOSTS <TARGET_IP>
set USERNAME root
# Usar diccionarios específicos de bases de datos si es posible
set PASS_FILE /usr/share/wordlists/metasploit/mysql_default_pass.txt
set VERBOSE false
run
```

_Si tienes éxito, Metasploit guardará la sesión credencial en `creds`._

---

## 3. Post-Exploitation: Enumeration

Una vez autenticado (credenciales válidas requeridas).

### System Recon & Hash Dumping

Extrae hashes de contraseñas de otros usuarios de la base de datos (tabla `mysql.user`).

Bash

```
use auxiliary/scanner/mysql/mysql_enum
set RHOSTS <TARGET_IP>
set USERNAME <USER>
set PASSWORD <PASS>
run
```

### Schema Dumping (Data Map)

Descarga la estructura de todas las tablas y columnas. Vital para entender dónde está la información crítica (ej. tablas `credit_cards`, `admin_logins`).

Bash

```
use auxiliary/scanner/mysql/mysql_schemadump
set RHOSTS <TARGET_IP>
set USERNAME <USER>
set PASSWORD <PASS>
run
```

### SQL Query Execution (Ad-Hoc)

Ejecutar consultas arbitrarias para leer datos o escribir archivos.

Bash

```
use auxiliary/admin/mysql/mysql_sql
set RHOSTS <TARGET_IP>
set USERNAME <USER>
set PASSWORD <PASS>
set SQL "SHOW DATABASES;"
run
```

---

## 🏭 OT/ICS Impact: Historians

En la industria, las bases de datos SQL son el corazón de los **Historians** (OSIsoft PI, Wonderware Historian a menudo usan SQL Server o interfaces SQL).

- **Riesgo Crítico:** Estas BBDD almacenan años de datos de producción. Un `DROP TABLE` o un ransomware aquí destruye el registro de calidad y cumplimiento legal de la fábrica.
    
- **Architecture:** A menudo se encuentran en la Zona 3 (Supervisión) del Modelo Purdue.
    

````

---

### Archivo 3: `05_Web_Server_Arsenal.md`
*Fusión de Apache, Web Server y Dir Scanning. Enfoque: Infraestructura Web.*

```markdown
# 🌐 Web Infrastructure Arsenal (Apache/IIS)

**Scope:** Enumeración de servidores web a nivel de infraestructura y directorios.
**Tools:** Metasploit Framework (Auxiliary Modules).

---

## 1. Server Fingerprinting & Headers
Identificar tecnología subyacente (Apache, Nginx, IIS) y módulos habilitados.

```bash
# Versión exacta del servidor
use auxiliary/scanner/http/http_version
set RHOSTS <TARGET_IP>
run

# Análisis de cabeceras de seguridad y métodos permitidos (GET, POST, PUT, DELETE)
use auxiliary/scanner/http/http_header
set RHOSTS <TARGET_IP>
run
````

---

## 2. Directory & File Enumeration

### The "Robots" Standard

Siempre es el primer paso. Busca el archivo de exclusión de robots.

Bash

```
use auxiliary/scanner/http/robots_txt
set RHOSTS <TARGET_IP>
run
```

### Directory Brute Force

Fuerza bruta para encontrar paneles de administración, backups o carpetas ocultas.

Bash

```
use auxiliary/scanner/http/dir_scanner
set RHOSTS <TARGET_IP>
set DICTIONARY /usr/share/wordlists/dirb/common.txt
set THREADS 20
run
```

### Apache Specific: UserDir Enumeration

Si el módulo `mod_userdir` está activo en Apache (común en configuraciones antiguas), permite enumerar usuarios del sistema operativo a través de `http://server/~user/`.

Bash

```
use auxiliary/scanner/http/apache_userdir_enum
set RHOSTS <TARGET_IP>
set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt
run
```

---

## 3. Vulnerable Methods (HTTP PUT)

Verificar si el servidor permite subir archivos arbitrarios (potencial RCE via Webshell).

Bash

```
use auxiliary/scanner/http/http_put
set RHOSTS <TARGET_IP>
set PATH /uploads/
set FILENAME test_vuln.txt
set FILEDATA "Security Check"
run
```

---

## 🏭 OT/ICS Impact: The HMI

En entornos modernos, la **HMI (Human Machine Interface)** suele ser una aplicación web corriendo sobre Apache o IIS en un panel táctil.

- **Peligro:** Enumerar o atacar estos servidores web puede congelar la pantalla del operador. Si el operador pierde visión del proceso (ceguera), no puede reaccionar ante una emergencia física.
    

```

---

### Siguiente Paso
Has completado la **Fase de Enumeración de Servicios**.
Ahora tienes:
1.  Infraestructura de Red (SSH/SMTP).
2.  Bases de Datos (MySQL).
3.  Servidores Web (Apache).

¿Listo para el **Lote 3 (Vulnerabilidades y Explotación)**?
Aquí es donde entraremos en `Nessus`, `EternalBlue` y `Payloads`. Sube el ZIP cuando quieras.
```
