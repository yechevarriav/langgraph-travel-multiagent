# 🌍✈️ Planificador de Viajes Multi-Agente con LangGraph

Sistema inteligente de planificación de viajes que utiliza **múltiples agentes especializados** trabajando en conjunto para crear itinerarios personalizados mediante coordinación automática.

**Autor**: Yvonne Echevarria  
**Basado en**: [LangGraph Multi-Agent Tutorial](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/)  
**Fecha**: Octubre 2025

---

## 📋 Tabla de Contenidos

1. [Descripción General](#-descripción-general)
2. [Arquitectura del Sistema](#️-arquitectura-del-sistema)
3. [Agentes Especializados](#-agentes-especializados)
4. [Tecnologías y Librerías](#️-tecnologías-y-librerías)
5. [Instalación](#-instalación)
6. [Uso](#-uso)
7. [Conceptos Avanzados](#-conceptos-avanzados)
8. [Estructura del Proyecto](#-estructura-del-proyecto)
9. [Troubleshooting](#-troubleshooting)

---

## 🎯 Descripción General

Este proyecto implementa un **sistema multi-agente basado en LangGraph** que automatiza la planificación de viajes mediante la coordinación inteligente de **4 agentes especializados**:

1. **👤 Entrevistador**: Recopila información del usuario (destino, duración, preferencias, presupuesto)
2. **🔍 Supervisor**: Determina cuándo hay suficiente información para proceder  
3. **🗺️ Investigador**: Busca información detallada usando **Wikipedia + API de Clima**
4. **📋 Generador**: Crea un itinerario personalizado día por día

El sistema utiliza **routing condicional**, **estado compartido** y **checkpointing** para coordinar estos agentes de manera eficiente.

### Características Principales

✨ **Conversación Natural**: Interfaz amigable con el usuario  
🤖 **Google Gemini 2.0**: LLM de última generación gratuito  
📚 **Wikipedia API**: Información enciclopédica de destinos  
🌤️ **Open-Meteo API**: Clima en tiempo real sin API key  
🔄 **Sistema Robusto**: Mecanismos de respaldo para decisiones  
💾 **Memoria Persistente**: Checkpointing con sesiones únicas  

---

## 🛠️ Tecnologías y Librerías

### Librerías Principales

| Librería | Versión | Propósito |
|----------|---------|-----------|
| **langgraph** | Latest | Framework de grafos multi-agente |
| **langchain** | Latest | Orquestación de LLMs |
| **langchain-google-genai** | Latest | Integración con Google Gemini |
| **wikipedia** | Latest | Búsqueda de información de destinos |
| **requests** | Latest | Llamadas HTTP a Open-Meteo API |
| **termcolor** | Latest | Colores en terminal |

### Modelo de Lenguaje

**Google Gemini 2.0 Flash Experimental**

- **Costo**: **GRATUITO** 
- **Límites**: 15 req/min, 1500 req/día, 1M tokens/día
- **Ventajas**: Rápido, sin billing, excelente calidad

```python
llm = ChatGoogleGenerativeAI(
    model="gemini-2.0-flash-exp",
    temperature=0.7,
    convert_system_message_to_human=True
)
```

### APIs Externas

1. **Wikipedia API**: Información enciclopédica (gratis, sin límites)
2. **Open-Meteo API**: Clima en tiempo real (**sin API key**, gratis)

---

## 📦 Instalación

### Paso 1: Obtener API Key de Google Gemini

1. Ve a [Google AI Studio](https://makersuite.google.com/app/apikey)
2. Crea tu API Key (gratis)
3. Cópiala

### Paso 2: Instalar Dependencias

```python
!pip install langgraph langchain langchain-google-genai wikipedia termcolor requests -q
```

### Paso 3: Configurar API Key en Colab

1. Click en 🔑 **Secrets** (menú izquierdo)
2. Crear secret: `GOOGLE_API_KEY`
3. Pegar tu API key
4. Activar "Notebook access"

```python
from google.colab import userdata
import os

GOOGLE_API_KEY = userdata.get('GOOGLE_API_KEY')
os.environ['GOOGLE_API_KEY'] = GOOGLE_API_KEY
```

### Paso 4: Verificar Instalación

```python
from langchain_google_genai import ChatGoogleGenerativeAI

llm = ChatGoogleGenerativeAI(
    model="gemini-2.0-flash-exp",
    temperature=0.7,
    convert_system_message_to_human=True
)

print(llm.invoke("Di 'OK'").content)  # Debe mostrar: OK
```

---

## 🚀 Uso

### Inicio Rápido

1. Ejecutar celdas 1-8 para configurar el sistema
2. Ejecutar celda 9 para iniciar el chat
3. Responder las preguntas del entrevistador
4. Recibir tu itinerario personalizado

### Ejemplo de Conversación

```
🤖 Entrevistador:
¡Hola! 👋 ¿A dónde te gustaría viajar? 🌍✈️

👤 Tú: Me gustaría ir a Tokio

🤖 Entrevistador:
¡Tokio! 🗼 ¿Cuántos días tienes disponibles?

👤 Tú: 5 días

🤖 Entrevistador:
Perfecto. ¿Qué actividades te interesan?
(cultura, templos, comida, tecnología...)

👤 Tú: Cultura japonesa, templos y comida local

🤖 Entrevistador:
¡Suena increíble! 🍜⛩️ ¿Cuál es tu presupuesto?
(bajo/medio/alto)

👤 Tú: Presupuesto medio

🔍 Supervisor:
✅ Información completa → Investigando destino...

🗺️ Investigador:
📚 Wikipedia: Información de Tokio recopilada
🌤️ Clima: 18°C - Parcialmente nublado ⛅

📋 Generador:
✨ Creando itinerario personalizado...

═══════════════════════════════════════
🎉 ¡TU ITINERARIO ESTÁ LISTO!
═══════════════════════════════════════

🗼 VIAJE A TOKIO - 5 DÍAS 🇯🇵

📍 Clima actual: 18°C ⛅  
💡 Recomendación: Lleva chaqueta ligera

DÍA 1: Llegada y Exploración Inicial 🌆

🌅 Mañana (9:00-12:00):
   • Llegada a Aeropuerto de Narita
   • Check-in en Shibuya
   • Desayuno en Ichiran Ramen

🌆 Tarde (12:00-18:00):
   • Templo Senso-ji en Asakusa
   • Nakamise Street
   • Cruce de Shibuya

🌙 Noche (18:00-22:00):
   • Cena de sushi local
   • Tokyo Skytree
   • Karaoke en Shinjuku

💡 Tips:
   - Compra Suica Card (transporte)
   - Lleva efectivo
   - Google Maps offline

[... DÍA 2, 3, 4, 5 ...]

💰 PRESUPUESTO ESTIMADO: ~$925 USD

¡Buen viaje a Tokio! 🎌✈️
```

---

## 📁 Estructura del Proyecto

```
Challenge_YvonneEchevarria.ipynb
│
├── CELDA 1: Instalación
├── CELDA 2: API Keys
├── CELDA 3: LLM + Tools (Wikipedia + Clima)
├── CELDA 4: Estado Compartido (TravelState)
├── CELDA 5: Prompts Especializados
├── CELDA 6: Nodos (4 agentes)
├── CELDA 7: Router Condicional
├── CELDA 8: Construcción del Grafo
└── CELDA 9: Playground Interactivo
```

---

## 🔧 Troubleshooting

### Error: Rate Limit Exceeded

```
429 - You exceeded your current quota
```

**Solución**: Espera hasta medianoche PST (reset diario)

### Error: Recursion Limit

```
GraphRecursionError: Recursion limit reached
```

**Solución**: Aumentar recursion_limit en config:

```python
config = {
    "configurable": {"thread_id": thread_id},
    "recursion_limit": 50  # default: 25
}
```

### API Key Inválida

```
400 - API key not valid
```

**Solución**:
1. Verificar API key en Google AI Studio
2. Regenerar si es necesario
3. Actualizar en secretos de Colab

---

## 📚 Recursos

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Google Gemini API](https://ai.google.dev/docs)
- [Wikipedia API](https://wikipedia.readthedocs.io/)
- [Open-Meteo API](https://open-meteo.com/en/docs)

---

## 👨‍💻 Autor

**Yvonne Echevarria**

Proyecto de aprendizaje de sistemas multi-agente con LangGraph.

---

## 🎉 Estado

**✅ COMPLETADO Y FUNCIONAL**

- ✅ 4 agentes especializados
- ✅ Google Gemini 2.0 Flash (gratis)
- ✅ Wikipedia + Open-Meteo APIs
- ✅ Routing condicional
- ✅ Itinerarios personalizados

**Versión**: 1.0  
**Fecha**: Octubre 2025  
**Status**: Producción estable

---

**¡Happy Coding!** 🚀✈️🌍
