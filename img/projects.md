# Guia de Desarrollo: Proyectos del Portafolio

Este documento contiene los planos tecnicos completos (blueprints) para que puedas desarrollar desde cero cada uno de los 5 proyectos. Incluye arquitectura, esquemas de bases de datos, flujos de ejecucion y planes paso a paso.

Al final del documento encontraras el Roadmap con la recomendacion exacta de por cual proyecto comenzar.

---

## 01. ai-agent-saas-platform

### 1. Concepto y Valor de Negocio
Un SaaS (Software as a Service) donde usuarios pagan una suscripcion para delegar tareas complejas a agentes autonomos. Mientras el agente investiga, navega o analiza, el usuario ve el progreso en tiempo real en un dashboard. Demuestra que sabes construir un producto monetizable, no solo un script.

### 2. Arquitectura y Tech Stack
- **Framework Backend:** Django + Django REST Framework.
- **Orquestador IA:** LangGraph (para definir el grafo de estados del agente).
- **Asincronia:** Celery (Workers) + Redis (Broker).
- **Base de Datos:** PostgreSQL.
- **Frontend:** Vue.js 3 + TypeScript.
- **Pagos:** Stripe API.

### 3. Esquema de Base de Datos (Tablas Clave)
- `User`: id, email, password, stripe_customer_id, is_active_subscriber.
- `AgentTask`: id, user_id, prompt, status (pending, running, success, failed), final_result, created_at.
- `TaskLog`: id, task_id, step_name, content, timestamp. (Guarda lo que el agente piensa en cada paso).

### 4. Plan de Implementacion Paso a Paso
1. **Infraestructura Base:** Configura Django, PostgreSQL y Celery con Redis.
2. **Autenticacion y Pagos:** Implementa login JWT. Conecta Stripe Checkout y crea un endpoint de Webhook para actualizar el campo `is_active_subscriber`.
3. **Desarrollo del Agente (Aislado):** Crea un archivo Python puro usando LangGraph. Define un nodo "Investigador" y un nodo "Redactor".
4. **Integracion Asincrona:** Crea una tarea de Celery que reciba un `task_id`, ejecute el grafo de LangGraph, y por cada paso que de el agente, inserte un registro en `TaskLog`.
5. **WebSockets (Tiempo Real):** Usa Django Channels. Cuando Celery guarde un `TaskLog`, emite un evento por WebSocket al frontend.
6. **Frontend:** En Vue.js, crea el panel donde el usuario envia su prompt y una terminal visual que se actualiza via WebSocket.

---

## 02. enterprise-rag-gateway

### 1. Concepto y Valor de Negocio
Las grandes empresas no pueden usar ChatGPT publico por privacidad. Este proyecto es una API RAG B2B multi-tenant: la Empresa A y la Empresa B suben sus documentos al mismo servidor, pero una jamas puede buscar en los datos de la otra. Incluye un sistema que auto-evalua si la IA esta mintiendo (alucinando).

### 2. Arquitectura y Tech Stack
- **Framework Backend:** FastAPI.
- **Motor de Ingesta/Recuperacion:** LlamaIndex (ideal para parsear PDFs complejos con tablas).
- **Base de Datos / Vector Store:** PostgreSQL con la extension `pgvector`.
- **Seguridad:** Row-Level Security (RLS) nativo de PostgreSQL.
- **Evaluacion:** Libreria `Ragas`.

### 3. Esquema de Base de Datos (Tablas Clave)
- `Tenant`: id, company_name, api_key.
- `Document`: id, tenant_id, filename, upload_date.
- `VectorEmbedding`: id, document_id, tenant_id, embedding (tipo vector), text_chunk.

### 4. Plan de Implementacion Paso a Paso
1. **Configuracion BD Multi-tenant:** Instala pgvector. Configura politicas RLS en la tabla `VectorEmbedding` para que las queries SQL requieran un `tenant_id` en el contexto de la sesion.
2. **API Base:** Crea en FastAPI el middleware de autenticacion que identifica el `tenant_id` basandose en un Bearer Token.
3. **Pipeline de Ingesta:** Crea un endpoint `/upload`. Usa LlamaIndex para leer el PDF, dividirlo en chunks y guardar los embeddings asegurandote de adjuntar el `tenant_id`.
4. **Pipeline de Busqueda Hibrida:** Crea el endpoint `/chat`. Configura LlamaIndex para buscar combinando similitud semantica (pgvector) y busqueda por palabras clave (BM25).
5. **Framework de Evaluacion:** Crea un script independiente usando `Ragas` que tome 50 preguntas de prueba y evalue "Faithfulness" (Fidelidad) y "Answer Relevance" de tus respuestas.

---

## 03. llm-observability-proxy

### 1. Concepto y Valor de Negocio
Las startups gastan miles de dolares llamando a la API de OpenAI repetidamente con las mismas preguntas. Este proyecto es un "Proxy Intermediario". La app del cliente le habla a tu Proxy, y tu Proxy decide si ya tiene la respuesta en cache o si debe llamar a OpenAI. Ademas, emite metricas de gasto en vivo.

### 2. Arquitectura y Tech Stack
- **Proxy Inverso:** FastAPI.
- **Cache Semantica:** Redis (usando redis-vl o buscando similitud de vectores en cache).
- **Almacenamiento de Metricas:** PostgreSQL.
- **Dashboard:** Vue.js + TypeScript + WebSockets.

### 3. Esquema de Base de Datos (Tablas Clave)
- `ApiRequest`: id, user_id, prompt_tokens, completion_tokens, total_cost_usd, model, latency_ms, was_cached, created_at.

