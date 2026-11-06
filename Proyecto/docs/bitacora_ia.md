# Bitácora de uso de IA

**Proyecto:** Analizador de Ventas — TP2, Elementos de Programación IA y LowCode  
**Período documentado:** 27 de octubre al 2 de noviembre de 2026  
**Alcance:** asistencia de IA durante el desarrollo y uso de IA como componente funcional del sistema.

> Esta bitácora conserva el registro original y documenta decisiones y comprobaciones realizadas durante la integración. La IA asistió en tareas técnicas; las configuraciones, pruebas y publicaciones se verificaron en el entorno del proyecto. No deben confundirse las sugerencias de un asistente con resultados efectivamente probados.

## 1. Registro de utilización de herramientas

| Fecha | Herramienta | Para qué se usó | Decisión / corrección |
|---|---|---|---|
| 2026-10-27 | Claude | Pasar el TP1 a arquitectura TP2 (FastAPI, Streamlit, Docker) | Se mantuvo JSON como persistencia y se agregó `id` por venta. *Registro conservado del documento original.* |
| 2026-11-01 a 2026-11-02 | ChatGPT | Orientación para integrar Ubuntu, Docker Compose, n8n y FastAPI; configurar conexiones HTTP y depurar workflows | n8n se ejecuta en Docker; FastAPI y Streamlit se ejecutan como procesos Python en Ubuntu. Para que n8n acceda a FastAPI se utiliza `http://host.docker.internal:8000`, habilitado por `extra_hosts` en Compose. |
| 2026-11-01 a 2026-11-02 | ChatGPT | Asistencia en la construcción de **WF01 — Detección de Anomalías** | Se consulta `GET /anomalias`, se evalúa `cantidad > 0` y, si corresponde, se registra una alerta en la Data Table `alertas_anomalias`. |
| 2026-11-01 a 2026-11-02 | ChatGPT y Google Gemini (mediante n8n) | Configuración y prueba de **WF02 — Informe Inteligente de Ventas** | Se consulta `GET /indicadores`, se valida `cantidad_operaciones > 0` y se utiliza un AI Agent con Gemini. El agente dispone de una herramienta HTTP para consultar `GET /anomalias`. |
| 2026-11-01 a 2026-11-02 | ChatGPT | Diseño de **WF03 — Integración Streamlit IA** y del botón **Generar informe con IA** | WF03 recibe `POST /webhook/informe-ventas`, invoca WF02 como subworkflow y devuelve el informe a Streamlit. Se decidió reutilizar WF02, no duplicar el agente. |
| 2026-11-02 | ChatGPT | Revisión de requisitos y mejora de WF03 | Se incorporó un IF que valida `body.accion == "generar_informe"`. Solicitudes válidas ejecutan WF02 y retornan el informe; las inválidas retornan HTTP 400 y un JSON de error. |
| 2026-11-02 | ChatGPT | Asistencia en documentación y organización de los archivos versionados | Se actualizaron el README principal, la documentación de arquitectura y los JSON exportados. Se verificaron las rutas del repositorio y se excluyeron del commit los datos modificados durante pruebas. |
| 2026-11-06 | ChatGPT | Verificación del entorno y revisión final del proyecto | Se comprobó Docker con n8n activo, FastAPI mediante /health, acceso a Swagger UI y Streamlit, e integración con Gemini. Se actualizó GitHub y se eliminó test_api.py por contener pruebas dependientes del dataset original. |
## 2. Decisiones técnicas principales

1. **Mantener cálculos y validación en Python:** FastAPI, Pydantic y `logica.py` gestionan operaciones, indicadores y detección de anomalías. La IA no reemplaza estos cálculos ni modifica `data/ventas.json`.
2. **Usar Gemini desde un AI Agent de n8n:** se optó por una API externa en lugar de desplegar un modelo local. La clave se configura en las credenciales de n8n y no se publica en Git.
3. **Separar responsabilidades en tres workflows:** WF01 registra anomalías; WF02 elabora informes; WF03 expone el webhook, valida las solicitudes y delega la generación del informe a WF02.
4. **Usar Docker para n8n, sin contenedorización total:** el `compose.yaml` inicia n8n; FastAPI y Streamlit permanecen ejecutándose directamente en Ubuntu. Esta elección satisface la integración utilizada en el laboratorio, sin afirmar que todos los servicios se desplieguen mediante Compose.
5. **Controlar explícitamente los errores de entrada:** Streamlit envía `{"accion":"generar_informe"}`; WF03 rechaza solicitudes sin esa acción mediante HTTP 400.
6. **Versionar de manera selectiva:** se publicaron cambios en la rama `RamaDeCarlos`, incluyendo los JSON de workflows. Se evitó incorporar por accidente modificaciones locales de `Proyecto/data/ventas.json`.

## 3. Dificultades y correcciones

