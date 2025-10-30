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
5. [Técnicas de LangGraph](#-técnicas-de-langgraph-implementadas)
6. [Estado Compartido](#-estado-compartido)
7. [Flujo de Ejecución](#-flujo-de-ejecución)
8. [Instalación](#-instalación)
9. [Uso](#-uso)
10. [Conceptos Avanzados](#-conceptos-avanzados)
11. [Estructura del Proyecto](#-estructura-del-proyecto)


---

## 🎯 Descripción General

Este proyecto implementa un **sistema multi-agente basado en LangGraph** que automatiza la planificación de viajes mediante la coordinación inteligente de **4 agentes especializados**:

1. **👤 Entrevistador**: Recopila información del usuario (destino, duración, preferencias, presupuesto)
2. **🔍 Supervisor**: Determina cuándo hay suficiente información para proceder
3. **🗺️ Investigador**: Busca información detallada del destino en Wikipedia
4. **📋 Generador**: Crea un itinerario personalizado día por día

El sistema utiliza **routing condicional**, **estado compartido** y **checkpointing** para coordinar estos agentes de manera eficiente.

---

## 🏗️ Arquitectura del Sistema

### Diagrama del Grafo de Estados
```
        ┌─────────┐
        │  START  │
        └────┬────┘
             ↓
      ┌──────────────┐
      │ Entrevistador│ ←──────┐
      │   (Nodo 1)   │        │
      └──────┬───────┘        │
             ↓                │
      ┌──────────────┐        │
      │  Supervisor  │        │
      │   (Nodo 2)   │        │
      └──────┬───────┘        │
             ↓                │
      [ROUTER CONDICIONAL]    │
             ↓                │
       ¿CONTINUE? ────────────┘
             │
       ¿INVESTIGATE?
             ↓
      ┌──────────────┐
      │  Wikipedia   │
      │   (Nodo 3)   │
      └──────┬───────┘
             ↓
      ┌──────────────┐
      │  Generador   │
      │   (Nodo 4)   │
      └──────┬───────┘
             ↓
        ┌─────────┐
        │   END   │
        └─────────┘
```

### Componentes del Grafo

| Componente | Tipo | Función |
|-----------|------|---------|
| **START** | Punto de entrada | Inicializa el sistema |
| **Entrevistador** | Nodo asíncrono | Recopila información mediante conversación |
| **Supervisor** | Nodo determinístico | Evalúa completitud de información |
| **Router** | Condicional | Decide CONTINUE o INVESTIGATE |
| **Wikipedia** | Nodo asíncrono | Investiga el destino |
| **Generador** | Nodo asíncrono | Crea itinerario personalizado |
| **END** | Punto de salida | Finaliza el proceso |

### Conexiones del Grafo

- **START → Entrevistador**: Punto de entrada inicial
- **Entrevistador → Supervisor**: Después de cada respuesta del usuario
- **Supervisor → Entrevistador** (si CONTINUE): Loop de recopilación
- **Supervisor → Wikipedia** (si INVESTIGATE): Avance a investigación
- **Wikipedia → Generador**: Secuencial
- **Generador → END**: Finalización

---

## 👥 Agentes Especializados

### 1. 👤 Agente Entrevistador

**Responsabilidad**: Recopilar información del usuario mediante conversación natural

**Información que obtiene**:
- ✈️ **Destino**: Ciudad o país a visitar
- 📅 **Duración**: Número de días del viaje
- 🎯 **Actividades**: Tipo de experiencias (cultura, aventura, playa, gastronomía)
- 💰 **Presupuesto**: Nivel aproximado (bajo, medio, alto)

**Características**:
- Hace UNA pregunta a la vez (no abruma)
- Conversación natural y amigable
- Se adapta a las respuestas del usuario
- Usa emojis para mejor experiencia
- Genera prompt de finalización cuando completa

**Implementación**:
- Usa LLM con prompt engineering específico
- Prompt especializado para entrevistas de viaje
- Genera mensaje inicial sin llamar al LLM (optimización)

---

### 2. 🔍 Agente Supervisor

**Responsabilidad**: Determinar cuándo se ha recopilado suficiente información para avanzar

**Mecanismo de decisión dual**:

1. **Detección primaria** (basada en LLM):
   - Busca frases clave: "ENTREVISTA_COMPLETA:", "tengo todo lo necesario", "listo para planificar"

2. **Sistema de respaldo** (determinístico):
   - Verifica que existan las 4 respuestas clave:
     - ✓ Destino mencionado
     - ✓ Duración especificada
     - ✓ Actividades descritas
     - ✓ Presupuesto indicado
   - Cuenta número de turnos (mínimo 4 preguntas)

**Decisiones posibles**:
- `CONTINUE`: Volver al entrevistador (información incompleta)
- `INVESTIGATE`: Avanzar a investigación (información suficiente)

**Ventaja del diseño**:
- **Robusto**: No depende 100% del LLM
- **Confiable**: Sistema de respaldo previene loops infinitos
- **Transparente**: Muestra en pantalla qué información falta

---

### 3. 🗺️ Agente Investigador (Wikipedia)

**Responsabilidad**: Buscar, procesar y estructurar información del destino

**Proceso en 4 pasos**:

1. **Extracción**: Usa LLM para extraer datos estructurados de la conversación
   - Destino exacto
   - Duración del viaje
   - Tipo de actividades
   - Nivel de presupuesto

2. **Búsqueda**: Consulta Wikipedia API con el destino identificado
   - Maneja desambiguación automática
   - Obtiene resumen de ~10 oraciones

3. **Procesamiento**: LLM filtra y estructura información relevante
   - Atracciones principales
   - Cultura y costumbres
   - Clima típico
   - Tips prácticos

4. **Almacenamiento**: Guarda en el estado compartido

**Técnicas utilizadas**:
- Extracción estructurada con LLM
- Integración con Wikipedia API
- Manejo de errores (destino no encontrado, desambiguación)
- Prompt especializado para filtrado turístico

---

### 4. 📋 Agente Generador de Itinerario

**Responsabilidad**: Crear un itinerario personalizado día por día

**Proceso de generación**:

1. **Análisis completo**:
   - Lee TODA la conversación previa
   - Identifica preferencias implícitas
   - Considera información de Wikipedia

2. **Personalización**:
   - Adapta según presupuesto (bajo = gratis, medio = mix, alto = premium)
   - Respeta tipo de actividades preferidas
   - Distribuye experiencias a lo largo de los días

3. **Estructuración**:
   - Divide cada día en: Mañana / Tarde / Noche
   - Incluye nombres específicos de lugares
   - Agrega horarios sugeridos y tips prácticos
   - Usa emojis para visualización

**Formato de salida**:
```
DÍA 1: [Título temático] 🌆

🌅 Mañana (9:00-12:00):
   • Actividad específica
   • Lugar concreto con nombre
   • Tip práctico

🌆 Tarde (12:00-18:00):
   • [...]

🌙 Noche (18:00-22:00):
   • [...]
```

**Técnica**: Prompt engineering avanzado con contexto completo

---

## 🛠️ Tecnologías y Librerías

### Stack Tecnológico Completo
```
┌─────────────────────────────────────┐
│         Interface (Usuario)         │
│      Google Colab Notebook          │
└────────────┬────────────────────────┘
             │
┌────────────┴────────────────────────┐
│      Orchestration Layer            │
│         LangGraph                   │
│      (Grafo de Estados)             │
└────────────┬────────────────────────┘
             │
┌────────────┴────────────────────────┐
│         LLM Layer                   │
│   OpenAI GPT-4o-mini (via API)      │
│      LangChain-OpenAI               │
└────────────┬────────────────────────┘
             │
┌────────────┴────────────────────────┐
│      External APIs                  │
│      Wikipedia API                  │
└─────────────────────────────────────┘
```

### Librerías Utilizadas

| Librería | Versión | Propósito | Funcionalidad Específica |
|----------|---------|-----------|--------------------------|
| **langgraph** | Latest | Framework de grafos | Construcción del sistema multi-agente |
| **langchain** | Latest | Orquestación | Cadenas de prompts, gestión de mensajes |
| **langchain-openai** | Latest | Integración LLM | Conexión con OpenAI API |
| **openai** | Latest (API) | Modelo de lenguaje | GPT-4o-mini para generación de texto |
| **wikipedia** | Latest | Búsqueda de info | API de Wikipedia en español |
| **termcolor** | Latest | UI en terminal | Output colorizado para mejor UX |
| **python-dotenv** | Latest | Configuración | Variables de entorno seguras |

### Modelo de Lenguaje

**OpenAI GPT-4o-mini**
- **Razón de elección**: Balance óptimo entre costo y calidad
- **Costo**: ~$0.01 USD por sesión completa
- **Ventajas**: 
  - Rápido (latencia baja)
  - Económico
  - Suficientemente capaz para este uso
  - Soporta conversaciones largas

**Alternativas consideradas**:
- ❌ Gemini: Límites de cuota restrictivos
- ✅ GPT-4o: Más potente pero 10x más caro
- ✅ GPT-4o-mini: **Elegido** (mejor costo-beneficio)

---

## 🧩 Técnicas de LangGraph Implementadas

### 1. Estado Compartido con Reducción

**Concepto**: Un diccionario tipado que viaja por todo el grafo y es accesible/modificable por todos los nodos.

**Implementación**:
```
TravelState (TypedDict):
├── messages: Annotated[Sequence[BaseMessage], add_messages]  ← Reducción
├── destination: str
├── duration: str
├── activities: str
├── budget: str
├── supervisor_decision: str
├── interview_complete: bool
├── destination_research: str
└── travel_itinerary: str
```

**Función de reducción `add_messages`**:
- Concatena nuevos mensajes al historial
- NO reemplaza mensajes anteriores
- Mantiene el contexto completo de la conversación

**Ventaja**: Cada agente puede leer/escribir sin comunicación directa entre ellos.

---

### 2. Routing Condicional

**Concepto**: El flujo del grafo toma diferentes caminos según el estado actual.

**Implementación en el proyecto**:
```
Supervisor evalúa → decision = "CONTINUE" o "INVESTIGATE"
                           ↓
            Router decide basado en decision:
                           ↓
            ┌──────────────┴──────────────┐
            ↓                             ↓
      "CONTINUE"                    "INVESTIGATE"
            ↓                             ↓
    Volver a Entrevistador         Ir a Wikipedia
```

**Función del router**:
- Lee `supervisor_decision` del estado
- Retorna un string: "CONTINUE" o "INVESTIGATE"
- LangGraph usa este string para elegir la siguiente arista

**Ventaja**: Permite loops dinámicos sin hardcodear número de iteraciones.

---

### 3. Nodos Asíncronos

**Concepto**: Operaciones no bloqueantes que permiten mejor rendimiento.

**Implementación**:
- Todos los nodos son funciones `async`
- Usan `await` para llamadas al LLM
- Permiten procesamiento concurrente

**Ejemplo de firma**:
```
async def nodo(state: TravelState) -> dict:
    response = await llm.ainvoke(...)
    return {"messages": [response]}
```

**Ventaja**: Sistema más responsivo, especialmente con múltiples usuarios.

---

### 4. Checkpointing y Persistencia

**Concepto**: Guardar el estado del grafo en cada paso para poder:
- Pausar y resumir conversaciones
- Debugging más fácil
- Recuperación ante fallos

**Implementación**:
```
memory = MemorySaver()
workflow.compile(checkpointer=memory)
```

**Configuración de sesión**:
```
thread_id = "travel_session_abc123"
config = {"configurable": {"thread_id": thread_id}}
```

**Ventaja**: Cada sesión mantiene su propio estado independiente.

---

### 5. Streaming de Eventos

**Concepto**: Ejecutar el grafo paso a paso en lugar de todo de una vez.

**Implementación**:
```
async for event in travel_graph.astream(state, config):
    # Procesar cada nodo individualmente
    # Pausar para interacción del usuario
```

**Diferencia vs `ainvoke`**:
- `ainvoke`: Ejecuta TODO el grafo hasta END
- `astream`: Ejecuta nodo por nodo, con control fino

**Ventaja**: Permite interacción usuario en medio del flujo.

---

## 📊 Estado Compartido

### Estructura Completa
```python
class TravelState(TypedDict):
    # ========== CONVERSACIÓN ==========
    messages: Annotated[Sequence[BaseMessage], add_messages]
    # Historial completo: [AI, Human, AI, Human, ...]
    
    # ========== INFORMACIÓN DEL VIAJE ==========
    destination: str       # "Tokio"
    duration: str          # "5 días"
    activities: str        # "Cultura, templos, comida"
    budget: str            # "Medio"
    
    # ========== CONTROL DE FLUJO ==========
    supervisor_decision: str    # "CONTINUE" o "INVESTIGATE"
    interview_complete: bool    # False → True cuando listo
    
    # ========== RESULTADOS ==========
    destination_research: str   # Info procesada de Wikipedia
    travel_itinerary: str       # Itinerario completo
```

### Ciclo de Vida del Estado
```
1. Inicialización:
   state = {
       "messages": [],
       "destination": "",
       ...
       "interview_complete": False
   }

2. Fase de Entrevista:
   messages → [AI, Human, AI, Human, AI, Human, AI, Human]
   destination, duration, activities, budget → se van llenando

3. Detección de Completitud:
   interview_complete → True
   supervisor_decision → "INVESTIGATE"

4. Fase de Investigación:
   destination_research → "Tokio es la capital de..."

5. Fase de Generación:
   travel_itinerary → "DÍA 1: Llegada a Tokio..."

6. Finalización:
   Estado completo disponible para mostrar resultados
```

---

## 🔄 Flujo de Ejecución

### Flujo Completo Paso a Paso
```
┌─────────────────────────────────────────┐
│ 1. INICIO                               │
│    - Usuario ejecuta: await run_travel_planner() │
│    - Se crea thread_id único            │
│    - Estado inicial vacío               │
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│ 2. PRIMERA INTERACCIÓN                  │
│    - Grafo ejecuta nodo Entrevistador   │
│    - Sin mensajes → genera saludo       │
│    - Muestra: "¿A dónde quieres viajar?"│
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│ 3. USUARIO RESPONDE                     │
│    - Input: "Tokio"                     │
│    - Se agrega HumanMessage al estado   │
│    - Grafo continúa ejecución           │
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│ 4. LOOP DE ENTREVISTA (2-5 veces)      │
│    ┌────────────────────────────────┐   │
│    │ Entrevistador → pregunta       │   │
│    │ Usuario → responde             │   │
│    │ Supervisor → evalúa            │   │
│    │ Router → decide CONTINUE       │   │
│    └───────────┬────────────────────┘   │
│                ↓                        │
│            (repite hasta completo)      │
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│ 5. DETECCIÓN DE COMPLETITUD            │
│    - Supervisor detecta 4 respuestas    │
│    - interview_complete = True          │
│    - Router decide: INVESTIGATE         │
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│ 6. INVESTIGACIÓN                        │
│    - Wikipedia node extrae "Tokio"      │
│    - Busca en Wikipedia API             │
│    - LLM procesa información            │
│    - Guarda en destination_research     │
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│ 7. GENERACIÓN DE ITINERARIO            │
│    - Generador lee toda la conversación │
│    - Combina con info de Wikipedia      │
│    - Crea itinerario día por día        │
│    - Guarda en travel_itinerario        │
└────────────┬────────────────────────────┘
             ↓
┌─────────────────────────────────────────┐
│ 8. PRESENTACIÓN DE RESULTADOS          │
│    - Muestra destination_research       │
│    - Muestra travel_itinerary           │
│    - Mensaje de despedida               │
│    - Sistema termina                    │
└─────────────────────────────────────────┘
```

### Tiempo de Ejecución Típico

- Fase de entrevista: 2-3 minutos (4-5 turnos)
- Investigación: 5-10 segundos
- Generación: 10-20 segundos
- **Total**: ~3-4 minutos por sesión completa

---

## 📥 Instalación

### Requisitos Previos

- Python 3.10 o superior
- Cuenta de OpenAI con API Key
- Google Colab (recomendado) o entorno local con Jupyter

### Paso 1: Instalar Dependencias
```bash
pip install langgraph langchain langchain-openai wikipedia termcolor python-dotenv -q
```

### Paso 2: Configurar API Key
1. Click en 🔑 (panel izquierdo)
2. Agregar nuevo secreto:
   - **Name**: `OPENAI_API_KEY`
   - **Value**: `sk-proj-...` (tu API key)
   - **Toggle**: ✅ Activado
3. Dar acceso al notebook


### Paso 3: Verificar Instalación
```python
# Ejecutar esto para verificar
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini")
print("✅ Sistema listo")
```

---

## 🚀 Uso

### Ejecución Básica
```python
# En una celda de Jupyter/Colab
await run_travel_planner()
```

### Ejemplo de Sesión
```
╔══════════════════════════════════════════════════════════╗
║    ✈️  PLANIFICADOR DE VIAJES MULTI-AGENTE ✈️           ║
╚══════════════════════════════════════════════════════════╝

🔑 Sesión iniciada: travel_session_a1b2c3d4

🤖 Asistente: ¡Hola! 👋 Soy tu asistente personal de viajes.
              ¿A qué destino te gustaría viajar? 🌍✈️

👤 Tú: Tokio
────────────────────────────────────────────────────────────

🤖 Asistente: ¡Excelente elección! Tokio es fascinante. 
              ¿Cuántos días planeas quedarte?

👤 Tú: 5 días
────────────────────────────────────────────────────────────

🤖 Asistente: Perfecto, 5 días en Tokio. 
              ¿Qué tipo de actividades te interesan más?

👤 Tú: Cultura, templos y comida japonesa
────────────────────────────────────────────────────────────

🤖 Asistente: ¡Me encanta! Tokio tiene increíbles opciones gastronómicas.
              ¿Cuál es tu presupuesto aproximado? (bajo, medio o alto)

👤 Tú: Medio
────────────────────────────────────────────────────────────

[Sistema procesa automáticamente...]

============================================================
✅ PLANIFICACIÓN COMPLETADA
============================================================

📚 INFORMACIÓN DEL DESTINO:
────────────────────────────────────────────────────────────
Tokio es la capital de Japón y una de las ciudades más 
pobladas del mundo. Combina tradición milenaria con 
tecnología de vanguardia...

Principales atracciones:
- Templo Senso-ji (más antiguo de Tokio)
- Torre de Tokio con vistas panorámicas
- Palacio Imperial
- Santuario Meiji
- Barrio de Shibuya con su famoso cruce
...

🗓️  TU ITINERARIO PERSONALIZADO:
════════════════════════════════════════════════════════════

# Itinerario de 5 Días en Tokio 🗾✨

## DÍA 1: Llegada y Exploración de Shibuya 🌆

### Mañana (9:00-12:00)
🚶 **Cruce de Shibuya**
   El cruce peatonal más transitado del mundo
   💡 Tip: Mejor vista desde Starbucks segundo piso

☕ **Desayuno en Streamer Coffee**
   Café de especialidad japonés
   💰 ~1000 yen

🛍️ **Shibuya 109**
   Centro comercial icónico
   
### Tarde (12:00-18:00)
🍜 **Almuerzo: Ichiran Ramen**
   Ramen personalizado en cubículos privados
   💰 ~1200 yen

⛩️ **Santuario Meiji**
   Santuario sintoísta en bosque urbano
   💡 Gratis, abierto hasta 17:00

🌳 **Paseo por Parque Yoyogi**
   
### Noche (18:00-22:00)
🍱 **Cena en Izakaya local (Torikizoku)**
   Ambiente japonés auténtico
   💰 ~2500 yen

🌃 **Shibuya Sky**
   Mirador a 230m de altura
   💰 2500 yen

---

## DÍA 2: Cultura Tradicional en Asakusa ⛩️

### Mañana (8:00-12:00)
⛩️ **Templo Senso-ji**
   Llega temprano para evitar multitudes
   💡 Toma el metro Ginza Line hasta Asakusa

🏮 **Calle Nakamise**
   Souvenirs tradicionales
   
### Tarde (12:00-17:00)
🍱 **Almuerzo: Tempura Daikokuya**
   Tempura tradicional desde 1887
   💰 ~1500 yen

🗼 **Tokyo Skytree** (vista externa)
   La torre más alta de Japón
   
### Noche (17:00-21:00)
🍜 **Cena: Ramen Street (Tokyo Station)**
   8 restaurantes de ramen famosos
   💰 ~1000 yen

🌃 **Tokyo Station de noche**
   Arquitectura iluminada espectacular

[... continúa hasta día 5 ...]

✈️  ¡Buen viaje! 🌍
```

### Comandos Durante la Ejecución

- Escribe tus respuestas naturalmente
- Usa `salir`, `exit` o `quit` para terminar anticipadamente

---

## 🧠 Conceptos Avanzados

### 1. Sistema de Respaldo del Supervisor

**Problema**: El LLM no siempre genera la frase exacta "ENTREVISTA_COMPLETA".

**Solución**: Doble mecanismo de detección.

**Mecanismo Primario**:
- Busca frases clave en el último mensaje
- Lista de señales: "entrevista_completa", "tengo todo", "listo para planificar"

**Mecanismo de Respaldo** (más robusto):
```
Para cada tema clave, buscar en TODA la conversación:
├── ¿Se mencionó un destino? → has_destination
├── ¿Se especificó duración? → has_duration
├── ¿Se describieron actividades? → has_activities
└── ¿Se indicó presupuesto? → has_budget

Si las 4 son True Y se hicieron ≥4 preguntas:
    → decision = "INVESTIGATE"
```

**Ventaja**: Sistema robusto que no depende 100% del LLM.

---

### 2. Extracción Estructurada de Información

**Problema**: El Wikipedia node necesita saber exactamente qué buscar.

**Solución**: Usar LLM para extraer info estructurada de conversación natural.

**Prompt de extracción**:
```
Analiza esta conversación:
[conversación completa]

Extrae en formato:
DESTINO: [ciudad/país]
DURACIÓN: [X días]
ACTIVIDADES: [lista]
PRESUPUESTO: [bajo/medio/alto]
```

**Resultado**:
```
DESTINO: Tokio
DURACIÓN: 5 días
ACTIVIDADES: Cultura, templos, comida
PRESUPUESTO: Medio
```

**Ventaja**: Conversión de texto libre → datos estructurados.

---

### 3. Streaming vs Invoke

**Diferencia fundamental**:

**`ainvoke` (no usado)**:
- Ejecuta TODO el grafo de una vez
- No permite pausas
- No apto para interacción usuario

**`astream` (usado)**:
- Ejecuta nodo por nodo
- Permite pausar entre nodos
- Control fino del flujo
- Apto para interacción

**Implementación**:
```python
async for event in travel_graph.astream(state, config):
    for node_name, node_output in event.items():
        if node_name == "interviewer":
            # Pausar aquí para pedir input usuario
            break
```

---

### 4. Prompt Engineering Específico por Rol

Cada agente tiene un prompt optimizado para su tarea:

**Entrevistador**:
- Tono: Amigable, entusiasta
- Estilo: Conversacional
- Instrucción: UNA pregunta a la vez

**Supervisor**:
- Tono: Neutral, analítico
- Estilo: Determinístico
- Instrucción: Responder solo "CONTINUE" o "INVESTIGATE"

**Investigador**:
- Tono: Informativo, práctico
- Estilo: Estructurado
- Instrucción: Filtrar info relevante para viajeros

**Generador**:
- Tono: Creativo, detallista
- Estilo: Organizado por días
- Instrucción: Incluir nombres específicos y horarios

---

### 5. Manejo de Errores de Wikipedia

**Casos manejados**:

1. **Desambiguación** (múltiples resultados):
```python
   except wikipedia.exceptions.DisambiguationError as e:
       # Usar la primera opción automáticamente
       resumen = wikipedia.summary(e.options[0])
```

2. **Página no encontrada**:
```python
   except wikipedia.exceptions.PageError:
       return "No se encontró información"
```

3. **Otros errores**:
```python
   except Exception as e:
       return f"Error: {str(e)}"
```

**Ventaja**: Sistema robusto que no falla si Wikipedia no tiene info.

---

## 📁 Estructura del Proyecto

### Organización en Celdas (Jupyter/Colab)
```
Notebook: planificador_viajes_multiagente.ipynb
│
├── CELDA 1: 📦 Instalación de Dependencias
│   └── pip install langgraph, langchain, openai, wikipedia, termcolor
│
├── CELDA 2: 🔑 Configuración de API Keys
│   ├── Cargar OPENAI_API_KEY desde secretos
│   └── Verificación de configuración
│
├── CELDA 3: 🤖 Configuración de LLM y Herramientas
│   ├── Inicialización de ChatOpenAI (gpt-4o-mini)
│   ├── Función buscar_destino_wikipedia()
│   └── Prueba de funcionamiento
│
├── CELDA 4: 📊 Definición del Estado Compartido
│   └── class TravelState(TypedDict)
│       ├── messages (con reducción)
│       ├── destination, duration, activities, budget
│       ├── supervisor_decision, interview_complete
│       └── destination_research, travel_itinerary
│
├── CELDA 5: 📝 Prompts Especializados
│   ├── interviewer_prompt (ChatPromptTemplate)
│   ├── supervisor_prompt (no usado, sustituido por lógica)
│   ├── wikipedia_prompt (ChatPromptTemplate)
│   └── study_guide_prompt (ChatPromptTemplate)
│
├── CELDA 6: 🔧 Implementación de Nodos
│   ├── async def interviewer_node(state)
│   ├── async def supervisor_node(state)
│   ├── async def wikipedia_node(state)
│   └── async def study_guide_node(state)
│
├── CELDA 7: 🔀 Función de Routing
│   └── def supervisor_router(state) -> Literal["CONTINUE", "INVESTIGATE"]
│
├── CELDA 8: 🏗️ Construcción del Grafo
│   └── def create_travel_system()
│       ├── workflow = StateGraph(TravelState)
│       ├── workflow.add_node() × 4
│       ├── workflow.add_edge() × 3
│       ├── workflow.add_conditional_edges()
│       └── return workflow.compile(checkpointer=MemorySaver())
│
└── CELDA 9: 🎮 Playground Interactivo
    └── async def run_travel_planner()
        ├── Inicialización de sesión
        ├── Loop de conversación con streaming
        ├── Mostrar resultados finales
        └── Manejo de comandos de salida
```

### Flujo de Datos
```
Usuario → CELDA 9 → Grafo (CELDA 8) → Nodos (CELDA 6) → LLM (CELDA 3)
                           ↓
                    Estado (CELDA 4)
                           ↓
                 Wikipedia API (CELDA 3)
                           ↓
                 Usuario ← Resultados
```

---


## 🔧 Troubleshooting

### Problemas Comunes y Soluciones

#### 1. Error: "Rate Limit Exceeded"

**Síntoma**: `429 - You exceeded your current quota`

**Causa**: Límite de API agotado

**Soluciones**:
- **Gemini**: Esperar al reset diario (medianoche PST)
- **OpenAI**: Agregar créditos a la cuenta ($5 mínimo)
- **Temporal**: Usar modelo más ligero o reducir requests

#### 2. Error: "Recursion Limit"

**Síntoma**: `GraphRecursionError: Recursion limit of 25 reached`

**Causa**: Loop infinito (supervisor nunca detecta finalización)

**Solución**:
```python
config = {
    "configurable": {"thread_id": thread_id},
    "recursion_limit": 50  # Aumentar límite
}
```

#### 3. Sistema no pide respuestas (corre automáticamente)

**Síntoma**: El grafo ejecuta todos los nodos sin pausar

**Causa**: Usando `ainvoke` en lugar de `astream`

**Solución**: Usar streaming en CELDA 9

#### 4. Itinerario genérico ("no especificado")

**Síntoma**: El itinerario dice "necesito que me proporciones..."

**Causa**: Wikipedia node no extrae bien la información

**Solución**: Verificar que CELDA 6 tenga versión mejorada de `wikipedia_node`

---


## 📚 Recursos y Referencias

### Documentación Oficial

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [LangGraph Multi-Agent Tutorial](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/)
- [LangChain Documentation](https://python.langchain.com/docs/get_started/introduction)
- [OpenAI API Documentation](https://platform.openai.com/docs/)
- [Wikipedia API for Python](https://wikipedia.readthedocs.io/)

### Guía Base

Este proyecto está basado en:
**"Building Multi-Agent Systems with LangGraph"**
- Tutorial oficial de LangChain
- Patrón: Interviewer + Supervisor + Researchers
- Adaptado de dominio educativo a viajes

### Artículos Relacionados

- [State Management in LangGraph](https://langchain-ai.github.io/langgraph/concepts/low_level/#state)
- [Conditional Edges](https://langchain-ai.github.io/langgraph/how-tos/branching/)
- [Checkpointing and Memory](https://langchain-ai.github.io/langgraph/how-tos/persistence/)

---

## 👨‍💻 Autor

**Yvonne Echevarria**

Proyecto desarrollado como parte del aprendizaje de sistemas multi-agente con LangGraph.

---

## 🎉 Estado del Proyecto

**✅ COMPLETADO Y FUNCIONAL**

El sistema está completamente operativo y genera itinerarios de viaje personalizados mediante la coordinación de 4 agentes especializados trabajando con LangGraph.

**Versión**: 1.0  
**Última actualización**: Octubre 2025  
**Status**: Producción estable

---
