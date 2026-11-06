# WF01 - Detección de anomalías

## Objetivo
Detectar anomalías de ventas mediante FastAPI y registrar alertas en n8n.

## Componentes
- Manual Trigger: ejecución de pruebas.
- Schedule Trigger: ejecución diaria a las 09:00.
- HTTP Request: GET http://host.docker.internal:8000/anomalias
- IF: verifica si cantidad > 0.
- Edit Fields: genera el mensaje de alerta.
- Data Table: almacena el resultado.

## Configuración previa

Crear en n8n una Data Table llamada alertas_anomalias
con las siguientes columnas:

- fecha: String
- cantidad: Number
- mensaje_alerta: String

Después de importar el workflow, seleccionar esta tabla
en el nodo Insert row.

## Requisitos
- FastAPI ejecutándose en el puerto 8000.
- n8n ejecutándose mediante Docker Compose.
- FastAPI accesible desde el contenedor n8n.

## Estado
Probado manualmente con 7 anomalías detectadas.
Persistencia de una alerta verificada en Data Table.

## Pendiente
Publicar y verificar la ejecución programada.
