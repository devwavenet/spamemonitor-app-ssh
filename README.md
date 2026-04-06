# SPAMEMONITOR 🛡️

Monitor de Spam interactivo para servidores con **Exim4 + SpamExperts**. Script Bash diseñado para ejecutar por SSH directamente en el servidor antispam.

> Herramienta de investigación, detección y prevención de spam, phishing, brute-force y ataques de reputación contra servidores de correo.

---

## 📋 Tabla de Contenidos

- [Descripción General](#-descripción-general)
- [Requisitos](#-requisitos)
- [Instalación](#-instalación)
- [Uso](#-uso)
- [Funciones del Script](#-funciones-del-script)
  - [Sección: Spam Detection](#1-spam-detection-opciones-1-4)
  - [Sección: Mail Management](#2-mail-management-opciones-5-9)
  - [Sección: Blacklists](#3-blacklists-opción-10)
- [Arquitectura del Sistema](#-arquitectura-del-sistema)
- [Logs que Utiliza](#-logs-que-utiliza)
- [Correcciones Realizadas](#-correcciones-realizadas)
- [Patrones de Ataque Detectables](#-patrones-de-ataque-detectables)
- [Guía de Investigación](#-guía-de-investigación)
- [Limitaciones Actuales](#-limitaciones-actuales)
- [Mejoras Futuras Sugeridas](#-mejoras-futuras-sugeridas)
- [Estructura del Proyecto](#-estructura-del-proyecto)
- [Contribuciones](#-contribuciones)
- [Licencia](#-licencia)

---

## 📝 Descripción General

SPAMEMONITOR es un script Bash interactivo que se ejecuta directamente en el servidor antispam (vía SSH) y proporciona un menú con **10 funciones de análisis** organizadas en 3 categorías:

| Categoría | Funciones | Propósito |
|-----------|-----------|-----------|
| **Spam Detection** | Opciones 1-4 | Detectar spam entrante/saliente y clasificaciones "unsure" |
| **Mail Management** | Opciones 5-9 | Investigar correos por remitente, IP, cola y cuarentena |
| **Blacklists** | Opción 10 | Monitorear IPs bloqueadas por RBLs (Spamhaus, Hostkarma, SpamRL) |

### ¿Qué lo hace único?

- **Trabaja con logs reales** del servidor (`/var/log/exim4/mainlog` y `/var/log/spamexperts/local_scan.log`)
- **Correlaciona datos** entre Exim4 (MTA) y SpamExperts (filtro antispam)
- **No requiere instalación** de dependencias externas (usa herramientas estándar: `grep`, `awk`, `sed`, `sort`, `uniq`)
- **Soporte para logs comprimidos** con `zgrep`
- **Interfaz visual** con colores para facilitar la lectura

---

## ✅ Requisitos

- Servidor con **Exim4** como MTA
- **SpamExperts** como filtro antispam (local_scan)
- Acceso **SSH root** al servidor
- Herramientas estándar de Linux: `grep`, `zgrep`, `awk`, `sed`, `sort`, `uniq`, `tr`, `head`, `clear`
- Logs disponibles en:
  - `/var/log/exim4/mainlog`
  - `/var/log/spamexperts/local_scan.log`

---

## 📦 Instalación

### Método 1: Copy-Paste directo (recomendado)

```bash
ssh root@antispam-server
nano /usr/local/bin/spamemonitor.sh
# Pegar el contenido del script
chmod +x /usr/local/bin/spamemonitor.sh
```

### Método 2: Clonar y ejecutar

```bash
git clone https://github.com/tu-usuario/spamemonitor.git
cd spamemonitor
chmod +x spamemonitor.sh
./spamemonitor.sh
```

### Ejecutar directamente

```bash
./spamemonitor.sh
```

---

## 🖥️ Uso

Al ejecutar el script, se muestra un menú interactivo:

```
=================================
   SPAMEMONITOR - MENU
=================================

--- SPAM DETECTION ---
1) Correos Outgoings detectados como SPAM 📤
2) Correos Incomings detectados como SPAM 📥
3) Dominios Incomings detectados como SPAM 🌐📥
4) Correos Outgoings detectados como UNSURE 📤

--- MAIL MANAGEMENT ---
5) Lista completa de correos enviados 📨
6) Correos ENTRANTES desde IP 📥📍
7) Correos SALIENTES por IP 📤📍
8) Ver correo en cuarentena por Message ID 🔍🛡️
9) Ver todos los mensajes en cola de envío  📬⏳

--- BLACKLISTS ---
10) IPs en Blacklists (RBLs) 📍🚫

0) Salir
```

Cada opción ejecuta una función específica y muestra resultados en pantalla. Presionar **Enter** para volver al menú después de cada consulta.

---

## 🔍 Funciones del Script

### 1. SPAM DETECTION (Opciones 1-4)

#### Opción 1: Correos Outgoings detectados como SPAM 📤

**Función:** `outgoings_email_spam()`

**Qué hace:** Detecta qué **cuentas de usuario** están enviando spam saliente desde dentro de la red gestionada.

**Fuente de datos:** `/var/log/spamexperts/local_scan.log`

**Cómo funciona:**
1. Busca `"Queuing outgoing spam message"` en el log de SpamExperts
2. Obtiene la línea anterior con `zgrep -B1` para extraer `"Submission identity"`
3. Extrae la identidad del remitente autenticado
4. Agrupa y ordena por cantidad (mayor ofensor primero)

**Qué revela:** Si una cuenta legítima fue comprometida y está enviando spam masivo, aparecerá en el top con un número anormalmente alto.

**Ejemplo de salida:**
```
547 usuario-comprometido@dominio.com
23 otro-usuario@dominio.com
5 cuenta-legitima@dominio.com
```

**Caso de uso real:** Detectar cuentas comprometidas por malware o credenciales robadas que envían spam a través del servidor.

---

#### Opción 2: Correos Incomings detectados como SPAM 📥

**Función:** `incomings_email_spam()`

**Qué hace:** Detecta qué **destinatarios** están recibiendo más spam entrante. Permite buscar un email específico o ver todos.

**Fuente de datos:** `/var/log/spamexperts/local_scan.log`

**Cómo funciona:**
1. Busca `"Queuing incoming spam message"` en el log
2. Extrae los destinatarios después de `"for "`
3. Separa múltiples destinatarios (un correo puede tener varios)
4. Limpia espacios y puntos finales
5. Filtra solo direcciones con `@`
6. Permite filtrar por email específico o mostrar todos
7. Agrupa y ordena por cantidad

**Qué revela:** Si un destinatario específico recibe cantidades anormales de spam, puede indicar que su dirección fue publicada en listas de spam o es víctima de un ataque dirigido.

**Ejemplo de salida:**
```
200 administrador@empresa.com.ar
85 ventas@empresa.com.ar
12 info@empresa.com.ar
```

**Caso de uso real:** Identificar qué buzones son blanco de ataques para aplicar reglas de filtrado específicas o verificar configuraciones SPF/DMARC del dominio.

---

#### Opción 3: Dominios Incomings detectados como SPAM 🌐📥

**Función:** `incomings_domain_spam()`

**Qué hace:** Similar a la opción 2, pero muestra **dominios completos** en lugar de emails individuales. Detecta ataques masivos contra dominios enteros.

**Fuente de datos:** `/var/log/spamexperts/local_scan.log`

**Cómo funciona:**
1. Misma búsqueda que opción 2: `"Queuing incoming spam message"`
2. Misma extracción y limpieza
3. **Diferencia clave:** `grep -v '@'` — excluye emails, deja solo dominios sueltos
4. Agrupa y ordena por cantidad

**Qué revela:** Si un dominio entero recibe spam masivo (no solo un buzón), puede indicar un data breach, configuración DNS débil, o ataque generalizado.

**Ejemplo de salida:**
```
500 wavenet.com.ar
45 emba.com.ar
12 yuhmak.com.ar
```

**Caso de uso real:** Si un dominio aparece con números desproporcionados respecto a otros, requiere investigación de sus registros DNS (SPF, DMARC, DKIM).

---

#### Opción 4: Correos Outgoings detectados como UNSURE 📤

**Función:** `unsure_outgoings_email_spam()`

**Qué hace:** Detecta correos salientes clasificados como **"unsure"** (incierto). No son spam confirmado, pero tampoco son claramente legítimos. Es la **zona gris** que requiere atención.

**Fuente de datos:** `/var/log/spamexperts/local_scan.log`

**Cómo funciona:**
1. Busca `"main: unsure"` en el log (clasificación principal de SpamExperts)
2. Usa `zgrep -A1` para obtener la línea siguiente con `"Submission identity"`
3. Extrae la identidad, agrupa y ordena

**Qué revela:** Esta es la función más valiosa para detectar **falsos positivos**. Sistemas automáticos internos sin DKIM, con contenido genérico o patrones repetitivos caen aquí.

**Caso de uso real detectado:** El sistema `autogestion@civiles.org.ar` generaba 58+ correos clasificados como "unsure" con asuntos como "Error al Actualizar Boletas desde Autogestion". No era spam real, sino un sistema interno sin firma DKIM que SpamExperts no podía clasificar claramente.

**Solución recomendada:** Agregar la identidad a whitelist o configurar DKIM para esos correos.

---

### 2. MAIL MANAGEMENT (Opciones 5-9)

#### Opción 5: Lista completa de correos enviados 📨

**Función:** `filter_exim_by_sender()`

**Qué hace:** Lista **TODOS los remitentes** que enviaron correo a través del servidor, ordenados por cantidad de mensajes. Permite buscar un remitente específico.

**Fuente de datos:** `/var/log/exim4/mainlog`

**Cómo funciona:**
1. Lee el mainlog completo con un script `awk`
2. Extrae el ID del mensaje: `match($0, /\] ([0-9A-Za-z-]+) /)`
3. Extrae el remitente con 3 patrones (en orden):
   - `<= remitente@dominio` (entrada de correo)
   - `MAIL FROM:<remitente@dominio>` (comando SMTP)
   - `F=remitente@dominio` (campo del log)
4. Guarda en array asociativo `sender[id] = from` (evita duplicados)
5. Excluye bounces (`from != "<>"`)
6. Convierte a minúsculas
7. Cuenta, ordena y limita a top 100

**Qué revela:** Un remitente con volumen anormalmente alto indica cuenta comprometida, sistema mal configurado, o ataque de backscatter.

**Ejemplo de salida:**
```
15000 root@servidor-comprometido.com
342 leandro@learsystem.com.ar
89 turnos-services@yuhmak.com.ar
```

---

#### Opción 6: Correos ENTRANTES desde IP 📥📍

**Función:** `filter_exim_incoming_by_ip()`

**Qué hace:** Busca todos los correos **entrantes** que llegaron desde una **IP específica**. Permite investigar si una IP sospechosa está enviando correo a tus usuarios.

**Fuente de datos:** `/var/log/exim4/mainlog`

**Cómo funciona:**
1. Pide la IP al usuario
2. Filtra líneas con `<=` (entrada) Y `H=... [` (host con IP)
3. Extrae: ID del mensaje, remitente, IP del remitente
4. Si la IP coincide, cuenta el remitente
5. Muestra remitentes desde esa IP, ordenados

**Qué revela:** Permite ver TODOS los correos que llegaron desde una IP sospechosa, confirmar si es spam o legítimo, y tomar decisión de bloqueo.

**Ejemplo de salida (IP atacante):**
```
26 root@vps18.falesiaho.com
```

**Ejemplo de salida (IP legítima):**
```
3 moraskate02@gmail.com
```

---

#### Opción 7: Correos SALIENTES por IP 📤📍

**Función:** `filter_exim_outgoing_by_ip()`

**Qué hace:** Busca todos los correos **salientes** que tu servidor envió usando una **interfaz IP específica**. Permite rastrear qué usuarios envían correo por una IP de salida particular.

**Fuente de datos:** `/var/log/exim4/mainlog`

**Cómo funciona:**
1. Pide la IP al usuario
2. **Primera pasada:** guarda todos los remitentes por ID de mensaje (`<=`)
3. **Segunda pasada:** busca entregas (`=>`) por IP de interfaz (`I=[IP]`)
4. Cruza los datos: si la IP de interfaz coincide, busca el remitente original
5. Muestra remitentes que usaron esa IP

**Diferencia clave con opción 6:**
- Opción 6: `H=host [IP]` → IP del **remitente externo** (entrante)
- Opción 7: `I=[IP]` → IP de la **interfaz local** de salida (saliente)

**Qué revela:** Si una IP de salida está siendo usada para spam masivo, permite identificar qué remitente la usa.

---

#### Opción 8: Ver correo en cuarentena por Message ID 🔍🛡️

**Función:** `search_quarantined_mails()`

**Qué hace:** Busca un mensaje específico en el **directorio de cuarentena** de SpamExperts. Permite recuperar o inspeccionar mensajes puestos en cuarentena.

**Fuente de datos:** `/var/spool/mail/w/wa/` (directorio de cuarentena)

**Cómo funciona:**
1. Verifica que exista el directorio de cuarentena
2. Pide el Message ID al usuario
3. Usa `grep -Rl` para buscar recursivamente el ID en todos los archivos
4. Muestra la ruta del archivo encontrado

**Qué revela:** Cuando SpamExperts clasifica un mensaje como spam/phish pero no lo rechaza, lo pone en cuarentena. Esta función permite encontrarlo y recuperarlo si es un falso positivo.

---

#### Opción 9: Ver todos los mensajes en cola de envío 📬⏳

**Función:** `show_exim_queue()`

**Qué hace:** Muestra los mensajes actualmente en la **cola de salida** de Exim4 (esperando ser entregados). Permite buscar por email.

**Fuente de datos:** Cola de Exim4 (`exim -bp`) + configuración `/var/lib/exim4/outgoing-config.autogenerated`

**Cómo funciona:**
1. Verifica que exista el archivo de configuración de Exim
2. Ejecuta `exim -C "$CONFIG" -bp` para listar la cola
3. Filtra por email ingresado (o muestra todo si está vacío)

**Qué revela:** Mensajes atascados por:
- Connection timeout (servidor destino no responde)
- Network unreachable (IPv6 no disponible)
- Errores SMTP temporales (4xx)
- Mensajes frozen (congelados por error permanente)

---

### 3. BLACKLISTS (Opción 10)

#### Opción 10: IPs en Blacklists (RBLs) 📍🚫

**Función:** `check_blacklist_drops()`

**Qué hace:** Genera un **reporte completo** de todas las IPs bloqueadas por listas negras. Consulta **dos fuentes de logs** para obtener una visión completa del estado de blacklisting.

**Fuentes de datos:**
- `/var/log/exim4/mainlog` → para Spamhaus y SpamRL
- `/var/log/spamexperts/local_scan.log` → para Hostkarma

**Cómo funciona:**

**Sección 1 — Spamhaus (ZEN/DBL/HBL):**
1. Busca `"spamhaus"`, `"zen"`, `"dbl"`, `"hbl"` en el mainlog
2. Extrae la IP del **host remoto** con `sed` (campo `H=hostname [IP]`)
3. Excluye la IP local del servidor (`I=[IP]`)
4. Cuenta y muestra top 10

**Sección 2 — Hostkarma (Yellow Listed):**
1. Busca `"Yellow listed on Hostkarma"` en local_scan.log
2. Cuenta las ocurrencias totales

**Sección 3 — SpamRL / Otras listas:**
1. Busca `"spamrl"`, `"listed on"` en el mainlog
2. Extrae la IP del host remoto (mismo proceso que Spamhaus)
3. Muestra top 10

**Sección 4 — Resumen:**
- Total de eventos Spamhaus
- Total de advertencias Hostkarma
- Total de eventos SpamRL

**Qué revela:**
- IPs de atacantes recurrentes
- Redes de spam organizadas
- Posibles delistings necesarios si una IP legítima aparece listada

**Ejemplo de salida corregida:**
```
🚫 Spamhaus (ZEN/DBL/HBL):
  26 bloqueos - IP: 194.38.20.143
  22 bloqueos - IP: 194.38.20.68
  21 bloqueos - IP: 194.38.21.42
  19 bloqueos - IP: 194.38.21.70

🟡 Hostkarma (Yellow Listed):
  Total advertencias: 2938

🔴 SpamRL / Otras listas:
  26 - IP: 194.38.20.143
  22 - IP: 194.38.20.68
  21 - IP: 194.38.21.42
```

---

## 🏗️ Arquitectura del Sistema

```
┌──────────────────────────────────────────────────────────────────┐
│                     FLUJO DE CORREO ENTRANTE                      │
│                                                                   │
│  Remitente → Internet → antispam:25 (I=[45.173.0.25])            │
│                            │                                      │
│                            ▼                                      │
│                  ┌──────────────────┐                              │
│                  │  Exim4 mainlog   │ ← Conexión, AUTH, TLS, HELO │
│                  │  (recepción)     │   IP, rechazos, rate limit   │
│                  └────────┬─────────┘                              │
│                           │                                        │
│                           ▼                                        │
│                  ┌──────────────────┐                              │
│                  │  SpamExperts     │ ← DKIM, DMARC, SPF, CRM114  │
│                  │  local_scan.log  │   Pyzor, DNSBL, heurísticas  │
│                  └────────┬─────────┘                              │
│                           │                                        │
│            ┌──────────────┼──────────────┐                         │
│            ▼              ▼              ▼                         │
│       not-spam         unsure          spam/phish                  │
│            │              │              │                         │
│            ▼              ▼              ▼                         │
│     Entrega normal   Se entrega    Se encola/rechaza               │
│                      (flagged)                                     │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                     FLUJO DE CORREO SALIENTE                      │
│                                                                   │
│  Cliente autenticado → antispam:587 o :465                        │
│  (Submission identity: usuario@dominio)                           │
│                            │                                      │
│                            ▼                                      │
│                  ┌──────────────────┐                              │
│                  │  Exim4 mainlog   │ ← Autenticación, envío       │
│                  │  (envío)         │                              │
│                  └────────┬─────────┘                              │
│                           │                                        │
│                           ▼                                        │
│                  ┌──────────────────┐                              │
│                  │  SpamExperts     │ ← Clasificación outgoing     │
│                  │  local_scan.log  │                              │
│                  └────────┬─────────┘                              │
│                           │                                        │
│            ┌──────────────┼──────────────┐                         │
│            ▼              ▼              ▼                         │
│       not-spam         unsure          spam                        │
│            │              │              │                         │
│            ▼              ▼              ▼                         │
│     Se envía normal  Se envía       Se bloquea                     │
│                      (flagged)                                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📂 Logs que Utiliza

### `/var/log/exim4/mainlog`

Log principal de Exim4. Registra **todos** los eventos de correo:

| Tipo de entrada | Ejemplo | Qué registra |
|-----------------|---------|-------------|
| Entrada de correo | `<= sender@domain H=host [IP]` | Correo recibido |
| Entrega exitosa | `=> recipient@domain R=router T=transport` | Correo entregado |
| Error fatal | `** recipient@domain` | Entrega fallida |
| Rechazo | `rejected RCPT/MAIL/AUTH` | Rechazo por política |
| Autenticación fallida | `login authenticator failed for` | Brute-force |
| Rate limiting | `IP is sending too fast` | Control de velocidad |
| DMARC | `Warning: DMARC DEBUG: 'accept'` | Verificación DMARC |
| TLS error | `TLS error on connection` | Problemas de certificado |
| Completado | `Completed` | Mensaje procesado |
| Spamhaus | `rejected by local_scan(): ... Spamhaus ZEN` | Bloqueo por RBL |

### `/var/log/spamexperts/local_scan.log`

Log detallado de SpamExperts. Registra el **análisis profundo** de cada mensaje:

| Tipo de entrada | Qué registra |
|-----------------|-------------|
| `Submission identity` | Identidad del remitente autenticado |
| `main: not-spam/unsure/spam/phish` | Clasificación principal |
| `Combined score` | Puntuación CRM114 |
| `DKIM result` | Estado de firma DKIM |
| `DMARC recommendation` | Resultado DMARC |
| `SPF softfail` | Fallo suave de SPF |
| `Yellow listed on Hostkarma` | Advertencia de reputación |
| `Queuing incoming/outgoing spam message` | Mensaje encolado como spam |
| `Handled message in X seconds` | Tiempo de procesamiento |
| `pyzor.org: Hits` | Resultado de Pyzor |

---

## 🔧 Correcciones Realizadas

### Corrección: Blacklist Drops — IP del servidor contaminando resultados

**Problema:** La función `check_blacklist_drops()` mostraba la IP del servidor local (`45.173.0.25`) como la #1 en bloqueos de Spamhaus con 718 registros, porque el regex capturaba **todas** las IPs entre corchetes, incluyendo `I=[45.173.0.25]` (interfaz local).

**Causa:** El regex original `grep -oE '\[([0-9.]+)\]'` no distinguía entre:
- `H=hostname [194.38.20.143]` ← IP del atacante (la que se necesita)
- `I=[45.173.0.25]:25` ← IP del servidor local (la que NO se necesita)

**Solución:** Se reemplazó el regex en ambas secciones (Spamhaus y SpamRL):

```bash
# ANTES (incorrecto)
grep -oE '\[([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)\]' | tr -d '[]'

# DESPUÉS (correcto)
sed -n 's/.*H=[^ ]* \[\([0-9.]*\)\].*/\1/p'
```

El `sed` extrae **exclusivamente** la IP del campo `H=` (host remoto), ignorando completamente `I=` (interfaz local).

**Resultado:** Ahora el reporte muestra solo IPs de atacantes reales, sin contaminación con la IP del servidor.

---

## 🚨 Patrones de Ataque Detectables

### 1. Brute-force de credenciales SMTP
- **Indicadores:** `login authenticator failed`, `denied authentication due to possible brute-force`
- **Ejemplo real:** IP `185.174.103.55` con HELOs aleatorios (`szHMlmR`, `u2ttRsSC`, `1WRMVbax2j`)
- **Funciones que lo detectan:** Opción 5, 6, 10

### 2. Spam desde VPS comprometidos
- **Indicadores:** `root@vps*.falesiaho.com`, IPs `194.38.20.x/21.x` en Spamhaus ZEN
- **Ejemplo real:** 4 VPS diferentes con rechazos constantes
- **Funciones que lo detectan:** Opción 6, 10

### 3. Red de spam recurrente
- **Indicadores:** Múltiples IPs del mismo dominio enviando spam
- **Ejemplo real:** `nugetcenter.com` desde `209.126.112.44` y `.45`
- **Funciones que lo detectan:** Opción 5, 10

### 4. Ataques persistentes (ráfagas)
- **Indicadores:** Múltiples intentos en corto tiempo desde misma IP
- **Ejemplo real:** `giize.com` (`94.156.152.219`) con 10+ rechazos
- **Funciones que lo detectan:** Opción 6, 10

### 5. Phishing entrante
- **Indicadores:** `main: phish, sub: pattern`, `Reject phishing with no bounce`
- **Ejemplo real:** `antispamcloud.phish.genphs4267`
- **Funciones que lo detectan:** Opción 2, 3

### 6. Falsos positivos de sistemas internos
- **Indicadores:** `main: unsure` recurrente del mismo remitente, sin DKIM
- **Ejemplo real:** `autogestion@civiles.org.ar` con 58+ registros "unsure"
- **Funciones que lo detectan:** Opción 4, 5

### 7. Scanning de buzones
- **Indicadores:** `rejected RCPT ... User unknown`, HELO falso
- **Ejemplo real:** `190.247.241.87` con HELO `(gmx.com)` falso
- **Funciones que lo detectan:** Opción 6, 10

### 8. Spam con URLs en blacklists
- **Indicadores:** `An URL in this email (...) is listed by Spamhaus DBL/HBL`
- **Ejemplo real:** `instatool.site`, `mobwebify.com`
- **Funciones que lo detectan:** Opción 2, 10

---

## 🧭 Guía de Investigación

Cuando detectas un problema de spam, seguí este flujo:

```
PASO 1: ¿Hay spam saliente? (cuentas comprometidas)
├── Opción 1 → Spam saliente confirmado
├── Opción 4 → Unsure saliente (zona gris)
└── Opción 5 → Volumen general de envío

    Si cuenta con números altos en Opción 1 → Cuenta comprometida
    Si cuenta aparece en Opción 4 → Posible falso positivo

PASO 2: ¿Hay spam entrante? (ataques externos)
├── Opción 2 → Spam por email destinatario
└── Opción 3 → Spam por dominio

    Si dominio/email con números altos → Verificar SPF/DMARC

PASO 3: ¿De dónde viene el spam? (investigar origen)
├── Opción 6 → Buscar correos entrantes desde IP sospechosa
└── Opción 10 → Ver si la IP está en blacklists

    Si IP está en Spamhaus → Confirmar que el bloqueo funciona

PASO 4: ¿Hay mensajes retenidos? (recuperar falsos positivos)
├── Opción 8 → Buscar en cuarentena por Message ID
└── Opción 9 → Ver cola de envío (mensajes atascados)
```

---

## ⚠️ Limitaciones Actuales

1. **No muestra detalles del análisis SpamExperts** — Solo la clasificación final, no scores, DKIM, DMARC
2. **No correlaciona entre logs** — Para ver el análisis completo de un mensaje, hay que buscar manualmente en ambos logs
3. **No detecta brute-force automáticamente** — Los intentos fallidos de autenticación no se reportan como sección dedicada
4. **No muestra tendencias temporales** — No dice "en la última hora hubo X spams"
5. **No alerta sobre dominios nuevos sospechosos** — `"SEM new sender domain"` aparece en los logs pero no se reporta
6. **No reporta rate limiting** — `"is sending too fast"` no se monitorea
7. **La cuarentena requiere Message ID exacto** — No hay forma de listar toda la cuarentena

---

## 🚀 Mejoras Futuras Sugeridas

### A) Detección de Brute-force

```bash
brute_force_detection() {
  echo -e "\n${COLOR_RED}=== Intentos de Brute-force SMTP ===${NC}\n"
  grep "login authenticator failed" /var/log/exim4/mainlog | \
    grep -oE '\[([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)\]' | \
    tr -d '[]' | sort | uniq -c | sort -nr | head -20 | \
    awk -v y="$COLOR_YELLOW" -v n="$NC" '{printf "%s%d%s intentos desde IP: %s\n", y, $1, n, $2}'
}
```

### B) Monitoreo de Rate Limiting

```bash
rate_limit_monitor() {
  echo -e "\n${COLOR_YELLOW}=== IPs con Rate Limit Excedido ===${NC}\n"
  grep "sending too fast" /var/log/exim4/mainlog | \
    grep -oE '^[0-9-]+ [0-9:]+ [0-9]+ \] [0-9.]+ is sending too fast' | \
    grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | \
    sort | uniq -c | sort -nr | head -20
}
```

### C) Detalle por Message ID (correlación entre logs)

```bash
message_detail() {
  read -p "Ingrese el Message ID: " msg_id
  echo -e "\n${COLOR_YELLOW}=== Exim4 mainlog ===${NC}"
  grep "$msg_id" /var/log/exim4/mainlog
  echo -e "\n${COLOR_YELLOW}=== SpamExperts local_scan ===${NC}"
  grep "$msg_id" /var/log/spamexperts/local_scan.log
}
```

### D) Candidatos a Whitelist

```bash
whitelist_candidates() {
  echo -e "\n${COLOR_YELLOW}=== Candidatos a Whitelist (unsure recurrentes) ===${NC}\n"
  zgrep "main: unsure" /var/log/spamexperts/local_scan.log -A1 | \
    grep "Submission identity" | \
    awk -F': ' '{print $2}' | \
    sort | uniq -c | sort -nr | head -20
}
```

### E) Fallos de DMARC

```bash
dmarc_failures() {
  echo -e "\n${COLOR_RED}=== Fallos de DMARC (Quarantine/Reject) ===${NC}\n"
  grep -i "DMARC.*[Qq]uarantine\|DMARC.*[Rr]eject" /var/log/exim4/mainlog | \
    grep -oE 'H=[^ ]+ \[[0-9.]+\]' | \
    sort | uniq -c | sort -nr | head -20
}
```

### F) Top Dominios Atacados

```bash
top_attacked_domains() {
  echo -e "\n${COLOR_RED}=== Top Dominios más Atacados por Spam ===${NC}\n"
  zgrep "Queuing incoming spam message" /var/log/spamexperts/local_scan.log | \
    awk -F'for ' '{print $2}' | \
    tr ',' '\n' | \
    sed 's/[[:space:]]//g; s/\.$//' | \
    grep '@' | \
    awk -F'@' '{print $2}' | \
    sort | uniq -c | sort -nr | head -20
}
```

---

## 📁 Estructura del Proyecto

```
proyecto-testeo/
├── spam-scripts-optimizado.txt    # Script principal SPAMEMONITOR
├── README.md                      # Este archivo (documentación)
├── direcciones-para-analisis.txt  # Direcciones IP para análisis
├── ejemplo-desglose.txt           # Ejemplo de formato de desglose
└── Ejemplo-de-logs/               # Logs de ejemplo para análisis
    ├── logs-ip-server.txt                    # grep por IP del servidor
    ├── reporte-directo-unsure.txt            # grep por identidad unsure
    ├── Reprote-spam.txt                      # grep por Spamhaus
    ├── logs-exim4-mainlog/
    │   ├── exim4-mainlog.txt                 # Mainlog día 1
    │   ├── exim4-mainlog2.txt                # Mainlog día 2
    │   └── exim4-mainlog3.txt                # Mainlog día 3
    ├── logs-spamexperts-local_scan/
    │   ├── spamexperts-local_scan.log.txt    # Local scan día 1
    │   ├── spamexperts-local_scan2.log.txt   # Local scan día 2
    │   └── spamexperts-local_scan3.log.txt   # Local scan día 3
    └── logs_web-antispam-experts/
        └── CSV Export OutgoingLogSearch.csv  # Export CSV de SpamExperts
```

---

## 🤝 Contribuciones

Las contribuciones son bienvenidas. Si querés agregar nuevas funciones al script:

1. Revisá la sección [Mejoras Futuras Sugeridas](#-mejoras-futuras-sugeridas) para ideas ya documentadas
2. Cada nueva función debe:
   - Validar que los logs existan antes de procesar
   - Usar `zgrep` en lugar de `grep` para soportar logs comprimidos
   - Mostrar resultados con colores consistentes
   - Incluir mensajes de error claros
3. Documentá la nueva función en este README siguiendo el formato existente

---

## 📄 Licencia

Uso interno. Proyecto de monitoreo y análisis de spam para servidores Exim4 + SpamExperts.

---

## 📊 Referencia Rápida de Clasificaciones SpamExperts

| Clasificación (`main:`) | Significado | Nivel de alerta |
|-------------------------|-------------|-----------------|
| `not-spam` | Legítimo | ✅ Verde |
| `unsure` | Incierto | 🟡 Amarillo |
| `spam` | Spam confirmado | 🔴 Rojo |
| `phish` | Phishing confirmado | 🔴 Rojo crítico |
| `whitelisted` | Lista blanca | ✅ Verde confirmado |

| Subclasificación (`sub:`) | Método de detección |
|---------------------------|---------------------|
| `combined` | Análisis combinado (CRM114 + otros) |
| `dnsbl` | DNS Blacklist (Spamhaus, SpamRL, etc.) |
| `heuristic` | Análisis heurístico (patrones, longitud) |
| `pattern` | Patrón conocido (firmas Sanesecurity) |
| `sender` | Reputación del remitente |

---

## 🔗 Recursos Útiles

- [Spamhaus ZEN](https://check.spamhaus.org/) — Verificar estado de IP
- [Spamhaus DBL](https://check.spamhaus.org/) — Verificar dominio en DBL
- [Spamhaus HBL](https://check.spamhaus.org/) — Verificar hash en HBL
- [Exim4 Documentation](https://www.exim.org/exim-html-current/doc/html/spec_html/) — Documentación oficial
- [SpamExperts Documentation](https://docs.spamexperts.com/) — Documentación oficial

---

> **Nota:** Este script está diseñado para funcionar en el servidor antispam directamente. No es una herramienta remota. Requiere acceso root y lectura de los logs del sistema.
# spamemonitor-app-ssh
# spamemonitor-app-ssh
