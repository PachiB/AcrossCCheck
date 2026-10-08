# 4. "Conflict Check" como pestaña en Teams

Hay dos maneras de conseguirlo. Usa la **A** si quieres algo inmediato en un
canal concreto; usa la **B** si quieres una app con marca propia de Across
Legal fija en la barra lateral de Teams de todo el despacho.

## Método A — Pestaña de canal en 30 segundos (recomendado para empezar)
Una vez la Power App esté publicada (carpeta `03-PowerApps-App`):

1. Abre el canal de Teams del grupo de socios/conflict check.
2. Pulsa el **+** junto a las pestañas de arriba.
3. Busca la app **"Power Apps"** (es nativa de Microsoft, ya está en todos los tenants).
4. Elige tu app **"Conflict Check"** de la lista (aparece automáticamente porque está en tu entorno de Power Apps).
5. Guardar.

Así cualquier miembro del canal abre la app de un clic, incluida la app móvil de Teams. No requiere nada de lo que hay en esta carpeta.

## Método B — App personalizada con marca Across Legal (icono propio en la barra lateral)

Este repo incluye ya `manifest.json`, `color.png` y `outline.png` listos para empaquetar.

### Paso 1 — Publica la Power App y copia sus IDs
1. En [make.powerapps.com](https://make.powerapps.com), abre la app **Conflict Check** → **Compartir/Detalles** → copia:
   - **App ID** (GUID en la URL o en Detalles)
   - **Environment ID** (en la URL, después de `/environments/`)
2. El **Tenant ID** lo obtienes en [admin.microsoft.com](https://admin.microsoft.com) → **Configuración** → **Organización**, o pídelo a tu administrador de Microsoft 365.

### Paso 2 — Rellena el manifest
Edita `manifest.json` y sustituye:
- `__ENVIRONMENT_ID__` → tu Environment ID
- `__POWERAPP_ID__` → tu App ID
- `__TENANT_ID__` → tu Tenant ID

(También puedes ajustar `developer.websiteUrl`, `privacyUrl`, `termsOfUseUrl` si Across Legal tiene páginas reales para esto; Teams exige que esos campos existan pero no valida mucho su contenido para apps internas.)

### Paso 3 — Empaqueta el .zip
El paquete de Teams es un `.zip` con exactamente estos 3 archivos **en la raíz** (sin subcarpeta):
```
manifest.json
color.png
outline.png
```
Puedes generarlo así desde esta carpeta:
```bash
cd 04-Teams-App
zip -j ConflictCheck-TeamsApp.zip manifest.json color.png outline.png
```

### Paso 4 — Súbela a Teams
- **Opción rápida (solo para ti/pruebas):** Teams → Apps → **Administrar tus apps** → **Cargar una app personalizada** → selecciona el `.zip`.
- **Opción para todo el despacho:** [admin.teams.microsoft.com](https://admin.teams.microsoft.com) → **Teams apps → Gestionar apps → Cargar una nueva app**, súbela, y luego en **Directivas de permisos de apps** autorízala para los usuarios o grupos que correspondan (p.ej. el grupo de socios).

Una vez instalada, cada usuario la fija a su barra lateral (clic derecho → **Fijar**) y tiene ahí mismo el acceso directo a "Nueva solicitud" y "Mis solicitudes", con el icono de Across Legal.

> Nota: el manifest incluido usa un tab **personal** (aparece en la barra lateral de cada usuario). Si además quieres forzar esta app concreta como pestaña de un canal específico sin pasar por el Método A, dímelo y amplío el manifest con `configurableTabs`, que requiere un paso extra de configuración la primera vez que se añade al canal.
