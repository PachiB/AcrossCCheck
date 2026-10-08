# 2. Flujo de Power Automate — `ConflictCheck - Procesar nueva solicitud`

Este es el corazón del proceso: en cuanto se crea una solicitud, reparte una
**aprobación nativa de Teams** (app "Approvals", funciona perfecto en móvil) a
cada miembro activo del grupo, registra cada respuesta en tiempo real, avisa
si alguien marca "Potencial conflicto", y cierra la solicitud cuando todos
han contestado.

Créalo en [make.powerautomate.com](https://make.powerautomate.com) → **Flujos de nube** → **+ Nuevo flujo** → **Automatizado**.

> Requisito previo: haber creado las 4 listas de SharePoint (carpeta `01-SharePoint-Listas`).

---

## Disparador

**SharePoint — "Cuando se crea un elemento"**
- Dirección del sitio: `https://acrosslegal.sharepoint.com/sites/Socios-ConflictCheck`
- Nombre de lista: `ConflictCheck_Solicitudes`

## Paso 1 — Obtener el grupo de revisores
**SharePoint — "Obtener elementos"**
- Lista: `ConflictCheck_Miembros`
- Consulta de filtro: `Activo eq 1`
- Ordenar por: `Orden asc`

## Paso 2 — Guardar el nº de miembros
**Variables — "Inicializar variable"**
- Nombre: `NumMiembros`, Tipo: Entero
- Valor: `length(outputs('Obtener_elementos')?['body/value'])`

## Paso 3 — Marcar la solicitud "En curso"
**SharePoint — "Actualizar elemento"**
- Lista: `ConflictCheck_Solicitudes`
- Id: `triggerOutputs()?['body/ID']`
- Estado: `En curso`
- TotalMiembros: `variables('NumMiembros')`
- NumRespondidos: `0`
- NumConflictos: `0`

## Paso 4 — Por cada miembro, en paralelo

**Aplicar a cada uno** sobre `value` del paso "Obtener elementos" (Miembros).
Abre **Ajustes** del "Aplicar a cada uno" → **Control de simultaneidad: Activado** →
**Grado de paralelismo: 20** (así los 12 reciben el aviso a la vez, no en cola).

Dentro del bucle, en este orden:

### 4.1 Crear la fila de seguimiento (estado "Pendiente")
**SharePoint — "Crear elemento"**
- Lista: `ConflictCheck_Respuestas`
- Title: `concat(triggerBody()?['Title'], '-', item()?['Email'])`
- SolicitudId: `triggerOutputs()?['body/ID']`
- SolicitudTitle: `triggerBody()?['Title']`
- MiembroEmail: `item()?['Email']`
- MiembroNombre: `item()?['Title']`
- Respuesta: `Pendiente`

Guarda el ID de este elemento (se usará en el paso 4.3): lo da
`outputs('Crear_elemento')?['body/ID']`.

### 4.2 Pedir la respuesta vía Approvals de Teams
**Approvals (Teams/Office 365) — "Iniciar y esperar una aprobación"**
- Tipo de aprobación: **Custom Responses - Wait for one response**
- Título: `concat('Conflict Check: ', triggerBody()?['Cliente'], ' — ', triggerBody()?['Asunto'])`
- Asignado a: `item()?['Email']`
- Detalles:
  ```
  concat(
    'Cliente: ', triggerBody()?['Cliente'], '
  ',
    'Asunto: ', triggerBody()?['Asunto'], '
  ',
    'Contraparte / otras partes: ', triggerBody()?['Contraparte'], '
  ',
    'Comentarios: ', coalesce(triggerBody()?['Comentarios'], '(ninguno)'), '
  ',
    'Urgente: ', if(equals(triggerBody()?['Urgente'], true), 'SÍ ⚠️', 'No'),
    '

Responde "Potencial conflicto" si reconoces alguna de las partes, y añade un comentario.'
  )
  ```
- Opciones de respuesta personalizadas: **`No hay conflicto`** y **`Potencial conflicto`** (exactamente estos dos textos, en este orden)
- Habilitar comentarios: **Opcional** (así no bloquea a quien responde "No hay conflicto", pero el que marca "Potencial conflicto" puede explicarse)
- Vínculo del elemento (opcional pero recomendado): enlaza a la solicitud en la Power App, p.ej. `concat('https://teams.microsoft.com/l/entity/<APP_ID>/detalle?context=', triggerOutputs()?['body/ID'])` — rellena `<APP_ID>` cuando publiques la app (paso 4 del README maestro). Si no lo tienes aún, déjalo vacío por ahora.

Esta acción se queda "esperando" solo dentro de su propia rama del bucle — como el bucle es paralelo, las 12 esperas corren a la vez, cada una se resuelve en cuanto esa persona contesta desde el móvil.

### 4.3 Guardar la respuesta en cuanto llega
**SharePoint — "Actualizar elemento"**
- Lista: `ConflictCheck_Respuestas`
- Id: `outputs('Crear_elemento')?['body/ID']` (el guardado en 4.1)
- Respuesta: contenido dinámico **"Response"** de la acción de aprobación del paso 4.2
- Comentario: contenido dinámico **"Response Summary" / "Comments"** de la misma acción
- FechaRespuesta: `utcNow()`

### 4.4 Avisar inmediatamente si hay potencial conflicto
**Condición**: `Respuesta` (dinámico del paso 4.2) **es igual a** `Potencial conflicto`

**Si es cierto → Teams — "Publicar un mensaje en un chat o canal"**
- Publicar como: Flow bot
- Publicar en: Canal — elige el canal "Conflict Check · Alertas" (o el chat/canal que prefieras)
- Mensaje:
  ```
  ⚠️ Potencial conflicto — concat(triggerBody()?['Cliente'], ' / ', triggerBody()?['Asunto'])
  Señalado por: item()?['Title']
  Comentario: [Comments del paso 4.2]
  ```

> Nota: si prefieres avisar por chat 1:1 al solicitante en vez de (o además de) un canal, usa "Publicar un mensaje en un chat" con destinatario `triggerBody()?['SolicitanteEmail']`.

---

## Paso 5 — Cuando todos han contestado (fuera del "Aplicar a cada uno")

El propio conector "Aplicar a cada uno" no continúa hasta que **todas** las ramas paralelas han terminado, así que este bloque solo se ejecuta cuando los 12 ya respondieron.

### 5.1 Recontar respuestas (evita problemas de concurrencia)
**SharePoint — "Obtener elementos"**
- Lista: `ConflictCheck_Respuestas`
- Consulta de filtro: `SolicitudId eq [triggerOutputs()?['body/ID']]`

### 5.2 Calcular el resultado
**Variables — "Inicializar variable"** `NumConflictos` (Entero) =
`length(filterArray(outputs('Obtener_elementos_respuestas')?['body/value'], 'Respuesta', 'Potencial conflicto'))`

*(Usa la acción de datos "Filtro de matriz" como paso previo si prefieres no escribir la expresión a mano: Filtra `value` donde `Respuesta` es igual a `Potencial conflicto`, y luego `length()` sobre su salida.)*

### 5.3 Cerrar la solicitud
**SharePoint — "Actualizar elemento"**
- Lista: `ConflictCheck_Solicitudes`, Id: `triggerOutputs()?['body/ID']`
- NumRespondidos: `variables('NumMiembros')`
- NumConflictos: `variables('NumConflictos')`
- Estado: `if(greater(variables('NumConflictos'), 0), 'Cerrado - Potencial conflicto', 'Cerrado - Sin conflicto')`

### 5.4 Notificar al solicitante
**Teams — "Publicar un mensaje en un chat"**
- Destinatario: `triggerBody()?['SolicitanteEmail']`
- Mensaje: resumen con cliente/asunto, estado final y, si hay conflictos, cuántos y quién los señaló (puedes construir la lista con una "Aplicar a cada uno" sobre las respuestas con `Respuesta = Potencial conflicto`).

### 5.5 Notificar a los administradores
Igual que 5.4 pero en bucle sobre la lista `ConflictCheck_Admins` (o, más simple, publica una vez en el canal de alertas con el resumen).

---

## Resumen de por qué funciona para "agilidad desde el móvil"

- Las aprobaciones de Teams aparecen en la pestaña nativa **Approvals**, incluida la app móvil de Teams — no hay que instalar nada ni entrar en la Power App para contestar.
- Responder es **un toque** sobre "No hay conflicto" o "Potencial conflicto"; si eligen la segunda, Teams les pide el comentario en el mismo gesto.
- Cada respuesta se escribe al instante en SharePoint, así que el solicitante y los administradores ven el progreso en vivo en la Power App (siguiente carpeta) sin esperar a que contesten los 12.

## Mejora futura opcional (no incluida en esta primera versión)
Un segundo flujo programado (p.ej. cada 4 horas) que repase `ConflictCheck_Respuestas`
con `Respuesta = Pendiente` y reenvíe un recordatorio a quien no ha contestado en
X horas. Se monta igual que este, cambiando el disparador por "Recurrencia" y
añadiendo la condición de antigüedad.
