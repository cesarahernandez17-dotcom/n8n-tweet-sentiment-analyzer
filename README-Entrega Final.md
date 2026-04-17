# 🚀 n8n Tweet Sentiment Analyzer - Automatización LLM

**Monitorea tweets → Analiza sentimiento Groq → Alertas Gmail automáticas**

![Demo](demo.gif) <!-- Agrega screenshot n8n -->

## 🎯 **Qué hace (cada 1min):**

## Análisis Automatizado de Tweets con Groq + Gmail

---

## Descripción general

Workflow automatizado en n8n que monitorea una hoja de Google Sheets con tweets,
clasifica su sentimiento usando LLMs (Groq/Llama), y actúa según el resultado:
- **Tweet NEGATIVO** → analiza el problema y envía alerta por email
- **Tweet POSITIVO/NEUTRAL** → genera un borrador de respuesta y lo envía por email
- Todos los tweets procesados quedan registrados en una hoja de Log

---

## Estructura del entregable

```
proyecto_n8n/
├── Entregable.json      → Workflow n8n listo para importar
├── prompts.txt          → Prompts de los 3 nodos LLM
├── documentacion.pdf    → Descripción técnica, diagramas y guía completa
└── README.md            → Este archivo
```

---

## Requisitos previos

- n8n v1.0+ (self-hosted o cloud)
- Cuenta de Groq con API key gratuita → [console.groq.com](https://console.groq.com)
- Cuenta de Google con acceso a Google Sheets y Gmail
- Google Cloud Project con Google Sheets API y Google Drive API habilitadas

---

## Estructura del Google Sheet

### Hoja 1 (tweets de entrada)
La fila 1 debe tener exactamente estos encabezados:

| tweet | author | date |
|-------|--------|------|
| Texto del tweet... | @usuario | 2024-01-15 |

### Hoja Log (generada automáticamente)
La fila 1 debe tener estos encabezados:

| fecha_procesado | autor | texto | sentimiento | score | accion |
|----------------|-------|-------|-------------|-------|--------|

---

## Pasos para importar y ejecutar

### 1. Importar el workflow en n8n
1. Abrir n8n → menú superior derecho **⋯** → **Import from file**
2. Seleccionar `Entregable.json`
3. El workflow aparece con todos los nodos conectados

### 2. Configurar credencial de Google Sheets
1. n8n → **Settings** → **Credentials** → **New** → **Google Sheets OAuth2**
2. En [Google Cloud Console](https://console.cloud.google.com):
   - Crear proyecto → habilitar **Google Sheets API** y **Google Drive API**
   - Crear credencial OAuth2 → tipo **Web application**
   - Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`
   - Copiar **Client ID** y **Client Secret**
3. Pegar en n8n → **Connect my account** → autorizar con Google

### 3. Configurar credencial de Gmail
1. n8n → **Settings** → **Credentials** → **New** → **Gmail OAuth2**
2. Usar el mismo proyecto de Google Cloud
3. Conectar la cuenta de Gmail desde donde se enviarán los emails

### 4. Configurar credencial de Groq
1. Ir a [console.groq.com](https://console.groq.com) → Sign Up → **API Keys** → **Create API Key**
2. n8n → **Settings** → **Credentials** → **New** → **Groq**
3. Pegar la API key → guardar

### 5. Asignar credenciales en los nodos
Una vez creadas las credenciales, asignarlas en los nodos correspondientes:
- **Leer Tweets** y **Leer Log Procesados** y **Log en Google Sheets** → Google Sheets OAuth2
- **Groq - Clasificador**, **Groq - Analizador**, **Groq - Respuesta Positiva** → Groq
- **Enviar Email de Alerta** y **Enviar Email Borrador Positivo** → Gmail OAuth2

### 6. Configurar email destinatario
En los nodos **Enviar Email de Alerta** y **Enviar Email Borrador Positivo**,
cambiar el campo `sendTo` por el email donde se quieren recibir las alertas.

### 7. Publicar el workflow
Click en **Publish** (arriba a la derecha) → el workflow queda activo y
comienza a ejecutarse automáticamente cada minuto.

---

## Cómo funciona el sistema

```
Cada 1 minuto
      ↓
[Schedule Trigger]
      ↓
[Leer Tweets] + [Leer Log Procesados]
      ↓
[Filtrar No Procesados] → descarta tweets ya procesados
      ↓
[Preparar Datos] → normaliza campos
      ↓
[LLM 1 - Clasificador] → devuelve {"sentiment":"NEGATIVO","score":0.95}
      ↓
[Parsear Respuesta LLM]
      ↓
[¿Es NEGATIVO?]
    ↙           ↘
[NEGATIVO]    [POSITIVO/NEUTRAL]
    ↓               ↓
[LLM 2]         [LLM 3]
[Analiza]       [Genera borrador]
    ↓               ↓
[Email rojo]   [Email verde]
    ↓               ↓
      [Log en Google Sheets]
```

---

## Tweets de prueba reproducibles

Agregar estas filas en **Hoja 1** para verificar el comportamiento:

**Tweet 1 — NEGATIVO (urgencia ALTA):**
```
Llevan 5 días sin responderme y encima me cobraron dos veces. Voy a hacer la denuncia.	@usuario_test1	2024-01-15
```

**Tweet 2 — POSITIVO:**
```
Recibí mi pedido antes de lo esperado y el packaging es hermoso. 100% recomendados!	@usuario_test2	2024-01-15
```

**Tweet 3 — NEGATIVO (urgencia MEDIA):**
```
La aplicación sigue tirando error al intentar iniciar sesión desde ayer.	@usuario_test3	2024-01-15
```

**Tweet 4 — NEUTRAL:**
```
Hice un pedido ayer, alguien sabe cuánto tarda normalmente el envío a Buenos Aires?	@consulta_envio	2024-01-15
```

Podés agregar varios tweets a la vez, el sistema los procesa todos en la siguiente ejecución automática.

---

## Manejo de errores

| Situación | Comportamiento |
|-----------|---------------|
| LLM devuelve JSON malformado | Nodo Code hace fallback con keywords |
| Valor de sentiment inesperado | Se asigna NEUTRAL por defecto |
| Tweet ya procesado | El filtro lo descarta automáticamente |
| Log vacío (primera ejecución) | Se procesan todos los tweets del Sheet |
| Error de API Groq (rate limit) | n8n registra el error en Executions |

---

## Notas de seguridad

- No subir credenciales reales en los archivos entregados
- Usar siempre las credenciales cifradas de n8n
- Las API keys de ejemplo están marcadas como `YOUR_*_ID`
- Para producción, configurar `N8N_ENCRYPTION_KEY` como variable de entorno

---

## Limitaciones conocidas

- El modelo Groq puede no detectar correctamente ironía o sarcasmo en español
- El plan gratuito de Groq tiene límite de requests por minuto
- El trigger se ejecuta cada 1 minuto, no en tiempo real
- Tweets muy similares pueden confundir al filtro anti-duplicados

---

## Equipo

Proyecto Final — Automatización con n8n y LLMs
