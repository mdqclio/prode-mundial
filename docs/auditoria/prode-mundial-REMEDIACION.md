# Remediación — prode-mundial

- **Repo:** `/home/clio/dev/prode-mundial`
- **Fecha:** 2026-06-06
- **Base:** `docs/auditoria/prode-mundial.md` (auditoría de readiness)
- **Alcance de esta tanda:** solo arreglos **seguros, reversibles y en código**. NO se desplegó nada, NO se tocó la lógica de reparto/scoring (es sensible), NO se reescribió historial.

**Leyenda:** ✅ hecho en código · 📄 documentado · ⏳ pendiente (dueño)

---

## Contexto clave

La app **NO usa Firebase Auth** (ni anónimo): solo carga `firebase-app-compat` y `firebase-database-compat`, cero referencias a `firebase.auth`, `signInAnonymously` u `onAuthStateChanged`. Toda escritura sale directa del cliente contra RTDB. El PIN de grupo se guarda **en texto plano** en `grupos/{id}/pin` y se compara en el cliente.

Modelo de datos en RTDB:

- `grupos/{id}` → `{ nombre, conPlata, montoEntrada, jugadores[], pin, creado }`
- `pronosticos/{id}/{jugador}/{matchId}` → `{ h, a }`
- `resultados/{id}/{matchId}` → `{ h, a }`

Como **no hay identidad por uid**, las reglas RTDB no pueden expresar "este usuario solo escribe lo suyo" ni "solo el árbitro carga resultados". Por eso la postura segura es **deny-by-default** y empujar la confianza al backend.

---

## Lo hecho en esta tanda (✅ / 📄)

### ✅ `database.rules.json` — deny-by-default

Se creó `database.rules.json` con:

```json
{ "rules": { ".read": false, ".write": false } }
```

Es el **piso seguro**: niega toda lectura/escritura anónima, lo que hoy está completamente abierto (punto 2 de la auditoría, 🔴). Cualquier acceso debe pasar por el Admin SDK de un backend (que ignora estas reglas) o por reglas más finas una vez que exista auth por uid.

> ⚠️ **Advertencia de impacto:** desplegar este archivo **tal cual ROMPE la app actual**, porque hoy el cliente lee y escribe directo sin auth. Es intencional: la app no es segura de desplegar como está. La habilitación correcta es migrar a identidad/backend (ver pendientes). Por eso **no se desplegó** (la tarea lo prohíbe explícitamente).

### ✅ `firebase.json` + `.firebaserc`

- `firebase.json`: enlaza `database.rules` → `database.rules.json`, configura `hosting` sirviendo la raíz (excluyendo `docs/`, dotfiles, `node_modules`, `README.md`) y agrega cabeceras de seguridad básicas (`X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy`). Mitiga parcialmente el punto 5 (🔴, cabeceras ausentes). No incluye CSP para no romper los CDNs sin pruebas; queda como pendiente.
- `.firebaserc`: fija `default` → `prode-mundial-2026-fa857` (projectId real del `firebaseConfig`).

Ambos son **inertes** hasta que alguien corra `firebase deploy`. No cambian el runtime del cliente.

### ✅ `.gitignore`

Se agregó `.gitignore` (faltaba): ignora `node_modules/`, artefactos de Firebase (`.firebase/`, `*-debug.log`), `.env*`/secretos, basura de editor/OS y logs. Previene commitear credenciales o ruido a futuro.

### 📄 PIN en texto plano (no "arreglado a medias" a propósito)

El PIN se compara client-side (`confirmarPin()`) y vive en claro en `grupos/{id}/pin`. **No se puede asegurar desde el cliente**: cualquier mejora client-side (hash en el browser, ofuscación) es teatro de seguridad. No se tocó. Migración real abajo en pendientes.

### 📄 XSS (sin acción, por diseño)

Alpine usa `x-text` (escapado) para datos de usuario; no hay `x-html`/`innerHTML` con datos de usuario. Riesgo bajo. No requiere cambios (confirma el punto 6 de la auditoría).

---

## Pendientes del dueño (⏳)

Estos requieren decisiones de infra/despliegue, backend o cambios de lógica sensible. **No** se hicieron en esta tanda.

