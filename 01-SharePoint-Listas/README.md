# 1. Listas de SharePoint

Todo el proceso se apoya en 4 listas dentro del sitio:
`https://acrosslegal.sharepoint.com/sites/Socios-ConflictCheck`

Cada lista se crea en **2 minutos** importando el Excel correspondiente:

1. En el sitio SharePoint → **Contenido del sitio** → **+ Nuevo** → **Lista** → **Desde Excel**.
2. Sube el archivo indicado abajo.
3. SharePoint crea la lista y las columnas automáticamente a partir de la cabecera y la fila de ejemplo.
4. Ajusta los tipos de columna según la tabla de cada lista (algunos tipos especiales como "Persona" o "Elección" no se pueden inferir de Excel y hay que cambiarlos a mano una vez creada la lista: columna → **Editar columna** → cambiar **Tipo**).
5. Borra la fila de ejemplo una vez verificada la importación.

Importa las listas **en este orden** (Miembros y Admins no dependen de nada; Solicitudes y Respuestas se pueden crear en cualquier momento, pero conviene que Respuestas exista antes de montar el flujo).

---

## 1.1 `ConflictCheck_Miembros`
Archivo: `Plantilla_ConflictCheck_Miembros.xlsx`

El grupo de 12 (o los que sean) revisores de conflicto. **Edita esta lista para añadir/quitar gente sin tocar el flujo.**

| Columna | Tipo final | Notas |
|---|---|---|
| Title | Una línea de texto | Nombre y apellidos |
| Email | Una línea de texto | Correo corporativo exacto (se usa para asignar la aprobación en Teams) |
| Activo | Sí/No | Solo los marcados "Sí" reciben la solicitud |
| Orden | Número | Opcional, para ordenar el panel de seguimiento |

## 1.2 `ConflictCheck_Admins`
Archivo: `Plantilla_ConflictCheck_Admins.xlsx`

Quién puede ver **todas** las solicitudes (no solo las propias) en la Power App.

| Columna | Tipo final | Notas |
|---|---|---|
| Title | Una línea de texto | Nombre |
| Email | Una línea de texto | Correo corporativo exacto |

## 1.3 `ConflictCheck_Solicitudes`
Archivo: `Plantilla_ConflictCheck_Solicitudes.xlsx`

Una fila = un conflict check pedido.

| Columna | Tipo final | Notas |
|---|---|---|
| Title | Una línea de texto | Código legible, p.ej. `CC-0001` (se puede generar automáticamente en la Power App) |
| Cliente | Una línea de texto | Nombre del cliente |
| Asunto | Varias líneas de texto | Descripción del asunto |
| Contraparte | Varias líneas de texto | Contraparte / otras partes / inversores |
| Comentarios | Varias líneas de texto | Opcional |
| Urgente | Sí/No | |
| SolicitanteEmail | Una línea de texto | Correo de quien pide el check |
| SolicitanteNombre | Una línea de texto | Nombre de quien pide el check |
| FechaSolicitud | Fecha y hora | |
| Estado | Elección: `En curso`, `Cerrado - Sin conflicto`, `Cerrado - Potencial conflicto` | La actualiza el flujo |
| TotalMiembros | Número | Nº de miembros activos a los que se envió (lo rellena el flujo al crear) |
| NumRespondidos | Número | Lo va incrementando el flujo |
| NumConflictos | Número | Nº de "Potencial conflicto" recibidos, lo va incrementando el flujo |

> El **ID** numérico interno de SharePoint (columna `ID`, automática) es el que se usa como `SolicitudId` para enlazar con la lista de Respuestas — no hay que crearlo, ya existe en toda lista de SharePoint.

## 1.4 `ConflictCheck_Respuestas`
Archivo: `Plantilla_ConflictCheck_Respuestas.xlsx`

Una fila = la respuesta de **una persona** a **una solicitud**. Esto es lo que permite el seguimiento en tiempo real ("8 de 12 han contestado").

| Columna | Tipo final | Notas |
|---|---|---|
| Title | Una línea de texto | `CC-0001-resp-1` o similar, informativo |
| SolicitudId | Número | ID de la lista Solicitudes a la que pertenece |
| SolicitudTitle | Una línea de texto | Código de la solicitud, para filtrar rápido en la app |
| MiembroEmail | Una línea de texto | Quién contesta |
| MiembroNombre | Una línea de texto | |
| Respuesta | Elección: `Pendiente`, `No hay conflicto`, `Potencial conflicto` | |
| Comentario | Varias líneas de texto | Obligatorio en la práctica si hay potencial conflicto |
| FechaRespuesta | Fecha y hora | |

> **Nota de diseño:** el flujo crea automáticamente una fila por cada miembro activo en el momento en que se crea la solicitud, con `Respuesta = Pendiente`, y luego la actualiza (no crea una nueva) cuando esa persona contesta. Así la Power App puede pintar siempre los 12 nombres con su estado, incluidos los que aún no han contestado.
