# 3. App de Power Apps — `Conflict Check`

Esta app cubre: **crear la ficha del caso**, **ver el seguimiento en vivo**
("X de 12 han contestado", quién y qué contestó) y el **panel de administración**
con todas las solicitudes de todo el mundo.

No hace falta construirla desde cero: Power Apps genera las 3 pantallas base
automáticamente a partir de la lista `ConflictCheck_Solicitudes`, y aquí abajo
tienes las fórmulas exactas para personalizarla.

## Paso 0 — Generar la app base
1. Ve a la lista `ConflictCheck_Solicitudes` en SharePoint.
2. Cinta superior → **Integrar** → **Power Apps** → **Crear una aplicación**.
3. Nómbrala **Conflict Check**. Power Apps crea 3 pantallas: `BrowseScreen1`, `DetailScreen1`, `EditScreen1`.
4. En **Archivo → Orígenes de datos**, añade también: `ConflictCheck_Respuestas`, `ConflictCheck_Miembros`, `ConflictCheck_Admins`.

## App.OnStart (App → propiedad `OnStart`)
```
Set(varIsAdmin, !IsBlank(LookUp(ConflictCheck_Admins, Lower(Email) = Lower(User().Email))));
Set(varUserEmail, User().Email);
Set(varVerTodas, false)
```
(Ejecútalo una vez: botón **≡** arriba a la izquierda → "Ejecutar OnStart" después de pegarlo.)

---

## Pantalla 1 — `BrowseScreen1` → renómbrala **"Mis Solicitudes"**

Añade un control **Toggle** llamado `toggleVerTodas` con:
- `Visible`: `varIsAdmin`
- `OnCheck`: `Set(varVerTodas, true)`
- `OnUncheck`: `Set(varVerTodas, false)`
- Etiqueta al lado: "Ver todas (admin)"

Añade un botón **"+ Nueva solicitud"**:
- `OnSelect`: `NewForm(FormNuevaSolicitud); Navigate(ScreenNuevaSolicitud, ScreenTransition.Cover)`
  *(este formulario se crea en la Pantalla 3, más abajo)*

**Galería principal → propiedad `Items`:**
```
SortByColumns(
  If(
    varIsAdmin && varVerTodas,
    ConflictCheck_Solicitudes,
    Filter(ConflictCheck_Solicitudes, Lower(SolicitanteEmail) = Lower(varUserEmail))
  ),
  "FechaSolicitud",
  Descending
)
```

**Dentro de cada tarjeta de la galería:**
- Título: `ThisItem.Cliente`
- Subtítulo: `ThisItem.Asunto`
- Etiqueta "urgente" (Visible = `ThisItem.Urgente`): texto `"URGENTE"`, color rojo
- Etiqueta de progreso:
  ```
  CountRows(Filter(ConflictCheck_Respuestas, SolicitudId = ThisItem.ID, Respuesta <> "Pendiente"))
  & " / " & ThisItem.TotalMiembros & " han contestado"
  ```
- Etiqueta de estado: `ThisItem.Estado`, con `Color`:
  ```
  Switch(ThisItem.Estado,
    "Cerrado - Potencial conflicto", Color.Red,
    "Cerrado - Sin conflicto", Color.Green,
    Color.DarkOrange
  )
  ```
- `OnSelect` de la tarjeta (o de toda la galería): `Set(varSolicitudSel, ThisItem); Navigate(ScreenDetalle, ScreenTransition.Cover)`

---

## Pantalla 2 — `DetailScreen1` → renómbrala **"Detalle"**

Borra el control de formulario de detalle por defecto y sustitúyelo por estas etiquetas/galería (más control sobre el diseño):

