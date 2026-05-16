# Wazuh AI Security Analyzer

> Un workflow de n8n que convierte cada alerta de alta severidad de Wazuh en un mensaje de Slack con triaje por IA, incluyendo evaluación de riesgo, causa probable y los comandos exactos de shell para investigar. Aproximadamente **$0.001 por alerta** con Claude Haiku.

[![Ver la demo](https://img.shields.io/badge/YouTube-Walkthrough-red?logo=youtube)](https://youtu.be/74TzhvvWmfk) [![Desarrollado por Agenius AI Labs](https://img.shields.io/badge/Desarrollado%20por-Agenius%20AI%20Labs-0033ff)](https://ageniuslabs.com) [![Licencia: MIT](https://img.shields.io/badge/Licencia-MIT-yellow.svg)](LICENSE)

![Alerta de Slack analizada por IA](screenshots/05-slack-output-live.png)

*Salida real de la demo en vivo: un intento de fuerza bruta SSH contra un nodo Proxmox interno fue correctamente clasificado como `MEDIO, probablemente falso positivo` porque el origen era interno, con cinco comandos de investigación adjuntos.*

---

## Por qué existe esto

Tu SIEM Wazuh genera cientos de alertas al día. La mayoría son ruido. Algunas son reales. Leerlas todas es imposible. Ignorarlas es peligroso. Silenciarlas anula el propósito de tener un SIEM.

Este workflow coloca un analista de IA frente a cada alerta de alta severidad. Webhook de entrada, evaluación de riesgo con contexto de infraestructura en la salida. Publícalo en tu canal de seguridad de Slack y lees la respuesta directamente, sin tener que revisar los dashboards.

## Qué hace

1. Wazuh lanza un webhook con cualquier alerta por encima del nivel configurado (nivel 10 por defecto).
2. El workflow extrae el payload de la alerta y lo envía al LLM elegido (Claude Haiku por defecto; también funciona con OpenAI, Ollama local o cualquier otro) junto con un **bloque de contexto de infraestructura** que le indica al modelo qué es normal en TU red.
3. El modelo devuelve un análisis estructurado: nivel de riesgo, resumen en una línea, causa probable, acciones a tomar y comandos exactos de shell para investigar.
4. El mensaje formateado llega a Slack (o Telegram, Teams, cualquier destino con webhook entrante) en segundos tras la alerta original.

## Inicio rápido

> **¿Configurando con un asistente de IA?** Pega [`AI-SETUP-PROMPT.md`](AI-SETUP-PROMPT.md) en Claude / ChatGPT / Gemini y te guiará a través del despliegue, incluyendo la construcción del bloque de contexto de infraestructura a partir de tu entorno específico. Recomendado.

1. **Importa el workflow** en tu instancia de n8n.
   ```bash
   # En n8n: Workflows > Importar desde archivo > selecciona wazuh-ai-security-analyzer.workflow.json
   ```

2. **Configura tres credenciales** en n8n:
   - Clave API del LLM (Anthropic, OpenAI o tu endpoint de Ollama)
   - URL del webhook entrante de Slack
   - Integración de webhook de Wazuh (configurada a continuación)

3. **Personaliza el bloque de contexto de infraestructura** en el nodo `Config`. Usa [`infrastructure-context-template.md`](infrastructure-context-template.md) como punto de partida. Este es el elemento que hace útil el análisis de IA, no lo omitas.

4. **Configura Wazuh** para lanzar un webhook en alertas a partir del nivel por defecto del workflow (9). Añade a `/var/ossec/etc/ossec.conf`:
   ```xml
   <integration>
     <name>custom-n8n-webhook</name>
     <hook_url>https://TU-INSTANCIA-N8N/webhook/wazuh-alert-ai-public</hook_url>
     <level>9</level>
     <alert_format>json</alert_format>
   </integration>
   ```
   Reinicia el manager de Wazuh: `systemctl restart wazuh-manager`.

   El nodo `Config` del workflow establece `minAlertLevel=9` y `criticalLevel=12` por defecto. Ajusta ambos valores según tu tolerancia al ruido.

5. **Prueba la cadena de extremo a extremo** con el script incluido:
   ```bash
   pip install paramiko
   python scripts/wazuh-bruteforce-test.py <ip-de-tu-objetivo-de-prueba>
   ```
   En unos 30 segundos deberías ver el mensaje analizado por IA en Slack.

## Contenido del workflow

![Vista general del workflow](screenshots/01-workflow-overview.png)

Cuatro secciones lógicas, todas documentadas inline en el JSON del workflow:

| Sección | Nodos | Qué hace |
|---|---|---|
| **Webhook de alerta entrante de Wazuh** | Trigger webhook | Escucha payloads JSON de Wazuh en `/webhook/wazuh-alert-ai-public` |
| **Configuración y cualificación** | Config + nodo IF | Define umbrales de alerta, elección de modelo y zona horaria. Filtra todo lo que esté por debajo de `minAlertLevel` (9 por defecto). |
| **Extracción y análisis** | Extract + llamada LLM | Extrae los detalles de la alerta, construye el prompt con contexto de infraestructura y llama al LLM. |
| **Formateo y notificación** | Format + Slack/Telegram/Teams | Construye el mensaje legible y lo publica. |

### Nodo Config

![Nodo Config](screenshots/02-config-node.png)

Fuente única de verdad para umbrales, proveedor LLM, modelo y zona horaria. Cambia una vez, se aplica en todo el workflow.

### Extracción y análisis

![Nodo de extracción y análisis](screenshots/03-extract-and-analyze-node.png)

El bloque de contexto de infraestructura vive aquí. El prompt es la clave. Consulta [`infrastructure-context-template.md`](infrastructure-context-template.md).

### Salida en Slack

![Nodo de salida en Slack](screenshots/04-slack-output-node.png)

Compatible directamente con cualquier canal de webhook entrante. Las variantes para Telegram y Teams están documentadas inline en las notas adhesivas del nodo.

## El workflow completo con documentación inline

![Workflow completo con notas adhesivas](screenshots/06-workflow-with-inline-docs.png)

Cada nodo tiene una nota adhesiva que explica qué hace, qué configurar y dónde encontrar la documentación relevante. La configuración está pensada para ser autoservicio.

![Panel de detalles de configuración del nodo](screenshots/07-node-configuration-details.png)

## Demo en vivo

![Vista panorámica: workflow, documentación y salida de Slack lado a lado](screenshots/08-birdseye-with-slack.png)

Walkthrough completo con un ataque de fuerza bruta real activando la cadena de extremo a extremo:

▶ **[https://youtu.be/74TzhvvWmfk](https://youtu.be/74TzhvvWmfk)**

## Contenido del repositorio

```
.
├── wazuh-ai-security-analyzer.workflow.json   # El workflow de n8n, listo para importar
├── infrastructure-context-template.md         # Bloque de prompt del sistema (personaliza esto)
├── AI-SETUP-PROMPT.md                         # Prompt complementario para despliegue asistido por IA
├── scripts/
│   └── wazuh-bruteforce-test.py               # Prueba de extremo a extremo del pipeline
├── screenshots/                               # Referencia del workflow y salida de Slack
├── README.md
└── LICENSE                                    # MIT
```

## Quién lo desarrolló

[Michael Frostbutter](https://ageniuslabs.com), fundador de Agenius AI Labs. Más de 25 años en ingeniería de redes y operaciones tecnológicas. Lo construyó para su propio laboratorio doméstico antes de empaquetarlo para que cualquiera pueda usarlo.

## Contribuir

Issues y PRs son bienvenidos. Especialmente interesado en:
- Variantes de notificación para Telegram / Teams / Discord probadas en producción
- Patrones de enrutamiento de bloques de contexto multi-tenant para MSPs
- Recomendaciones de modelos Ollama locales y comparativas de calidad
- Packs de reglas de Wazuh que funcionen bien con este workflow

## Licencia

MIT, consulta [LICENSE](LICENSE). Úsalo como quieras.