| Dificultad observada | Análisis y corrección | Cómo se comprobó |
|---|---|---|
| n8n, dentro de Docker, debía consultar la API ejecutada en Ubuntu | Se configuró `extra_hosts: host.docker.internal:host-gateway` y se usó la URL `http://host.docker.internal:8000` en los nodos HTTP de n8n. | Los workflows obtuvieron respuestas de `/indicadores` y `/anomalias`. |
| La configuración inicial de Gemini no produjo una respuesta utilizable | Se revisó la disponibilidad del modelo y se configuró uno disponible para la credencial utilizada. | WF02 generó un informe comercial real a partir de los indicadores de ventas. |
| El webhook de prueba no ejecutaba todo WF03 | Se ejecutó el workflow completo desde el editor y se distinguió la URL de prueba (`/webhook-test/`) de la de producción (`/webhook/`). | La llamada de prueba devolvió el resultado del subworkflow; posteriormente funcionó la URL de producción con WF03 publicado. |
| WF03 no podía publicarse porque dependía de WF02 | Se publicó primero WF02 y después WF03. | Ambos quedaron publicados y se obtuvo una respuesta desde Streamlit. |
| El primer WF03 no contenía una decisión propia | Se insertó un nodo IF entre Webhook y Execute Sub-workflow, con ramas True/False y respuestas HTTP diferenciadas. | Una solicitud `{}` devolvió HTTP 400 y `{"error":"Solicitud inválida: accion debe ser generar_informe"}`. Una solicitud válida generó el informe en Streamlit. |
| Los JSON exportados de n8n tenían nombres con espacios y no coincidían con los archivos rastreados | Se renombraron los archivos con los nombres utilizados en `Proyecto/n8n/workflows/` y se validó su sintaxis. | `python3 -m json.tool` confirmó JSON válido; Git detectó la actualización de WF02 y la incorporación de WF03. |
| El README actualizado se copió inicialmente dentro de `Proyecto/` en lugar de la raíz del repositorio | Se reemplazó el README principal y se eliminó la copia duplicada. | Se verificaron rutas relativas y cambios con `git status` y `git diff --cached --stat`. |
| El informe generado presentó pequeños defectos de formato Markdown | Se identificó como una limitación de presentación del texto generado por la IA; no afecta las cifras calculadas por FastAPI ni la comunicación del sistema. | El informe se visualizó en Streamlit; queda como mejora de presentación, sin afirmar que esté corregido. |

## 4. Verificaciones realizadas

- **API REST:** se realizaron pruebas automatizadas durante el desarrollo. Posteriormente se eliminó `test_api.py` porque sus verificaciones dependían de un dataset original de 14 ventas. La aplicación utiliza actualmente 18 registros de ejemplo. Se mantienen las verificaciones funcionales y de integración.
- **WF01:** ejecución manual de detección de anomalías y registro de una alerta en la Data Table de n8n. El Schedule Trigger fue configurado; **su ejecución automática no está acreditada por esa prueba manual**.
- **WF02:** generación de un informe comercial con Gemini, tomando indicadores reales devueltos por FastAPI y con la herramienta de anomalías configurada.
- **WF03, entrada inválida:** `POST /webhook/informe-ventas` con `{}` retornó **HTTP 400** y un cuerpo JSON con el campo `error`.
- **WF03, entrada válida:** Streamlit envió `{"accion":"generar_informe"}` y presentó el informe de Gemini. La interfaz recibió y mostró el resultado de la integración.
- **Git:** se verificaron los JSON con `python3 -m json.tool` y se publicaron commits en `RamaDeCarlos`, preservando fuera del commit las ventas modificadas en pruebas locales.

**Alcance de la evidencia:** las pruebas integradas de n8n y Streamlit descritas son **manuales**. No se desarrolló una suite automatizada de pruebas extremo a extremo para los tres workflows.

## 5. Criterios de revisión humana y seguridad

- Las respuestas generadas por Gemini son **interpretaciones y recomendaciones**, no fuentes autoritativas de ingresos, totales ni anomalías; deben contrastarse con `/indicadores` y `/anomalias`.
- Las propuestas de ChatGPT se aplicaron de forma incremental, revisando código, nodos, errores HTTP, salida de consola y estados de Git antes de publicar los cambios.
- Las credenciales de Gemini se almacenan en el gestor de credenciales de n8n y no deben aparecer en documentos, commits ni capturas compartidas. Los JSON exportados deben revisarse antes de subirlos.
- La importación de workflows en una instalación diferente requiere volver a asociar credenciales, Data Tables y referencias entre workflows; que un JSON sea sintácticamente válido no garantiza que funcione sin esa configuración.
- La solución es un **prototipo académico**: el webhook de laboratorio no implementa autenticación y no debe exponerse públicamente sin controles adicionales.

## 6. Pendientes o mejoras no acreditadas como completadas

- Confirmar y documentar la **activación y ejecución automática** del Schedule Trigger de WF01, si se desea utilizarlo en la demostración.
- Mejorar la presentación Markdown de algunos informes generados por Gemini.
- Verificar el procedimiento de importación en una instancia limpia de n8n y documentar la reasociación de credenciales y subworkflows si corresponde.
- Completar la asignación de responsabilidades de cada integrante en `Proyecto/docs/responsabilidades.md` si aún contiene valores de ejemplo.

**Referencias internas:** [`README.md`](../../README.md), [`arquitectura.md`](arquitectura.md), [`workflows n8n`](../n8n/workflows/)