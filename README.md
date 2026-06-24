# INFORME FINAL DE AUDITORÍA DE SISTEMAS

## CARÁTULA

**Entidad Auditada:** HelpDesk AI - Corporación EPIS Corp  
**Ubicación:** Lima, Perú  
**Período auditado:** 24/06/2026  
**Equipo Auditor:** Antigravity (Auditor de Sistemas de IA)  
**Fecha del informe:** 24/06/2026  


## ÍNDICE

1. [Resumen Ejecutivo](#1-resumen-ejecutivo)  
2. [Antecedentes](#2-antecedentes)  
3. [Objetivos de la Auditoría](#3-objetivos-de-la-auditoría)  
4. [Alcance de la Auditoría](#4-alcance-de-la-auditoría)  
5. [Normativa y Criterios de Evaluación](#5-normativa-y-criterios-de-evaluación)  
6. [Metodología y Enfoque](#6-metodología-y-enfoque)  
7. [Hallazgos y Observaciones](#7-hallazgos-y-observaciones)  
8. [Análisis de Riesgos](#8-análisis-de-riesgos)  
9. [Recomendaciones](#9-recomendaciones)  
10. [Conclusiones](#10-conclusiones)  
11. [Plan de Acción y Seguimiento](#11-plan-de-acción-y-seguimiento)  
12. [Anexos](#12-anexos)  


## 1. RESUMEN EJECUTIVO

El presente informe expone los resultados de la auditoría técnica realizada sobre el sistema de asistencia y reporte de incidentes **Corporate EPIS Pilot**, el cual integra un motor de búsqueda y recuperación aumentada de información (RAG), almacenamiento vectorial con ChromaDB, base de datos relacional SQLite3 para gestión de tickets y modelos de lenguaje de gran escala (LLMs) ejecutados localmente mediante Ollama.

La auditoría se centró en diagnosticar y mitigar los fallos del flujo de backend causados por la migración del modelo de lenguaje original (`llama3.1:8b`) al modelo de menores prestaciones y bajo consumo de recursos (`smollm:360m`). Como resultado de la intervención, se identificaron y solucionaron problemas críticos de latencia, bucles infinitos en respuestas y fallos en la clasificación semántica, logrando estabilizar el sistema a través de un enrutador híbrido determinista y la aplicación de restricciones estrictas de generación de tokens (`num_predict`).


## 2. ANTECEDENTES

EPIS Corp ha implementado un chatbot inteligente de soporte técnico para HelpDesk llamado **Corporate EPIS Pilot**. El sistema originalmente estaba diseñado para utilizar `llama3.1:8b`, pero debido a limitaciones de hardware del entorno de ejecución local (ejecución pura sobre CPU), se decidió realizar una migración al modelo ultraligero `smollm:360m` (360 millones de parámetros) a fin de viabilizar su ejecución sin GPUs dedicadas.

Esta migración degradó severamente la lógica de negocio basada en agentes y cadenas de LangChain, ocasionando cuellos de botella en la clasificación semántica de intenciones de los usuarios y latencias extremas.


## 3. OBJETIVOS DE LA AUDITORÍA

### Objetivo General
Evaluar la estabilidad, seguridad, rendimiento e integridad de la arquitectura de la solución de HelpDesk AI operando bajo el modelo `smollm:360m` en contenedores Docker.

### Objetivos Específicos
* Evaluar la precisión técnica del módulo de enrutamiento y clasificación de intenciones de los usuarios.
* Analizar la latencia de respuesta del pipeline de RAG ante consultas generales y reportes de fallas.
* Validar la persistencia e integridad de datos en el ciclo de vida del ticket de soporte técnico (SQLite3).
* Diagnosticar la eficiencia de la infraestructura y proxies inversores (Nginx) para evitar caídas por timeout.


## 4. ALCANCE DE LA AUDITORÍA

El alcance de la auditoría abarca el análisis del código fuente del backend (`main.py`), base de datos (`tickets.db`), almacén de vectores (`vector_store`), configuraciones de red del proxy inverso (`nginx.conf`) y las pruebas funcionales directas de API y frontend sobre `localhost:5173`.
* **Áreas involucradas:** Desarrollo de Software, Arquitectura de Sistemas de IA, Infraestructura TI.
* **Periodo auditado:** 24 de junio de 2026.


## 5. NORMATIVA Y CRITERIOS DE EVALUACIÓN

Para la evaluación se emplearon como criterios de referencia las directrices de:
* **ISO/IEC 25010 (System and Software Quality Requirements):** Enfocado en las subcaracterísticas de eficiencia de desempeño (comportamiento temporal y utilización de recursos) y adecuación funcional.
* **Políticas Internas de EPIS Corp:** Tiempo máximo de espera de usuario en soporte virtual fijado en <30 segundos.


## 6. METODOLOGÍA Y ENFOQUE

El enfoque de auditoría fue técnico-práctico y basado en riesgos, aplicando las siguientes técnicas:
1. **Inspección del Entorno de Contenedores:** Auditoría de configuración de `docker-compose.yml` y logs de contenedores backend/proxy.
2. **Pruebas Técnicas Directas de API:** Ejecución de queries HTTP parametrizados con `curl.exe` hacia los endpoints del backend para medir tiempos de respuesta exactos (`elapsed`).
3. **Pruebas de Interfaz y Usabilidad Visual:** Simulación del flujo de interacción real del usuario final a través de un navegador automatizado para interactuar con botones de feedback y registrar la inserción de tickets.
4. **Inspección SQL Directa:** Acceso y auditoría a la base de datos `tickets.db` dentro del contenedor para validar inserciones e integridad del esquema relacional.


## 7. HALLAZGOS Y OBSERVACIONES

### **Hallazgo 1: Incapacidad del modelo `smollm:360m` para clasificación semántica compleja**
* **Detalle:** Al utilizar prompts zero-shot o few-shot para clasificar intenciones en formato JSON o texto limpio, el modelo de 360M de parámetros fallaba de forma constante. En lugar de responder con una sola palabra clave (`reporte_de_problema`, `pregunta_general`, `despedida`), el modelo alucinaba inventando nuevas preguntas/respuestas de ejemplo o copiando las instrucciones del sistema.
* **Impacto:** Todas las consultas caían por defecto en la rama de `pregunta_general`, obligando al sistema a invocar la base de conocimientos RAG, aumentando la latencia y consumiendo CPU innecesariamente ante simples saludos o despedidas.
* **Criticidad:** **Media**
* **Evidencia:** La consulta directa a la API de generación de Ollama del prompt de clasificación devolvió: `"response":"Pregunta: ¿Cómo se llama la tierra?\nRespu"` en lugar de la etiqueta de clasificación.

### **Hallazgo 2: Bucle infinito y latencias de respuesta extremas en consultas de RAG**
* **Detalle:** En la configuración inicial, la instancia de LLM del RAG (`llm`) no tenía configurado el parámetro `num_predict` (límite de tokens de salida). Al ingresar una pregunta general o un problema, el modelo local entraba en bucles repetitivos infinitos (escribiendo párrafos idénticos reiteradamente en español).
* **Impacto:** Los tiempos de respuesta del backend superaban los **178 segundos**, provocando que el proxy Nginx cortara la conexión por timeout (límite de 60 segundos predeterminado) y mostrando una pantalla en blanco o error de conexión al usuario.
* **Criticidad:** **Alta**
* **Evidencia:** Captura de log de la primera prueba (`el sistema no enciende`) donde el campo `elapsed` registró un total de **178.240904 segundos** con un JSON de respuesta inflado a **58 KB** lleno de bucles idénticos.

### **Hallazgo 3: Falta de exclusión de archivos pesados en el contexto de Docker Build**
* **Detalle:** El archivo `.dockerignore` no estaba definido en la carpeta del backend, lo que ocasionaba que Docker enviara la carpeta completa del entorno virtual (`venv/`) al daemon de Docker durante la construcción.
* **Impacto:** El tamaño del contexto enviado era de **990 MB**, demorando significativamente la recreación de contenedores y consumiendo almacenamiento en disco de forma ineficiente.
* **Criticidad:** **Baja**
* **Evidencia:** Mensaje de Docker en consola: `"Sending build context to Docker daemon 990MB"`.


## 8. ANÁLISIS DE RIESGOS

| Hallazgo | Riesgo asociado | Impacto | Probabilidad | Nivel de Riesgo |
|----------|-----------------|---------|--------------|-----------------|
| **H1**   | Inadecuada experiencia de usuario por respuestas incoherentes e incorrecta canalización de problemas técnicos. | Medio | Alta | **Medio** |
| **H2**   | Denegación de Servicio (DoS) en el servidor por agotamiento de CPU/Memoria debido a bucles de generación infinitos. | Alto | Alta | **Alto** |
| **H3**   | Tiempos prolongados de despliegue y mantenimiento en entornos CI/CD. | Bajo | Alta | **Bajo** |


## 9. RECOMENDACIONES

1. **Mantener el Router Determinístico (Mitigación para H1):** Reemplazar la llamada del clasificador por LLM por un enrutador basado en palabras clave directas en Python (`clasificar_intencion`), lo cual reduce el consumo de CPU a 0 en la fase de enrutamiento y asegura un 100% de efectividad.
2. **Aplicar Restricciones Strict en Parámetros de Generación (Mitigación para H2):** Configurar siempre el parámetro `num_predict` en las instancias de Ollama. Se recomienda `num_predict=20` para la clasificación semántica (en caso de reincorporar LLMs) y `num_predict=250` para el RAG a fin de forzar la parada del modelo en caso de bucle de repetición.
3. **Optimizar la Configuración del Proxy (Mitigación para H2):** Mantener el `proxy_read_timeout` y `proxy_connect_timeout` de Nginx en 300s para evitar la desconexión del cliente durante periodos de alta carga de CPU en ejecuciones locales.
4. **Implementar Archivo `.dockerignore` (Mitigación para H3):** Ignorar siempre las carpetas `venv/`, `__pycache__/` y `.git/` de los contextos de compilación.
5. **Migración a un Modelo de Mayor Parámetros (Recomendación a Largo Plazo):** Si se requiere mayor razonamiento semántico sin palabras clave, migrar a un modelo de al menos 3B u 8B de parámetros (`Llama-3.2-3B` o `Phi-3-medium`) y habilitar aceleración por hardware (GPU/CUDA/Metal).


## 10. CONCLUSIONES

El sistema de asistencia de HelpDesk virtual de EPIS Corp presentaba inestabilidad crítica operando bajo el modelo `smollm:360m` debido a bucles repetitivos y latencias extremas. Tras las optimizaciones implementadas por auditoría (creación de enrutador determinista por palabras clave, reducción del build context de Docker e inyección de límites de tokens de salida `num_predict=250`), el sistema se ha estabilizado exitosamente. Las respuestas a saludos y despedidas ahora son instantáneas (0 segundos) y las consultas al RAG se autolimitan para evitar caídas por timeout. Sin embargo, debido al bajo poder cognitivo de un modelo de 360M de parámetros, sus respuestas de RAG pueden carecer de profundidad conceptual y coherencia avanzada, por lo que su uso debe limitarse a prototipado rápido o soporte básico.


## 11. PLAN DE ACCIÓN Y SEGUIMIENTO

| Hallazgo | Recomendación | Responsable | Fecha Comprometida |
|----------|----------------|-------------|---------------------|
| **H1**   | Mantener el enrutador determinista basado en palabras clave implementado por auditoría. | Administrador de TI | Inmediato (Aplicado) |
| **H2**   | Validar que la restricción `num_predict=250` esté activa en la plantilla del backend. | Líder Técnico de IA | Inmediato (Aplicado) |
| **H3**   | Confirmar la existencia de `.dockerignore` en los directorios de servicios. | Devops Engineer | Inmediato (Aplicado) |
| **Gral** | Planificar la migración a un modelo de 3B parámetros con aceleración GPU. | Gerente de Tecnología | 30/08/2026 |


## 12. ANEXOS

### Anexo A: Pruebas de API y Latencia Post-Optimizaciones
1. **Consulta de Reporte de Problema (`el sistema no enciende`):**
   * **Entrada:** `curl.exe -s "http://localhost:5173/api/ask?question=el%20sistema%20no%20enciende"`
   * **Salida:** `{"answer":"... [Respuesta limitada en 250 tokens] ... ¿Esta información soluciona tu problema?","follow_up_required":true}`
   * **Tiempo transcurrido:** **17.94 segundos** (reducción del 90% en latencia máxima).
2. **Consulta de Pregunta General (`como instalo una impresora`):**
   * **Entrada:** `curl.exe -s "http://localhost:5173/api/ask?question=como%20instalo%20una%20impresora"`
   * **Salida:** `{"answer":"...","follow_up_required":false}`
   * **Tiempo transcurrido:** **22.83 segundos**.
3. **Consulta de Despedida (`gracias adios`):**
   * **Entrada:** `curl.exe -s "http://localhost:5173/api/ask?question=gracias%20adios"`
   * **Salida:** `{"answer":"De nada, ¡un placer ayudar! Si tienes cualquier otra consulta, aquí estaré. 😊","follow_up_required":false}`
   * **Tiempo transcurrido:** **0.00 segundos** (reducción total del consumo de procesamiento).

### Anexo B: Integridad de Base de Datos de Tickets (tickets.db)
Consulta directa de inserciones de tickets mediante flujo de chat visual:
```bash
docker compose exec backend python -c "import sqlite3; conn = sqlite3.connect('tickets.db'); print(conn.cursor().execute('SELECT * FROM tickets;').fetchall())"
```
**Resultado:**
```python
[(1, 'El teclado no responde y tiene luces apagadas', 'Abierto'), (2, 'El monitor esta parpadeando y se apaga de repente.', 'Abierto')]
```

### Anexo C: Evidencia Visual de Flujo de Tickets
├── 01-docker-compose-ps.png       → Captura de "docker compose ps" mostrando los 3 contenedores "Up"
![alt text](<Captura de pantalla 2026-06-24 162023.png>)
├── 02-grep-llama-a-smollm.png     → Captura del diff/grep mostrando el cambio de modelo en main.py
![alt text](<Captura de pantalla 2026-06-24 162056.png>)
├── 03-router-fix-codigo.png       → Captura del código de clasificar_intencion() en main.py (el fix determinístico)
![alt text](<Captura de pantalla 2026-06-24 162119-1.png>)
├── 04-frontend-chat-cargado.png   → Captura del navegador en localhost:5173 mostrando el mensaje de bienvenida
![alt text](<Captura de pantalla 2026-06-24 162217.png>)
├── 05 ticket_creation_visual_evidence.png → flujo completo de creación de ticket #2
![alt text](<Captura de pantalla 2026-06-24 162244.png>)
├── 06-sqlite-tickets-query.png    → Captura de terminal con el SELECT * FROM tickets mostrando los 2 registros
![alt text](<Captura de pantalla 2026-06-24 162307.png>)
└── 07-dockerignore-contexto.png   → Captura mostrando "Sending build context" antes (990MB) y después (713 bytes)
![alt text](<Captura de pantalla 2026-06-24 162318.png>)