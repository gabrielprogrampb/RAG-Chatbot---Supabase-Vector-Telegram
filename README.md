# RAG Supabase + Telegram

Este proyecto implementa un sistema de Recuperación Aumentada por Generación (RAG) que combina Supabase como base de datos vectorial y PostgreSQL, y Telegram como interfaz de chat.

## Descripción General

El sistema consta de dos flujos principales:

1.  **Ingesta de Datos:**
    *   Escanea la carpeta especificada en Google Drive en busca de nuevos archivos o archivos modificados.
    *   Descarga los archivos encontrados.
    *   Procesa los archivos: los divide en fragmentos, genera embeddings (representaciones vectoriales) de los mismos utilizando modelos de OpenAI y los almacena en Supabase (PostgreSQL).
    *   Registra metadatos de los archivos procesados en una tabla de Supabase para evitar duplicados.

2.  **Chat vía Telegram:**
    *   Un usuario envía un mensaje a través de Telegram.
    *   El mensaje es recibido por el bot de Telegram.
    *   Un agente de IA (utilizando Gemini Flash) recibe el mensaje del usuario.
    *   El agente consulta la base de datos vectorial de Supabase para encontrar información relevante relacionada con la consulta del usuario.
    *   El agente utiliza la información recuperada y su propio conocimiento para generar una respuesta.
    *   La respuesta, formateada en HTML según reglas específicas, es enviada de vuelta al usuario a través de Telegram.
    *   Se utiliza memoria de chat en PostgreSQL para mantener el contexto de la conversación.

## Componentes Principales

*   **Google Drive:** Origen de los documentos a indexar.
*   **Supabase (PostgreSQL):**
    *   Almacena los embeddings vectoriales de los documentos.
    *   Almacena metadatos de los archivos indexados (`ingested_files`).
    *   Almacena la memoria de la conversación (`chat_memory`).
*   **n8n:** Plataforma de automatización que orquesta los diferentes nodos y flujos.
*   **OpenAI Embeddings:** Utilizado para generar las representaciones vectoriales de los documentos.
*   **Google Gemini Flash:** Modelo de lenguaje grande utilizado como agente de IA para procesar consultas y generar respuestas.
*   **Telegram:** Interfaz de usuario para interactuar con el sistema de chat.

## Configuración

### Pre-requisitos

*   Cuenta de n8n configurada.
*   Credenciales para:
    *   Google Drive API (OAuth2).
    *   OpenAI API.
    *   Google Gemini API (PaLM).
    *   Supabase (PostgreSQL).
    *   Telegram Bot.

### Pasos de Configuración

1.  **Google Drive:**
    *   Configura una cuenta de Google Drive API en n8n.
    *   Define el `TU_GOOGLE_DRIVE_FOLDER_ID` en el nodo "Listar archivos Drive" para especificar la carpeta de origen de los documentos.

2.  **Supabase:**
    *   Configura una cuenta de Supabase (PostgreSQL) en n8n.
    *   Asegúrate de que las siguientes tablas existan en tu base de datos Supabase:
        *   `ingested_files` (con columnas como `file_id`, `name`, `mime_type`, `md5_checksum`, `modified_time`, `last_ingested`).
        *   `n8n_vectors` (para almacenar los vectores; la estructura específica dependerá del plugin de Langchain para PGVector).
        *   `chat_memory` (para el historial de conversaciones).
    *   Asegúrate de tener instalada la extensión `pgvector` en tu base de datos Supabase si utilizas el nodo `vectorStorePGVector`.

3.  **OpenAI:**
    *   Configura tus credenciales de OpenAI API en n8n.

4.  **Gemini:**
    *   Configura tus credenciales de Google Gemini API (PaLM) en n8n.

5.  **Telegram:**
    *   Crea un bot de Telegram y obtén su token. Configura las credenciales de Telegram API en n8n.
    *   Configura el `webhookId` en los nodos de Telegram Trigger y Telegram Response.

### Flujos de n8n

*   **Flujo de Ingesta:**
    *   Se puede activar manualmente (`Ejecutar Ingesta Manual`) o programarse (`Cron Ingesta`).
    *   Lista archivos de Google Drive, verifica si ya han sido cargados en Supabase.
    *   Si no han sido cargados, los descarga, extrae metadatos, los divide, genera embeddings y los inserta en Supabase.
    *   Actualiza la tabla `ingested_files` con la información del archivo.
    *   Vincula los vectores generados al `file_id` correspondiente.

*   **Flujo de Chat:**
    *   Se activa mediante un mensaje recibido en Telegram (`Telegram Trigger`).
    *   El mensaje del usuario se pasa al `Agente IA (Telegram)`.
    *   El agente utiliza `Gemini 2.5 Flash (Agente)` como modelo de lenguaje y `Herramienta Búsqueda Vectorial` para consultar Supabase (`Supabase (Consultar)`).
    *   La memoria de chat se gestiona a través de `Postgres Chat Memory`.
    *   La respuesta generada se envía de vuelta al usuario vía Telegram (`Responder Telegram`).

## Instrucciones de Uso

1.  **Ingesta de Documentos:**
    *   Coloca los documentos que deseas que el bot pueda consultar en la carpeta de Google Drive configurada.
    *   Inicia el flujo de ingesta manualmente o espera a que se ejecute según la programación establecida.

2.  **Interacción por Telegram:**
    *   Abre la conversación con tu bot de Telegram.
    *   Envía tus preguntas. El bot consultará los documentos indexados y responderá.

## Notas Importantes

*   La primera ejecución del flujo de ingesta puede tardar considerablemente, dependiendo de la cantidad y tamaño de los archivos.
*   El formato de respuesta en Telegram está diseñado para ser compatible con la limitación de HTML de la plataforma.
*   Asegúrate de que las credenciales en n8n estén correctamente configuradas y sean válidas.
*   La lógica de verificación de archivos en Supabase (`Verificar en Supabase`, `¿Ya fue cargado?`) ayuda a evitar la reindexación innecesaria de archivos.