- Título: `varSolicitudSel.Cliente & " — " & varSolicitudSel.Asunto`
- Datos: `"Contraparte: " & varSolicitudSel.Contraparte`
- Datos: `"Comentarios: " & varSolicitudSel.Comentarios`
- Datos: `"Solicitante: " & varSolicitudSel.SolicitanteNombre & " — " & Text(varSolicitudSel.FechaSolicitud, "dd/mm/yyyy hh:mm")`
- Progreso grande:
  ```
  CountRows(Filter(ConflictCheck_Respuestas, SolicitudId = varSolicitudSel.ID, Respuesta <> "Pendiente"))
  & " de " & varSolicitudSel.TotalMiembros & " han contestado"
  ```
- Banner de alerta (Visible):
  ```
  CountRows(Filter(ConflictCheck_Respuestas, SolicitudId = varSolicitudSel.ID, Respuesta = "Potencial conflicto")) > 0
  ```
  Texto: `"⚠️ Potencial conflicto señalado"`, fondo rojo claro.

**Galería de miembros → propiedad `Items`:**
```
SortByColumns(
  Filter(ConflictCheck_Respuestas, SolicitudId = varSolicitudSel.ID),
  "MiembroNombre",
  Ascending
)
```
**Dentro de cada fila:**
- `ThisItem.MiembroNombre`
- Insignia de estado (`ThisItem.Respuesta`), color:
  ```
  Switch(ThisItem.Respuesta,
    "No hay conflicto", Color.Green,
    "Potencial conflicto", Color.Red,
    Color.Gray /* Pendiente */
  )
  ```
- `ThisItem.Comentario` (envolver texto, solo relevante si hay conflicto)

Botón "Volver": `OnSelect`: `Navigate(ScreenMisSolicitudes, ScreenTransition.Cover)`

---

## Pantalla 3 — `EditScreen1` → renómbrala **"Nueva Solicitud"**, formulario `FormNuevaSolicitud`

Deja visibles solo estas tarjetas de datos (quita el resto con "Quitar campos" / Eliminar tarjeta):
`Cliente`, `Asunto`, `Contraparte`, `Comentarios`, `Urgente`.

Para los campos que el usuario NO rellena a mano, en cada `DataCard` oculto pon su propiedad `Update`:
- `DataCardValue_SolicitanteEmail.Update`: `User().Email`
- `DataCardValue_SolicitanteNombre.Update`: `User().FullName`
- `DataCardValue_FechaSolicitud.Update`: `Now()`
- `DataCardValue_Estado.Update`: `"En curso"`
- `DataCardValue_TotalMiembros.Update`: `CountRows(Filter(ConflictCheck_Miembros, Activo = true))`
- `DataCardValue_NumRespondidos.Update`: `0`
- `DataCardValue_NumConflictos.Update`: `0`
- `DataCardValue_Title.Update`: `"(pendiente de código)"`

*(Si Power Apps no generó tarjetas para algunos de estos campos porque los quitaste de la vista, añádelas de nuevo desde "Editar campos" y simplemente ponlas `Visible: false` en vez de borrarlas — necesitas que existan para poder fijarles un `Update`.)*

**Botón "Enviar solicitud":**
- `OnSelect`: `SubmitForm(FormNuevaSolicitud)`

**`FormNuevaSolicitud.OnSuccess`:**
```
Patch(
  ConflictCheck_Solicitudes,
  FormNuevaSolicitud.LastSubmit,
  { Title: "CC-" & Text(FormNuevaSolicitud.LastSubmit.ID, "0000") }
);
Notify(
  "Solicitud enviada. Se ha avisado a " &
  CountRows(Filter(ConflictCheck_Miembros, Activo = true)) &
  " miembros del grupo.",
  NotificationType.Success
);
Navigate(ScreenMisSolicitudes, ScreenTransition.Cover)
```

> Esto es lo que dispara el flujo de Power Automate (carpeta `02-PowerAutomate-Flujo`): en cuanto `SubmitForm` crea la fila en `ConflictCheck_Solicitudes`, el flujo arranca solo.

---

## Guardar y publicar
**Archivo → Guardar → Publicar**. A partir de aquí la app ya es usable desde
el navegador o la app móvil de Power Apps. El siguiente paso (carpeta
`04-Teams-App`) la convierte en una **pestaña de Teams**.
