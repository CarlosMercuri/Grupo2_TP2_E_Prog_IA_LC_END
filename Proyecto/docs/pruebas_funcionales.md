# Pruebas funcionales — Analizador de Ventas (TP2)

**Materia:** Elementos de Programación IA y LowCode
**Proyecto:** Analizador de Ventas — Grupo 2
**Fecha de ejecución:** 6 de noviembre de 2026 (hora de Argentina; algunas respuestas HTTP muestran el 7 de noviembre en UTC).
**Modalidad:** pruebas manuales desde Streamlit, Swagger UI, terminal y n8n.
**Entorno:** Ubuntu en VirtualBox; FastAPI/Uvicorn en el puerto 8000; Streamlit en el 8501; n8n en Docker en el 5678; Google Gemini mediante el AI Agent de n8n.

## 1. Objetivo y preparación

Verificar las operaciones de ventas, la validación de entradas, los indicadores, la detección de anomalías, la importación parcial de CSV y la integración Streamlit → n8n → Gemini.

Antes de modificar los datos se respaldaron los archivos de trabajo:

- `data/ventas.json`
- `data/ventas_nuevas.csv`

**Ubicación del respaldo:** `/home/carlos/respaldos_tp2/20261106_224538/` (ruta del entorno de pruebas, no incluida en Git).

Se partió de **18 ventas**. Durante la evaluación se crearon las ventas 19 y 20, se actualizó y eliminó la venta 20 y, finalmente, se importaron tres registros válidos. Por eso el informe final presenta **22 operaciones**.

## 2. Casos ejecutados y resultados

| ID | Prueba | Herramienta / operación | Resultado observado | Estado |
|---|---|---|---|---|
| PF-01 | Alta de venta | Streamlit → `POST /ventas` | «Monitor Samsung», precio $250.000, cantidad 2; ID **19**, total **$500.000**. | **APROBADA** |
| PF-02 | Persistencia de alta | `GET /ventas/19` con `curl` y revisión de JSON | HTTP **200** y venta 19 recuperada con total $500.000. | **APROBADA** |
| PF-03 | Rechazo por precio negativo | `POST /ventas` con precio `-500` | HTTP **422**; Pydantic informó `Input should be greater than 0`. | **APROBADA** |
| PF-04 | No persistencia de venta rechazada | `GET /ventas?producto=Monitor%20Prueba` | Respuesta `[]`; el registro inválido no apareció. | **APROBADA** |
| PF-05 | Alta desde documentación interactiva | Swagger UI → `POST /ventas` | «Teclado Logitech», $45.000 × 2; HTTP **201**, ID **20**, total **$90.000**. | **APROBADA** |
| PF-06 | Consulta de venta por ID | Swagger UI → `GET /ventas/19` | HTTP **200**; datos correctos de «Monitor Samsung». | **APROBADA** |
| PF-07 | Modificación de venta | Swagger UI → `PUT /ventas/20` | HTTP **200**; precio $50.000, cantidad 3, total **$150.000**; mantiene ID 20. | **APROBADA** |
| PF-08 | Eliminación de venta | Swagger UI → `DELETE /ventas/20` | HTTP **200**, mensaje `Venta eliminada.` | **APROBADA** |
| PF-09 | Consulta de registro eliminado | Swagger UI → `GET /ventas/20` | HTTP **404**, mensaje `No existe la venta con id=20.` | **APROBADA** |
| PF-10 | Indicadores comerciales | Swagger UI → `GET /indicadores` | HTTP **200**; **19** operaciones, **$38.188.900** de ingresos y promedio **$2.009.942,11**; producto top «Remera Lisa» y categoría líder «Hogar». | **APROBADA** |
| PF-11 | Detección de anomalías | Swagger UI → `GET /anomalias` | HTTP **200**; se informaron duplicados y valores atípicos, entre ellos ventas de «Televisor» y «Notebook». | **APROBADA** |
| PF-12 | Importación parcial de CSV | Streamlit → «Importar CSV» | Archivo `ventas_prueba.csv` (mismo contenido que `ventas_nuevas.csv`): **3 aceptadas de 4**, **1 rechazada** por precio `-100`. | **APROBADA** |
| PF-13 | Informe comercial con IA | Streamlit → WF03 → WF02 → Gemini | Se generó y mostró el informe; incluyó **22** operaciones, ingresos **$38.368.300** y promedio **$1.744.013,64**. | **APROBADA** |
| PF-14 | Registro de alertas en n8n | WF01 → Data Table `alertas_anomalias` | Nodo **Insert row** finalizó correctamente; nueva fila con **8 anomalías** y mensaje de alerta. | **APROBADA** |
| PF-15 | Ejecución automática programada | WF01 con Schedule Trigger temporal cada minuto | Ejecuciones exitosas registradas a las **12:55:07** y **12:56:07** del 06/11. Después se restauró la programación diaria a las 09:00 y se publicó. | **APROBADA** (frecuencia temporal) |

