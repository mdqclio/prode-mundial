# Auditoría de readiness para producción — prode-mundial

- **Repo:** `/home/clio/dev/prode-mundial`
- **Fecha:** 2026-06-06
- **Qué es:** App web de prode/quiniela del Mundial 2026. Los grupos de amigos cargan jugadores, pronostican resultados de los partidos de fase de grupos, un "árbitro" carga los resultados oficiales y la app reparte un pozo en plata proporcional a los puntos.
- **Stack:** SPA estática de un solo archivo (`index.html`, ~72 KB). Front: Alpine.js 3.x (CDN jsDelivr) + Google Fonts. Backend: **Firebase Realtime Database** (SDK compat 10.12.0 vía CDN gstatic). Sin build, sin backend propio, sin Cloud Functions. Toda la lógica vive en el cliente.

**Leyenda:** 🟢 ok · 🟡 mejorable · 🔴 bloqueante · ⚪ N/A

---

## 1) Front comprimido / sin secretos de cliente — 🟡

El front es un único HTML servido tal cual, sin minificar ni comprimir en build (depende de gzip/brotli del hosting). Carga Alpine y Firebase desde CDNs de terceros **sin SRI (Subresource Integrity)**: si un CDN se compromete, se ejecuta JS arbitrario sobre los datos. En cuanto a secretos: la `firebaseConfig` (`apiKey: "AIzaSy..."`, `databaseURL`, etc.) es config **pública de Firebase y NO cuenta como secreto**. No se detectaron tokens, API keys de terceros ni credenciales privadas embebidas.

**Riesgo:** bajo por secretos; medio por cadena de suministro (CDN sin SRI) y peso sin optimizar.

## 2) RLS / reglas de seguridad de la base — 🔴

**Punto más grave.** No existe ningún archivo de reglas (`database.rules.json`) en el repo, y toda la lógica de escritura es client-side directo contra rutas RTDB (`grupos/`, `pronosticos/`, `resultados/`). Sin reglas restrictivas, la base está **abierta a lectura/escritura para cualquiera** con la `databaseURL` (que es pública y está en el HTML). Cualquiera puede: leer todos los grupos y PINs, borrar/sobrescribir cualquier grupo, falsificar pronósticos de otros, y cargar resultados oficiales falsos. El "PIN de 4 dígitos" se guarda **en texto plano** dentro del propio nodo del grupo y se compara en el cliente (`confirmarPin()`), por lo que no protege nada a nivel base.

**Riesgo:** crítico. Integridad total del prode y del reparto de plata comprometida; fuga de PINs y datos de todos los grupos.

## 3) Git sin secretos — 🟢

Historial revisado (7 commits, `git log -p --all`). No hay `.env`, ni keys de terceros, ni tokens filtrados. Solo aparece la `firebaseConfig` pública, que es esperable y no es un secreto.

**Riesgo:** nulo.

## 4) APIs: auth / permisos / validación — 🔴

No hay capa de API: el cliente escribe directo en la base. **No hay autenticación de ninguna clase** (ni Firebase Auth anónimo). La autorización es puramente cosmética en el cliente:

- **Modo árbitro:** se activa con un botón "ACTIVAR" sin contraseña (`adminMode = true`). Cualquier usuario carga resultados oficiales (`guardarResultado` → `resultados/{grupo}/{match}`), lo que reescribe el ranking y el reparto del pozo de todos.
- **Pronósticos:** `guardarPronostico` escribe en `pronosticos/{grupo}/{jugador}/{match}`. Como el "jugador" es solo un string elegido en un selector sin verificación, cualquiera puede **editar los pronósticos de otro jugador**.
- **Scoring/reparto:** `calcularPuntos` y los repartos de plata se computan **solo en el cliente**; nada se valida ni firma del lado servidor.
- **Validación:** existe validación de entrada básica (PIN 4 dígitos regex, caracteres inválidos para keys RTDB, min 2 jugadores, límites de goles 0–15 en el input) pero es **bypasseable** al escribir directo en la base.

**Riesgo:** crítico. Un usuario edita puntajes/resultados ajenos y los oficiales; es exactamente el escenario que un prode no debe permitir.

## 5) Hosting / entornos / env vars — 🔴

No hay configuración de hosting en el repo (`firebase.json`, headers, `_redirects`, CI/CD, Dockerfile — ninguno). No hay separación de entornos (dev/staging/prod): un solo proyecto Firebase hardcodeado. Sin env vars (apropiado para un sitio estático, pero la falta de proyecto separado de prod significa que cualquier prueba toca datos reales). Sin cabeceras de seguridad declaradas (CSP, HSTS, X-Content-Type-Options), lo que agrava el riesgo de XSS/supply-chain.

**Riesgo:** alto. Sin proceso de deploy reproducible ni aislamiento de datos de prueba vs producción.

## 6) Login / sesiones / vulnerabilidades — 🔴

No hay login ni sesión real. La "identidad" es un nombre de jugador en `localStorage` y un PIN compartido por grupo guardado en claro en la base y en `localStorage` (`prode-auth-{id}`). Implicancias:

