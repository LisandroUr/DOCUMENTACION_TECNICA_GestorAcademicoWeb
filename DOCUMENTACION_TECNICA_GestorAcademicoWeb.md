# LogSentinel AI - Documentación Técnica

Sistema Automatizado de Auditoría, Clasificación y Diagnóstico de Logs de Servidor con IA Local

---

## 📝 Descripción General

**LogSentinel AI** es una solución de monitoreo, análisis y diagnóstico automatizado de infraestructura desarrollada para entornos de servidor Linux. Su propósito principal es eliminar la sobrecarga operativa y el tiempo requerido para auditar manualmente grandes volúmenes de eventos y archivos de registro (`/var/log/`). 

El sistema ingiere eventos del sistema operativo y servicios web de forma continua y no bloqueante, normaliza datos desestructurados en esquemas limpios y utiliza un Modelo de Lenguaje de Gran Escala (LLM) desplegado 100% de manera local para categorizar la severidad de incidentes, diagnosticar la causa raíz de fallos y proponer pasos concretos de remediación técnica. Todo esto se ejecuta de forma autónoma, sin comprometer datos confidenciales hacia servicios en la nube y persistiendo el histórico para trazabilidad y auditorías.

---

## 🛠️ Stack Tecnológico

* **Sistema Operativo Base:** Linux Debian 12 (Bookworm) (Entorno de desarrollo, pruebas y servidor de ejecución).
* **Lenguajes / Entornos:** Python 3.10+ (Motor de ingestión asincrónica, normalización y orquestación con `asyncio`, `aiofiles` y `httpx`) / JavaScript ES6+ (Scripts auxiliares de automatización y consumo de eventos).
* **Base de Datos / Persistencia:** MariaDB 10.11 / PostgreSQL 15 (Almacenamiento relacional estructurado con esquemas indexados por timestamp, nivel de severidad, servicio y código de error).
* **Componentes de IA / Automatización:** Ollama ejecutando modelos locales cuantizados (Llama 3 8B Instruct / Mistral 7B) en host local (`http://127.0.0.1:11434`), consumido vía API REST con modo estricto de salida JSON.

---

## 🔄 Flujo de Datos y Arquitectura

```mermaid
flowchart LR
    A["/var/log/* (Syslog, Nginx, Auth)"] -->|"Lectura asíncrona (aiofiles)"| B["Ingestor & Sanitizador (Python)"]
    B -->|"Regex & Máscaras de datos sensibles"| C["Parser a JSON Estructurado"]
    C -->|"Prompt Engineering (Modo JSON)"| D["API Local Ollama (Llama 3 / Mistral)"]
    D -->|"Diagnóstico + Severidad + Remediación"| E["Validador de Esquema (Pydantic)"]
    E -->|"Persistencia e Índices"| F["MariaDB / PostgreSQL (Auditoría)"]
```

1. **Entrada de datos:**
   Un servicio *daemon* en segundo plano monitorea y lee de forma continua y asincrónica los flujos de logs del sistema (`/var/log/syslog`, `/var/log/nginx/error.log`, `/var/log/auth.log`) mediante colas de lectura no bloqueantes (`aiofiles`), detectando nuevas líneas en tiempo real mediante punteros de archivo (*tailing*).
2. **Procesamiento lógico y sanitización:**
   El pipeline en Python depura los registros entrantes: descarta eventos irrelevantes mediante filtros por patrones regex, enmascara datos sensibles (direcciones IP privadas, credenciales accidentales, hashes de sesión) y convierte el texto plano en una estructura de datos estándar en formato JSON con campos normalizados (`timestamp`, `service`, `raw_message`, `facility`).
3. **Interacción con la IA Local:**
   El backend realiza una petición HTTP asincrónica a la API REST de Ollama (`/api/generate`). Se implementa *Prompt Engineering* con instrucciones de sistema delimitadas y *few-shot examples* para exigir una respuesta estrictamente estructurada en formato JSON que contenga:
   - Nivel de criticidad (`CRITICAL`, `WARNING`, `INFO`).
   - Categoría técnica del problema (Red, Permisos, Base de Datos, Agotamiento de Recursos, Kernel).
   - Diagnóstico resumido de la causa raíz.
   - Acción o comando de remediación recomendado para el administrador.
4. **Almacenamiento / Salida:**
   El diagnóstico estructurado emitido por el modelo es validado sintácticamente y persistido en una base de datos MariaDB/PostgreSQL en la tabla `audit_logs`. Esta información queda indexada para auditorías posteriores, generación de métricas y eventual disparo de alertas a canales de administración en caso de eventos con severidad crítica.

---

## 🛡️ Consideraciones de Seguridad y Buenas Prácticas

* **Privacidad Absoluta (Zero Data Egress):**
  Al desplegar modelos de lenguaje de forma 100% local (*Self-Hosted*) sobre el entorno Debian, ningún registro del sistema operativo, traza interna ni dato de infraestructura abandona el servidor. Esto garantiza el cumplimiento estricto de normativas de confidencialidad y previene fugas de información hacia APIs públicas de terceros.
* **Principio de Mínimo Privilegio:**
  El servicio de auditoría se ejecuta bajo un usuario de sistema dedicado sin privilegios de superusuario (`logaudit`). El acceso a los archivos de registro se otorga de manera granular mediante Listas de Control de Acceso POSIX (`setfacl -m u:logaudit:r /var/log/...`), impidiendo cualquier capacidad de modificación o escritura sobre los logs y eliminando vectores de escalada de privilegios.
* **Gestión Segura de Entornos:**
  Uso riguroso de variables de entorno mediante archivos `.env` (excluidos explícitamente del repositorio a través de `.gitignore`), aislando credenciales de conexión a bases de datos, tokens de notificación interna y parámetros de red del entorno de ejecución.

---

## 🚀 Desafíos Técnicos y Resoluciones (Troubleshooting)

* **Desafío 1: Alto consumo de memoria y saturación por ráfagas de logs pesados.**
  * *Síntoma:* En incidentes graves de infraestructura donde se generaban miles de líneas por segundo, la lectura ingenua provocaba picos de consumo de memoria RAM y bloqueo del hilo de procesamiento.
  * *Resolución:* Implementación de un pipeline asincrónico por lotes (*batches*) con colas en memoria (`asyncio.Queue`) y ventanas de tiempo deslizantes. Se limitó el tamaño de los lotes y se aplicó *backpressure*, garantizando un consumo de RAM estable y predecible en el servidor Debian.

* **Desafío 2: Variabilidad en las respuestas del LLM y fallos de parseo sintáctico.**
  * *Síntoma:* Ocasionalmente, el modelo de lenguaje introducía texto conversacional antes o después del JSON, provocando excepciones en el parser del backend.
  * *Resolución:* Configuración del parámetro nativo `format="json"` de Ollama acoplado a un esquema estricto validado mediante modelos Pydantic en Python. Se implementó una capa de validación resiliente con mecanismo de reintento automático y fallback a categorización heurística por regex si la respuesta no superaba la validación.

* **Desafío 3: Latencia en inferencia durante eventos de error recurrentes.**
  * *Síntoma:* Logs repetitivos generaban llamadas redundantes al modelo local, incrementando el uso de CPU/GPU innecesariamente.
  * *Resolución:* Implementación de una caché en memoria basada en huellas digitales (hashing SHA-256 del mensaje normalizado). Si un error con el mismo patrón ocurre dentro de una ventana de 5 minutos, se reutiliza el diagnóstico previo e incrementa el contador de recurrencia en MariaDB sin reenviar la petición a la IA.
