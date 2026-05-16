# Prompt de Configuración con IA

> Pega el bloque de prompt que aparece abajo en Claude, ChatGPT, Gemini o cualquier LLM con capacidad de razonamiento. Te guiará a través del despliegue de este workflow en TU infraestructura: configuración de Wazuh, importación en n8n, credenciales, personalización del contexto de infraestructura y prueba de extremo a extremo. El modelo te hará las preguntas que necesita responder; tú solo responde con cómo es realmente tu entorno.

**Por qué usarlo:** el workflow en sí se importa y funciona de inmediato, pero el *bloque de contexto de infraestructura* es donde reside la mayor parte del valor. El prompt que aparece a continuación lo extrae de ti con preguntas específicas en lugar de dejarte mirando una plantilla en blanco.

**Modelos recomendados:** Claude Sonnet 4.6 / Opus 4.7, GPT-5 o Gemini 2.5 Pro. Los modelos más pequeños también funcionan, pero generan bloques de contexto menos detallados.

---

## Copia todo lo que hay a continuación y pégalo en tu IA

```
Me estás ayudando a desplegar el workflow Wazuh AI Security Analyzer de
https://github.com/Agenius-AI-Labs/Wazuh-AI-Security-Analyzer
en mi propio entorno. El workflow canaliza las alertas de alta severidad del SIEM Wazuh
a través de un LLM (Claude Haiku por defecto) y publica evaluaciones de riesgo
con triaje por IA en Slack.

## Tu función

Guíame a través del despliegue en orden. Hazme una pregunta concreta a la vez
o un grupo reducido de preguntas relacionadas. Espera mi respuesta antes de continuar.
No generes el plan completo por adelantado. No des sermones. No rellenes las respuestas
con frases de ánimo.

## Orden de operaciones (síguelo en secuencia)

1. **Auditoría del entorno.** Confirma que tengo:
   - Un manager de Wazuh en ejecución (anota la versión)
   - n8n autoalojado o n8n cloud (indica cuál y el hostname público)
   - Al menos una cuenta de proveedor LLM (Anthropic / OpenAI / Ollama)
   - Un workspace de Slack donde pueda crear webhooks entrantes (o Telegram /
     Teams si no uso Slack)
   Si falta alguno, dame la ruta de instalación más corta viable para
   ese componente y espera hasta que confirme que está operativo.

2. **Importación del workflow.** Dime la ruta exacta de menú en n8n para importar
   `wazuh-ai-security-analyzer.workflow.json`. Una vez importado, lista las
   credenciales que necesito crear en el panel de Credenciales de n8n (clave API del LLM,
   webhook de Slack/Telegram/Teams). Espera a que confirme que cada una está creada.

3. **Webhook entrante de Slack (o alternativa).** Guíame a través de la creación de un
   webhook entrante en Slack en https://api.slack.com/apps para el canal donde
   quiero recibir las alertas. Recuérdame que la URL es un SECRETO y que nunca debo
   pegarla en el chat ni en los commits. Si uso Telegram o Teams, dame la ruta
   equivalente para esa plataforma.

4. **Integración con Wazuh.** Genera el bloque `<integration>` exacto que debo
   añadir a `/var/ossec/etc/ossec.conf`, usando mi hostname público de n8n.
   El `<level>` por defecto es 9. Dime el comando systemctl para reiniciar el
   manager de Wazuh y cómo hacer tail de `/var/ossec/logs/integrations.log` para
   confirmar que la integración se ha cargado.

5. **Bloque de contexto de infraestructura.** Este es el paso más importante.
   Usa la estructura de plantilla de
   https://github.com/Agenius-AI-Labs/Wazuh-AI-Security-Analyzer/blob/main/infrastructure-context-template.md
   Luego ENTREVÍSTAME sobre:
   - Postura de red (¿zero-trust? ¿solo VPN? ¿puntos de ingreso públicos?)
   - IPs / hostnames de origen de confianza (jump hosts, estaciones de trabajo de administración,
     runners de automatización)
   - Activos críticos que deben escalar incluso si el origen parece legítimo
     (el propio SIEM, bóveda de secretos, bases de datos de producción, cualquier sistema con datos PII)
   - Comportamientos automáticos esperados que NO deben generar alertas (cron jobs,
     ejecuciones de ansible, rotaciones de claves programadas)
   - Anulaciones de riesgo (qué debe ser siempre CRÍTICO independientemente del origen)
   Genera un bloque de contexto terminado a partir de mis respuestas e indícame exactamente
   en qué nodo y campo de n8n debo pegarlo.

6. **Prueba de extremo a extremo.** Guíame para ejecutar el script incluido
   `scripts/wazuh-bruteforce-test.py` contra un objetivo de prueba que yo autorice.
   Dime qué esperar en cada etapa:
   - Fallos SSH registrados por paramiko
   - El manager de Wazuh generando la alerta de la regla 5712 (~30s)
   - Webhook de n8n disparado
   - Respuesta del LLM en Slack con el análisis estructurado
   Si algún paso falla, hazme preguntas de diagnóstico específicas para localizar
   el problema (¿log de Wazuh? ¿log de ejecución de n8n? ¿permisos del canal de Slack?).

7. **Ajuste fino.** Una vez que una alerta de prueba llegue correctamente, pregúntame:
   - ¿Fue precisa la evaluación de riesgo de la IA?
   - ¿Fueron útiles los comandos de investigación sugeridos?
   - ¿Se disparó la alerta con demasiada frecuencia o se filtró demasiado agresivamente?
   Recomienda ediciones específicas al bloque de contexto, `minAlertLevel`,
   `criticalLevel` o elección de modelo basándote en mis respuestas.

## Reglas para ti

- Sé conciso. Omite el preámbulo. No digas cosas como "¡Buena pregunta!" o
  "Esto es lo que haremos." Hazlo directamente.
- Un paso a la vez. No te adelantes.
- Cuando pegue una salida (logs, mensajes de error, capturas de pantalla), analízala y
  dime la siguiente acción concreta.
- Nunca me pidas que pegue una credencial, clave API, URL de webhook o token de sesión
  en el chat. Si necesito usar uno, dime que lo guarde en el almacén de credenciales de n8n
  y haz referencia a él solo por el nombre de la credencial.
- Si un paso es específico de la plataforma (Wazuh en Ubuntu vs RHEL, n8n en Docker
  vs bare-metal vs cloud), pregúntame cuál uso antes de generar los comandos.
- Si pregunto sobre costes: el modelo por defecto es Claude Haiku. Los precios cambian;
  dime que consulte los precios actuales en https://www.anthropic.com/pricing en lugar
  de citarlos directamente.

Empieza ahora. Primera pregunta: ¿cuál es la versión de mi manager de Wazuh y dónde
está ejecutándose mi instancia de n8n (Docker autoalojado, n8n cloud, bare-metal)?
```

---

## Qué obtendrás

Un despliegue funcional con un bloque de contexto que se adapta realmente a tu entorno, en lugar de una plantilla genérica que marca cada ejecución interna de `cron` como una amenaza potencial.

El formato de entrevista detecta cosas que de otro modo pasarías por alto: ese runner de Ansible nocturno que olvidaste, los trabajos de migración de Proxmox que parecen intentos de fuerza bruta, los patrones de consulta DNS de Pi-hole que Wazuh a veces clasifica incorrectamente. El modelo los detecta preguntando, no adivinando.

## Si tu IA se queda atascada

La mayoría de los fallos ocurren en el paso 4 (integración con Wazuh). Confirma que `/var/ossec/logs/integrations.log` existe y muestra que tu integración se ha cargado. Si no es así, la causa más común es que el umbral `<level>` está por encima de cualquier alerta activa; bájalo temporalmente a `5` para las pruebas y luego vuelve a subirlo a `9` una vez que la cadena funcione.

Para problemas con n8n, pega el JSON de salida de la ejecución fallida a tu IA y deja que analice el error real. La mayoría de los problemas de n8n son errores de credenciales, no errores en la lógica del workflow.
