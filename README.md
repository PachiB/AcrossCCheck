# Across Legal — Conflict Check

Proceso digital de conflict check: el solicitante crea la ficha del caso, se
avisa automáticamente a los ~12 miembros del grupo de conflicto por Teams
(respuesta de un toque, también desde el móvil), y tanto el solicitante como
los administradores ven en tiempo real quién ha contestado, qué ha
contestado y si hay alguna alerta de potencial conflicto.

## Arquitectura elegida
- **Datos:** 4 listas de SharePoint, en el sitio ya compartido
  `https://acrosslegal.sharepoint.com/sites/Socios-ConflictCheck`.
- **Distribución y respuesta:** Power Automate + **Approvals nativo de
  Microsoft Teams** (app móvil incluida) — sin desarrollo de UI para
  responder, máxima agilidad desde el móvil.
- **Interfaz del solicitante / seguimiento / panel admin:** una Power App
  (canvas) sobre esas listas.
- **Acceso desde Teams:** la Power App como pestaña (personal o de canal).

## Qué hace cada carpeta (constrúyelo en este orden)

| Carpeta | Contenido | Tiempo estimado |
|---|---|---|
| [`01-SharePoint-Listas`](01-SharePoint-Listas/README.md) | 4 plantillas Excel para crear las listas `Solicitudes`, `Respuestas`, `Miembros`, `Admins` | ~10 min |
| [`02-PowerAutomate-Flujo`](02-PowerAutomate-Flujo/README.md) | Receta exacta, acción por acción, del flujo que reparte la aprobación a los 12, registra respuestas en vivo y cierra el caso | ~25 min |
| [`03-PowerApps-App`](03-PowerApps-App/README.md) | Fórmulas para las 3 pantallas: Nueva Solicitud, Mis Solicitudes (+ panel admin), Detalle con seguimiento | ~30 min |
| [`04-Teams-App`](04-Teams-App/README.md) | Cómo añadirla como pestaña de Teams: método rápido (nativo) y método con marca propia (manifest incluido) | ~5–15 min |

Las plantillas Excel te las adjunto directamente en el chat (y están en la
carpeta `01-SharePoint-Listas/` de este repositorio) — esta sesión solo tiene
permiso de **lectura** sobre vuestro SharePoint, así que no he podido
subirlas yo mismo a la carpeta `⚖️ Conflict Check`. Descárgalas y súbelas tú
al crear cada lista (es el mismo paso de "Desde Excel" que ya tocaba hacer).

## Flujo end-to-end (resumen)

1. El solicitante abre la Power App (desde Teams o el navegador) → **Nueva
   solicitud** → rellena Cliente, Asunto, Contraparte, Comentarios, Urgente.
2. Al enviar, se crea una fila en `ConflictCheck_Solicitudes`, lo que
   dispara automáticamente el flujo de Power Automate.
3. El flujo lee quién está activo en `ConflictCheck_Miembros` y lanza, **en
   paralelo**, una aprobación de Teams a cada uno con los datos del caso y
   dos botones: **"No hay conflicto"** / **"Potencial conflicto"** (esta
   última pide comentario).
4. Cada respuesta se guarda al instante en `ConflictCheck_Respuestas`. Si es
   "Potencial conflicto", se avisa de inmediato por Teams.
5. El solicitante y los administradores ven en la Power App, en todo
   momento, cuántos de los 12 han contestado y qué ha dicho cada uno.
6. Cuando responden todos, el flujo cierra la solicitud como
   **"Sin conflicto"** o **"Potencial conflicto"** y notifica el resultado
   final.

## Qué he podido hacer yo mismo y qué falta por hacer en el portal

Esta sesión no tiene control del navegador/escritorio de tu ordenador ni
acceso de administrador a vuestro tenant de Microsoft 365/Power Platform,
así que no he podido crear las listas, el flujo ni la app directamente en
vuestro entorno. Lo que he dejado listo:

- Las 4 plantillas Excel (importables en 2 minutos cada una).
- La receta del flujo con cada acción y cada expresión ya escrita, para
  copiar y pegar en el diseñador.
- Las fórmulas exactas de cada pantalla y botón de la Power App.
- El manifest y los iconos de la app de Teams, listos para empaquetar.

Siguiendo las 4 carpetas en orden, todo el proceso queda operativo en menos
de 1h15 sin necesidad de ningún desarrollador adicional. Si en algún paso
el portal te pide algo que no coincide con lo descrito aquí (Microsoft
cambia la UI de vez en cuando), dime exactamente qué ves y te doy la
equivalencia actual.