- **Sin identidad por usuario:** elegir "quién sos" es un dropdown; no hay nada que ate a una persona a su jugador.
- **PIN débil:** 4 dígitos (10.000 combos) comparados en cliente, sin rate limit ⇒ fuerza bruta trivial; además se puede leer directo de la base.
- **XSS:** Alpine usa `x-text` (escapado) para los datos del usuario (nombres de jugador, nombre de grupo), lo cual es seguro. No se encontró `innerHTML`/`x-html` con datos de usuario. El riesgo de XSS reflejado/almacenado es **bajo**, pero la ausencia de CSP deja la puerta abierta si se introduce `x-html` a futuro o si se compromete un CDN.

**Riesgo:** alto. No hay control de identidad y el secreto de acceso (PIN) es débil y expuesto.

## 7) Rate limiting — 🔴

Inexistente. Al escribir directo en RTDB desde el cliente sin reglas ni backend, no hay throttling de escrituras ni de intentos de PIN. Un script puede borrar o inflar la base, o brute-forcear PINs sin freno.

**Riesgo:** alto. Abuso, fuerza bruta de PIN y posible vaciado/spam de la base.

## 8) Caché — 🟡

Sin `Cache-Control` declarado (depende del hosting) y sin Service Worker, pese a tener manifest-like meta tags de PWA (`apple-mobile-web-app-capable`). Las dependencias por CDN cachean bien, pero el HTML principal no tiene estrategia. La app mantiene listeners `on('value')` en vivo (tiempo real) y un fallback `onceWithTimeout`, lo cual es razonable para los datos, pero no hay caché offline real ni revalidación del shell.

**Riesgo:** bajo/medio. Sin impacto de seguridad; afecta velocidad de carga y experiencia offline.

## 9) Escalabilidad — 🟡

Para un uso de "grupos de amigos" RTDB escala bien. Dos puntos de fricción en picos durante partidos:

- **`buscarGrupo()` descarga TODO el nodo `grupos` completo** (`db.ref('grupos').once`) y filtra en cliente. Con muchos grupos esto crece linealmente en ancho de banda por cada búsqueda y expone todos los grupos/PINs.
- Listeners `on('value')` sobre `pronosticos/{grupo}` traen **todos los pronósticos de todos los jugadores** del grupo en cada cambio; aceptable por grupo chico, costoso si un grupo crece.

Sin las reglas del punto 2, además, cualquier pico malicioso es ilimitado.

**Riesgo:** medio. Costos y latencia crecientes; la búsqueda global es el cuello de botella y un agujero de privacidad.

## 10) Monitoreo / alertas — 🔴

No hay nada: sin error tracking (Sentry/similar), sin analytics, sin logging, sin alertas de cuota/abuso de Firebase, sin health checks. Los errores se manejan con `alert()`/`console.error`. Ante un ataque o caída no hay forma de enterarse.

**Riesgo:** alto. Cero visibilidad operativa; un abuso de la base abierta pasaría inadvertido hasta ver la factura o la data corrupta.

---

## Tabla resumen

| # | Punto | Estado |
|---|-------|--------|
| 1 | Front comprimido / sin secretos cliente | 🟡 |
| 2 | RLS / reglas de la base | 🔴 |
| 3 | Git sin secretos | 🟢 |
| 4 | APIs: auth / permisos / validación | 🔴 |
| 5 | Hosting / entornos / env vars | 🔴 |
| 6 | Login / sesiones / vulns | 🔴 |
| 7 | Rate limiting | 🔴 |
| 8 | Caché | 🟡 |
| 9 | Escalabilidad | 🟡 |
| 10 | Monitoreo / alertas | 🔴 |

**Veredicto:** NO apto para producción. 5 bloqueantes, todos derivados de la misma raíz: base de datos abierta sin auth ni reglas, con toda la lógica de confianza en el cliente.

---

## Los 3 arreglos más urgentes

1. **Cerrar la base con reglas + auth real (puntos 2, 4, 6).** Publicar `database.rules.json` que niegue por defecto y exija identidad (Firebase Auth, mínimo anónimo). El PIN/credencial del grupo no debe vivir en el nodo legible; validar pertenencia y rol de árbitro en las reglas, no en el cliente. Esto es lo que impide que un usuario edite pronósticos ajenos o cargue resultados oficiales falsos.

2. **Mover scoring, reparto y carga de resultados oficiales fuera del cliente (puntos 4, 7).** Resultados oficiales y cálculo del pozo deben escribirse solo vía Cloud Function / backend con rol verificado y rate limiting; el cliente solo lee. Sin esto, el ranking y la plata son falsificables por cualquiera.

3. **Arreglar la búsqueda global y agregar monitoreo (puntos 9, 10, 5).** Reemplazar el `once('grupos')` que baja toda la base (fuga de PINs + costo en picos) por consulta indexada/por id, y conectar error tracking + alertas de cuota/abuso de Firebase, separando proyecto de prod del de pruebas.
