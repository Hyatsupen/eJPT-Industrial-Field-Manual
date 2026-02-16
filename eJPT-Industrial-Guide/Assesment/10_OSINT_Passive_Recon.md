---
creado: 2026-02-16 18:06
tags:
  - tipo/apunte
  - ejpt
  - recon
  - passive
  - osint
asignatura: Ciberseguridad Ofensiva
---

### DNS Enumeration (Map the Network)

Mapeo de infraestructura a través de registros DNS.

- **DNS Dumpster:** [https://dnsdumpster.com](https://gemini.google.com/app/fc5f5ce169fecd63) (Visualización gráfica de activos).

- **Sublist3r:** Enumeración de subdominios usando motores de búsqueda.
    Bash
    
    ```
   sublist3r -d <DOMAIN.COM> -e google,yahoo,bing
    ```
    
- **DNS Recon:**
    
    Bash
    
    ```
    dnsrecon -d <DOMAIN.COM>
    ```
    
- **Transferencia de Zona (Manual):**
    
    Intento de obtener la topología completa del dominio (Rara vez funciona en configuraciones modernas, crítico si falla).
    
    Bash
    
    ```
    dnsenum <DOMAIN.COM>
    ```
    

### WAF Detection (Firewall Check)

Identificar si el objetivo está detrás de Cloudflare, Imperva o AWS WAF antes de lanzar escaneos activos.

Bash

```
wafw00f <DOMAIN.COM>
```

---

## 2. Web Technology Stack (Fingerprinting)

Determinar qué corre el objetivo para buscar CVEs específicos antes de atacar.

- **Browser Extensions:** Wappalyzer, BuiltWith.
    
- **Command Line:**
    
    Bash
    
    ```
    whatweb <URL>
    ```
    
- **Mirroring (Análisis Offline):**
    
    Descargar el sitio para analizar código fuente, comentarios y metadatos sin generar tráfico constante.
    
    Bash
    
    ```
    # CLI
    httrack <URL>
    # GUI
    webhttrack
    ```
    

---

## 3. Google Dorks (Information Leakage)

Uso de operadores avanzados para encontrar archivos expuestos, paneles de admin o configuraciones sensibles.

|**Dork**|**Descripción**|
|---|---|
|`site:target.com`|Limita la búsqueda al dominio objetivo.|
|`inurl:admin`|Busca paneles de administración en la URL.|
|`filetype:pdf`|Filtra por documentos (pdf, docx, xlsx, conf).|
|`intitle:"index of"`|**CRÍTICO:** Directorios abiertos (Directory Listing).|
|`cache:target.com`|Ver versión antigua almacenada por Google.|

**Ejemplo Combinado:**

Plaintext

```
site:target.com filetype:pdf OR filetype:docx
site:target.com intitle:"index of" "backup"
```

---

## 4. Email & Credential Harvesting

- **TheHarvester:** Recolección automatizada de emails, subdominios y nombres de empleados desde fuentes públicas.
    
    Bash
    
    ```
    theHarvester -d <DOMAIN> -b all
    ```
    
- **Breach Data:** Verificar si las cuentas han sido comprometidas previamente (HaveIBeenPwned).
    

````

---

### Archivo 2: `02_Active_Discovery_Nmap.md`
*Fusión de tus notas de Nmap, Ping Sweep y Evasión de Firewall. He corregido el comando MTU y añadido la sección OT.*

```markdown
# ⚔️ Active Reconnaissance & Nmap Methodology

**Scope:** Interacción directa con los puertos y servicios del objetivo.
**Warning:** Esta fase genera ruido en los logs. Ajustar `Timing Templates` según el entorno.

---

## 1. Host Discovery (Ping Sweep)

Identificar activos vivos sin escanear puertos (ahorro de tiempo).

```bash
# ARP Scan (Solo red local - Capa 2)
netdiscover -r <NETWORK/24>

# ICMP Echo Sweep (Clásico)
nmap -sn -PE <NETWORK/24>

# TCP SYN Discovery (Bypass ICMP block)
# Envía paquetes SYN a puertos comunes (80, 443, 445) para detectar host vivo.
nmap -sn -PS21,22,80,445,3389 <TARGET>
````

---

## 2. Port Scanning & Service Enumeration

### Standard Scans

Bash

```
# Stealth Scan (SYN - Default root). No completa el 3-way handshake.
nmap -sS <TARGET>

# TCP Connect Scan (User privs). Ruidoso, completa conexión.
nmap -sT <TARGET>

# UDP Scan (DNS, SNMP, NTP). Lento pero crítico.
nmap -sU --top-ports 100 <TARGET>
```

### Advanced Enumeration (Services & OS)

Bash

```
# Service Version + OS Detection + Default Scripts + Traceroute
nmap -A -T4 <TARGET>

# Sólo versiones (Menos ruido que -A)
nmap -sV <TARGET>

# Escaneo de todos los puertos (Full Range)
nmap -p- -sS -T4 --min-rate 1000 <TARGET>
```

---

## 3. Firewall Evasion & IDS Bypassing

Técnicas para evadir filtros de paquetes y reglas de Snort/Suricata.

### Fragmentation

Fragmentar paquetes obliga al Firewall a reensamblarlos (consumo CPU) o dejarlos pasar.

Bash

```
# Fragmentar paquetes (MTU debe ser múltiplo de 8)
nmap --mtu 24 <TARGET>
nmap -f <TARGET>
```

### Decoys (Señuelos)

Ocultar tu IP real entre IPs falsas para confundir al analista del SOC.

Bash

```
# RND: Genera IPs aleatorias. ME: Tu IP real.
nmap -D RND:10,ME <TARGET>
```

### Source Port Manipulation

Muchos firewalls confían ciegamente en tráfico origen puerto 53 (DNS) o 80 (HTTP).

Bash

```
nmap --source-port 53 <TARGET>
nmap -g 80 <TARGET>
```

### Packet Padding (Data Length)

Añadir basura binaria al final del paquete para cambiar su firma estática.

Bash

```
# Añade 25 bytes de datos aleatorios
nmap --data-length 25 <TARGET>
```