### Detalle de la importación CSV (PF-12)

| Producto | Precio | Cantidad | Importe | Resultado |
|---|---:|---:|---:|---|
| Zapatillas Urbanas | $32.000 | 3 | $96.000 | Aceptada |
| Billetera | $9.000 | 6 | $54.000 | Aceptada |
| Vela Aromática | $4.200 | 7 | $29.400 | Aceptada |
| Producto Inválido | -$100 | 2 | — | Rechazada: precio debe ser mayor que 0 |

**Importe agregado por la importación:** $179.400. Se verificó consistencia entre los ingresos antes del CSV ($38.188.900) y el informe posterior ($38.368.300).

### Prueba de webhook realizada anteriormente durante la integración

La bitácora técnica del proyecto también registra una prueba de **WF03**: solicitud `POST /webhook/informe-ventas` con `{}` → **HTTP 400** y JSON de error; con `{"accion":"generar_informe"}` → generación de informe mediante WF02. Esta verificación corresponde a una sesión anterior y **no se repitió durante la presente ronda funcional**.

## 3. Hallazgos y consideraciones

1. **Normalización de categorías:** inicialmente aparecían `Informatica` e `Informática` como categorías separadas. Se corrigió directamente el archivo JSON durante las pruebas y `/indicadores` pasó a mostrar **Informática: $6.500.000**. Como mejora futura, conviene normalizar los nombres al ingresar datos desde la API; la edición directa del JSON no es el procedimiento operativo recomendado.
2. **Anomalía no implica error:** el detector marca ventas duplicadas por igualdad de producto, categoría, precio y cantidad; también marca totales elevados mediante **Q3 + 3 × IQR**. Los hallazgos necesitan revisión humana. La prueba final de WF01 registró **8** anomalías.
3. **Limitaciones del informe de IA:** el modelo genera interpretaciones y recomendaciones, pero no sustituye el cálculo de indicadores realizado en Python; las recomendaciones deben revisarse frente a ventas atípicas.
4. **Alcance:** se realizaron pruebas funcionales manuales; **no hay una suite de pruebas automatizadas vigente** (`test_api.py` fue eliminado). Tampoco se validó una reinstalación completa en una instancia limpia de n8n.
5. **Programación diaria de WF01:** se comprobó la ejecución automática con frecuencia temporal de un minuto. Se restauró la programación diaria a las 09:00 y se publicó; **no se documenta aquí una ejecución del siguiente día a las 09:00**.

## 4. Conclusión y pasos de cierre

**Resultado general: 15 casos funcionales aprobados según las salidas y capturas observadas**, con las salvedades señaladas. Se demostró la comunicación entre Streamlit, FastAPI, Pydantic, n8n y Gemini, así como el manejo de errores y la persistencia JSON.

Antes de la entrega definitiva:

- Decidir si restaurar `ventas.json` desde el respaldo o conservar el conjunto ampliado; documentar la decisión.
- Exportar de n8n los JSON definitivos de los workflows, especialmente WF01 con la programación diaria publicada.
- Incorporar este informe y las novedades de la bitácora mediante un commit y un Pull Request hacia `main`.
- Conservar capturas de Swagger UI, Streamlit y n8n como evidencia adicional si el equipo decide adjuntarlas al repositorio.

**Nota de evidencia:** los códigos, respuestas y cantidades consignados provienen de los resultados mostrados durante las pruebas manuales. No se generó un log automatizado ni se incorporaron capturas al presente archivo.
