# ThreatHunting-Events

Bienvenido al repositorio `#ThreatHunting-Events`, una guía práctica para cazar amenazas usando Event IDs de Windows y herramientas SIEM. Este README detalla los eventos más importantes para detectar actividad sospechosa, desde logins anómalos hasta gestión de cuentas maliciosas, extraídos de materiales de threat hunting profesional.

## Introducción a los Event Logs

Los event logs de Windows son esenciales para identificar acciones maliciosas en endpoints. Este repositorio se enfoca en los logs principales y sus Event IDs clave para detecciones efectivas.

### Logs Principales de Windows
- **Application**: Errores e información de aplicaciones (ej. reportes de antivirus).
- **System**: Eventos de componentes del sistema (drivers, servicios, red).
- **Security**: Autenticaciones y eventos de seguridad (logons, privilegios).

### Logs Adicionales
- **Setup**: Configuración de aplicaciones.
- **Forwarded Events**: Eventos recolectados de sistemas remotos.
- **Applications and Services**: Logs específicos (ej. Sysmon, PowerShell, Windows Defender).

### Ubicaciones
- **Windows XP/2003**: `%SYSTEMROOT%\System32\Config\` (`.evt`).
- **Windows Vista+**: `%SYSTEMROOT%\System32\Winevt\Logs\` (`.evtx`).
- **Registro**: `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Eventlog`.

### Acceso
Usa **Event Viewer** (`eventvwr`) para revisar logs y consulta la documentación de Microsoft para detalles de Event IDs.

---

## Event IDs Clave para Threat Hunting

A continuación, los Event IDs más relevantes para detectar amenazas, organizados por categoría, con su significado y por qué son sospechosos.

### Eventos de Logon y Autenticación

| Event ID | Descripción                     | Por qué es Sospechoso                              | Consejos para Hunting                          |
|----------|---------------------------------|----------------------------------------------------|------------------------------------------------|
| **4624** | Inicio de sesión exitoso        | Logons desde IPs desconocidas o fuera de horario   | Revisar `Logon Type` (3: red, 10: RDP)         |
| **4625** | Inicio de sesión fallido        | Múltiples fallos sugieren brute force             | Buscar alta frecuencia por IP o usuario        |
| **4634** | Cierre de sesión exitoso        | Sesiones cortas tras logons sospechosos           | Correlacionar con 4624 por `LogonID`           |
| **4647** | Cierre iniciado por usuario     | Posible limpieza tras ataque                      | Verificar tras 4624 anómalo                   |
| **4648** | Logon con credenciales explícitas | Uso para elevar privilegios o movimiento lateral  | Revisar contexto (usuario, máquina)            |
| **4672** | Privilegios especiales asignados | Escalada de privilegios por malware               | Comprobar si el usuario debería tenerlos       |
| **4768** | Ticket Kerberos (TGT) solicitado | Posible Golden Ticket                            | Buscar TGTs anómalos o desde IPs externas      |
| **4769** | Ticket de servicio Kerberos     | Movimiento lateral (Pass-the-Ticket)              | Correlacionar con 4624 y revisar servicio      |
| **4771** | Pre-autenticación Kerberos fallida | Fuerza bruta o Kerberoasting                   | Analizar frecuencia y origen                  |
| **4776** | Validación de credenciales      | Fallos repetidos sugieren credential harvesting   | Revisar códigos de error (ej. 0xC000006D)      |
| **4778** | Sesión reconectada (ej. RDP)    | Reconexiones frecuentes desde IPs externas        | Correlacionar con 4624 (Logon Type 10)         |
| **4779** | Sesión desconectada            | Parte de patrón de acceso malicioso               | Verificar duración y origen                    |

### Eventos de Gestión de Cuentas

| Event ID | Descripción                     | Por qué es Sospechoso                              | Consejos para Hunting                          |
|----------|---------------------------------|----------------------------------------------------|------------------------------------------------|
| **4720** | Cuenta creada                   | Cuentas nuevas como backdoors                     | Revisar creador y hora                        |
| **4722** | Cuenta habilitada               | Reactivación de cuentas inactivas maliciosamente  | Verificar historial de la cuenta              |
| **4724** | Intento de resetear contraseña  | Restablecimientos no autorizados                  | Correlacionar con 4625 previos                |
| **4728** | Usuario añadido a grupo global  | Escalada de privilegios (ej. Domain Admins)       | Comprobar legitimidad del cambio              |
| **4732** | Usuario añadido a grupo local   | Escalada local (ej. Administrators)               | Analizar quién realizó el cambio              |
| **4736** | Usuario añadido a grupo universal | Ampliación de acceso en el dominio              | Revisar grupo y contexto                      |

### Otros Eventos Relevantes

| Event ID | Descripción                     | Por qué es Sospechoso                              | Consejos para Hunting                          |
|----------|---------------------------------|----------------------------------------------------|------------------------------------------------|
| **1102** | Log de auditoría borrado        | Atacantes ocultan rastros                         | Correlacionar con actividad previa sospechosa  |
| **4688** | Nuevo proceso creado            | Ejecución de binarios maliciosos                  | Revisar comando y padre (ej. rutas raras)      |
| **4697** | Servicio instalado              | Persistencia mediante servicios                   | Verificar nombre y ruta del servicio           |
| **7045** | Nuevo servicio registrado       | Similar a 4697, usado para backdoors              | Analizar ejecutable asociado                  |

---

## Logon Types

Los `Logon Types` en eventos como 4624 y 4625 indican cómo se realizó el inicio de sesión:

- **2**: Interactivo (consola física).
- **3**: Red (ej. SMB, acceso remoto).
- **7**: Desbloqueo de pantalla.
- **10**: Remoto (RDP).
- **Sospechosos**: Tipo 10 desde IPs externas o tipo 3 repetitivo sin contexto legítimo.

---

## Estrategias de Detección

### En SIEM
- **Alertas**: Configura reglas para:
  - Alta frecuencia de 4625 o 4771.
  - Combinaciones 4624 + 4648 desde IPs sospechosas.
- **Dashboards**: Visualiza 4688 y 4697, filtrando por rutas inusuales.

### Correlaciones Clave
- **4624 + 4672**: Escalada de privilegios tras logon.
- **4720 + 4728/4732**: Nueva cuenta con privilegios elevados.
- **4625 → 4624**: Brute force exitoso.

### Fuentes Prioritarias
- **Security Log**: Autenticaciones y gestión de cuentas.
- **System Log**: Servicios sospechosos (4697, 7045).
- **Applications and Services**: Detalles granulares (ej. Sysmon).

---

## Recursos Adicionales

- **Documentación Microsoft**: Detalles de Event IDs.
- **Event Viewer**: Herramienta nativa para revisar logs (`eventvwr`).
- **BSidesCharm (Sean Metcalf)**: Lista extendida de Event IDs.

---