### 4. Plan de Implementacion Paso a Paso
1. **El Proxy Transparente:** En FastAPI, crea una ruta catch-all `/v1/chat/completions`. Toma el request del usuario, enviaselo a OpenAI con `httpx` asincrono, y devuelve la respuesta.
2. **Cache Semantica:** Antes de llamar a OpenAI, has un embedding rapido del prompt del usuario (usa un modelo pequeno y local como sentence-transformers). Busca en Redis si hay un prompt similar al 95%. Si lo hay, devuelve la respuesta vieja y ahorra el 100% del dinero.
3. **Registro de Metricas:** Extrae los `usage_tokens` de la respuesta de OpenAI. Guardalos asincronamente (BackgroundTasks de FastAPI) en `ApiRequest`.
4. **Streaming por WebSockets:** Por cada registro guardado, emite un JSON por WebSocket con el costo de la peticion.
5. **Dashboard UI:** Construye graficos de lineas en Vue.js que suban en tiempo real mostrando el "Gasto Acumulado" y el "Dinero Ahorrado por Cache".

---

## 04. bayesian-predictive-pipeline

### 1. Concepto y Valor de Negocio
Conectar tu publicacion cientifica con ingenieria real. Imagina sensores agricolas o de salud enviando datos constantes. Este pipeline los recolecta, ejecuta tus Modelos Espaciales Bayesianos (INLA/CAR) para encontrar anomalias que el ML tradicional no detecta, y avisa en un panel en vivo.

### 2. Arquitectura y Tech Stack
- **Framework Web:** Django.
- **Estadistica / Matematica:** Script en R (usando la libreria INLA).
- **Interoperabilidad:** Libreria `rpy2` para ejecutar R desde Python, o correr R en un contenedor Docker separado via API REST.
- **Tiempo Real:** Django Channels (WebSockets).

### 3. Flujo de Ejecucion (Pipeline)
1. **Endpoint Ingesta:** `/api/sensor/data` recibe un JSON con {id_sensor, lat, lon, valor, timestamp}.
2. **Agrupamiento:** Una tarea en Celery se ejecuta cada 5 minutos, toma todos los datos recientes.
3. **Inferencia Bayesiana:** Python llama al script de R pasandole un CSV. R ejecuta INLA, calcula el indice de autocorrelacion de Moran y las probabilidades predictivas.
4. **Alerta:** Si R devuelve una probabilidad de anomalia > 80%, Django Channels dispara una alerta roja por WebSocket al frontend.

---

## 05. edge-llm-inference-server

### 1. Concepto y Valor de Negocio
Demostrar que sabes de infraestructura IA, no solo de APIs. Crear tu propio clon de OpenAI que corre en servidores privados usando LLMs Open Source (como Llama 3) para empresas que no pueden sacar su informacion a internet.

### 2. Arquitectura y Tech Stack
- **Framework:** FastAPI.
- **Motor de Inferencia:** `llama-cpp-python`.
- **Concurrencia:** `asyncio` y Semaphoros.
- **Contenedores:** Docker (configurado para soporte GPU/CUDA si es posible).

### 3. Retos y Plan de Implementacion
1. **Carga del Modelo:** Escribe un script que descargue un modelo en formato `.gguf` (ej. Llama-3-8B-Instruct.Q4_K_M.gguf - cuantizado a 4 bits para usar poca RAM).
2. **Endpoint Compatible:** Crea un endpoint `/v1/chat/completions` que reciba el JSON exactamente en el mismo formato que pide OpenAI.
3. **Manejo de Concurrencia:** Si 10 usuarios piden texto a la vez, tu VRAM explotara. Debes implementar un sistema de colas en memoria (`asyncio.Queue`) para procesar las peticiones en batches o de forma secuencial sin que el servidor crashee.
4. **Generacion por Streaming:** Usa Server-Sent Events (SSE) para devolver el texto palabra por palabra, mejorando la experiencia de usuario.

---

## Roadmap: Por cual proyecto comenzar (Orden Estrategico)

Para maximizar tu motivacion y armar tu portafolio de manera inteligente, sigue este orden:

### FASE 1: Victorias Rapidas y Seguras
**Empieza por el Proyecto 02 (enterprise-rag-gateway)**
- *¿Por que?* Ya construiste un RAG basico. El salto conceptual es pequeno, pero el salto de calidad arquitectonica es enorme. Aprender LlamaIndex y pgvector te tomara unas semanas, y tendras un proyecto "Enterprise" muy rapido.

### FASE 2: Innovacion y Backend Duro
**Sigue con el Proyecto 03 (llm-observability-proxy)**
- *¿Por que?* Es un proyecto muy enfocado en Backend puro (Proxys, Cache, WebSockets) y resuelve un problema hiper-real (el costo de las APIs). Te posicionara como alguien que cuida el dinero de la empresa.

### FASE 3: El Masterpiece Comercial
**Sigue con el Proyecto 01 (ai-agent-saas-platform)**
- *¿Por que?* Es el mas largo de hacer porque involucra integracion con pagos (Stripe), dashboards complejos y flujos asincronos (Celery + LangGraph). Cuando termines este, tendras un MVP que incluso podrias vender.

### FASE 4: El Diferenciador Unico
**Sigue con el Proyecto 04 (bayesian-predictive-pipeline)**
- *¿Por que?* Es tu as bajo la manga. Dejalo para esta fase porque la integracion entre R y Django puede ser tediosa a nivel de infraestructura, pero es vital que exista en tu CV para darle peso a tu publicacion academica.

### FASE 5: Infraestructura Pura
**Finaliza con el Proyecto 05 (edge-llm-inference-server)**
- *¿Por que?* Requiere lidiar con hardware, compilacion de librerias C++ y limites de memoria. Es un proyecto fantastico, pero muy de nicho (MLOps). Haslo ultimo para completar la corona de tu portafolio.