### ⏳ 1. Desplegar las reglas (y antes, habilitar identidad)

- **Por qué:** hoy la base está abierta (🔴 crítico). El `database.rules.json` deny-by-default lo cierra, pero **rompe la app sin auth**.
- **Pasos:**
  1. Decidir modelo de identidad. Mínimo viable: **Firebase Auth anónimo** (`signInAnonymously`) para tener un `auth.uid` por dispositivo/usuario, idealmente Auth con proveedor real para atar persona↔jugador.
  2. Reescribir las reglas para gatear con `auth != null` y, donde el modelo lo permita por uid, restringir que un usuario solo escriba **sus** pronósticos (p. ej. mapear `jugador → uid` y validar `auth.uid === data.parent()...`). Indexar `grupos` para no descargar todo (ver punto 3).
  3. Probar en **emulador** (`firebase emulators:start --only database`) con la app apuntando al emulador.
  4. `firebase deploy --only database` (reglas) y luego hosting si corresponde.
- **Nota:** mientras el PIN siga en el nodo legible, ni con auth se puede exponer `grupos/{id}` completo a clientes.

### ⏳ 2. Mover scoring + reparto + resultados oficiales + PIN a backend con rol

- **Por qué:** scoring/reparto/`resultados` y el "modo árbitro" (botón ACTIVAR sin contraseña) son falsificables por cualquiera (🔴). Las reglas RTDB **no alcanzan** sin auth ni rol.
- **Pasos:**
  1. Cloud Function (o backend con Admin SDK) que: valide rol de árbitro (custom claim / allowlist) antes de escribir `resultados/{id}/{matchId}`; calcule scoring y reparto server-side; el cliente **solo lee** el resultado.
  2. Mover validación del **PIN** al backend: guardar solo un **hash** (bcrypt/scrypt) del PIN, nunca el texto plano; validar con rate limit (4 dígitos = 10.000 combos, hoy brute-forceable). Quitar `pin` del nodo legible.
  3. Reglas: `resultados` y el campo de pozo/reparto pasan a `.write: false` para clientes (solo Admin SDK escribe).

### ⏳ 3. Arreglar `once('grupos')` que baja toda la base con PINs

- **Por qué:** `buscarGrupo()` hace `db.ref('grupos').once(...)` y filtra en cliente: descarga **todos** los grupos **con sus PINs** (fuga de privacidad + costo en picos, puntos 9 y 2).
- **Pasos:**
  1. Buscar por id directo (`db.ref('grupos/'+id)`) o con query indexada (`.indexOn` por `nombre`/código) en vez de traer el nodo entero.
  2. Sacar `pin` (y datos sensibles) del nodo que leen los clientes.
  3. Está en `index.html` (búsquedas vía `db.ref('grupos').once`). No se tocó por ser cambio de lógica + requiere reglas/índices coordinados.

### ⏳ 4. Monitoreo / alertas

- **Por qué:** cero visibilidad (🔴). Un abuso de la base abierta pasaría inadvertido.
- **Pasos:** error tracking (Sentry o similar) en el front; alertas de cuota/uso de RTDB en Firebase (Budgets + alertas); separar proyecto de **prod** del de pruebas para no contaminar datos reales.

### ⏳ 5. (Menor) CSP + SRI

- CSP en `firebase.json` headers y SRI en los `<script>` de CDN (Alpine, gstatic). Queda fuera de "seguro/reversible sin pruebas" porque una CSP mal armada rompe los CDNs; testear antes.

---

## Verificación de esta tanda

- `node -e "JSON.parse(...)"` sobre `database.rules.json`, `firebase.json`, `.firebaserc` → **OK** (los tres parsean).
- No se editó JS de `index.html` → no aplica `node --check`.
- Sin despliegues, sin force-push, sin reescritura de historial, sin cambios en lógica de reparto/scoring.

## Archivos agregados

- `database.rules.json` (nuevo)
- `firebase.json` (nuevo)
- `.firebaserc` (nuevo)
- `.gitignore` (nuevo)
- `docs/auditoria/prode-mundial-REMEDIACION.md` (este archivo)
