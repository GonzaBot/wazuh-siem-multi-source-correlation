# Wazuh SIEM Multi-Source Event Correlation

Laboratorio práctico de configuración de **Wazuh como SIEM**, orientado a recopilar y correlacionar eventos de seguridad provenientes de múltiples fuentes de datos sobre un mismo endpoint — un servidor Debian 12 con un sitio WordPress — en lugar de utilizar Wazuh únicamente como EDR sobre un host aislado.

## Descripción general

Como SIEM, Wazuh está pensado para ingerir y correlacionar información desde fuentes diversas: agentes en los endpoints, logs de aplicaciones, firewalls, routers, entre otros. Este laboratorio se centra en dos de esas fuentes, ambas sobre el mismo endpoint Debian:

- **Fuente 1 — Agente Wazuh (endpoint):** eventos de autenticación (PAM), uso de privilegios (sudo) y monitoreo de integridad de archivos (syscheck).
- **Fuente 2 — Logs de aplicaciones:** logs de acceso y error de Apache, y logs de depuración de WordPress.

Sobre esa base se simularon tres escenarios de ataque para validar que Wazuh detecta y correlaciona correctamente la actividad proveniente de estas distintas fuentes, y se generó un reporte final desde el dashboard de Wazuh.

## Entorno de trabajo

| Rol | Sistema operativo | Dirección IP | Detalle |
|---|---|---|---|
| Wazuh Manager / Dashboard | Wazuh OVA v4.14.6 | 192.168.1.8 | Instalación all-in-one (manager, indexer y dashboard) |
| Endpoint monitoreado | Debian GNU/Linux 12 | 192.168.1.10 | Apache + WordPress 6.9.4 |

## Qué se hizo

1. **Despliegue del agente** — Instalación y registro del agente Wazuh en el endpoint Debian/WordPress mediante el asistente *Deploy new agent* del propio dashboard. Agente registrado como `Debian` (ID `003`).
2. **Configuración de fuentes de datos**
   - Verificación de que `ossec.conf` ya incluyera los bloques `<localfile>` necesarios para recolectar `access.log` y `error.log` de Apache.
   - Activación del registro de depuración de WordPress (`WP_DEBUG`, `WP_DEBUG_LOG`, `WP_DEBUG_DISPLAY`) en `wp-config.php`, para capturar advertencias y errores de PHP, ya que WordPress no genera logs propios por defecto.
   - Reinicio del agente para aplicar la configuración y reiniciar la recolección de logs desde cero.
3. **Simulación de ataques multi-fuente**
   - **Autenticación fallida:** 11 intentos consecutivos de login fallido con `su` (mapea a la técnica T1078 de MITRE ATT&CK — uso indebido de credenciales).
   - **Modificación de archivo sensible:** se agregó una línea a `/etc/passwd` para activar el monitoreo de integridad de archivos (syscheck).
   - **Tráfico web anómalo:** navegación normal por WordPress más solicitudes repetidas a rutas inexistentes, generando respuestas HTTP 404.
4. **Análisis en Threat Hunting** — Revisión de los eventos correlacionados en el dashboard de Wazuh: contadores de autenticación fallida/exitosa, grupos de reglas PAM/sudo/su, y anomalías de tráfico web (grupos `accesslog`, `attack` y `web` asociados a los 404).
5. **Generación de reporte** — Generación de un reporte PDF (*Threat Hunting Report*) desde el dashboard de Wazuh, resumiendo todas las alertas del agente durante la ventana de prueba (81 alertas, 22 fallos y 18 autenticaciones exitosas).

## Principales hallazgos

- Dos de los tres escenarios simulados quedaron claramente confirmados y correlacionados en el dashboard: los intentos de login fallidos (grupos `authentication_failed`, `pam`, `su`) y el tráfico anómalo de Apache (grupos `accesslog`, `attack`, `web` sobre las respuestas 404).
- La modificación de `/etc/passwd` se confirmó a nivel de sistema de archivos y generó eventos indirectos relacionados (uso de sudo, verificaciones del benchmark CIS sobre `/etc/passwd`), pero no se visualizó una alerta explícita de syscheck del tipo "integrity checksum changed" dentro de la ventana de trabajo, probablemente debido a la frecuencia de escaneo programada de dicho módulo (12 horas por defecto) en lugar de un monitoreo en tiempo real.
- Comprender la frecuencia de escaneo de cada módulo de Wazuh (en especial syscheck) es relevante a nivel operativo: puede existir una demora real entre un cambio en el sistema de archivos y la alerta correspondiente.

## Contenido del repositorio

- `wazuh-siem-multi-source-report.pdf` — informe formal completo del laboratorio (en español), con metodología, evidencia y conclusiones.
- `wazuh-threat-hunting-report.pdf` — reporte PDF oficial generado directamente desde el dashboard de Wazuh.
- `/screenshots` — evidencia recolectada a lo largo del laboratorio (despliegue del agente, archivos de configuración, simulaciones de ataque, vistas del dashboard).

## Herramientas utilizadas

`Wazuh 4.14.6` · `Debian 12` · `Apache2` · `WordPress 6.9.4` · `VirtualBox`

## Autor

**Gonzalo** — [GonzaBot](https://github.com/GonzaBot)
Especialización en Blue Team / SOC — 4Geeks Academy
