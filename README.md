# Landing 1.25 — DuckLuck (CRM)

Landing page con redirección instantánea a WhatsApp + envío de datos a Google Sheets + Meta CAPI (Lead/Purchase) via Apps Script. Integración con CRM externo (Whapify) para resolución de identidad.

---

## Arquitectura

```
Usuario abre la landing
  → Pixel PageView (diferido, no bloquea)
  → Click botón
    → Número seleccionado (aleatorio)
    → Promo code generado (UUID único)
    → Pixel Contact enriquecido (no bloquea)
    → (async) GEO detection + fetch /api/xz3v2q → Google Sheets (no bloquea)
    → Redirect instantáneo a wa.me

Google Sheets (Apps Script):
  → Guarda fila con estado "contact"
  → CRM (Whapify) envía phone + mensaje crudo → Apps Script resuelve identidad
  → Recibe acción LEAD → actualiza fila + envía Lead a Meta CAPI
  → Recibe acción PURCHASE → actualiza fila + envía Purchase a Meta CAPI
```

## Archivos

| Archivo | Descripción |
|---------|-------------|
| `index.html` | HTML + CSS crítico inline + Meta Pixel diferido + JavaScript inline |
| `styles.css` | Estilos completos, cargado async |
| `imagenes/` | `fondo.avif`, `logo.png`, `whatsapp.png`, `favicon.png` |
| `api/xz3v2q.js` | Serverless function (Vercel): recibe datos del frontend, extrae IP/UA, reenvía al Apps Script |
| `credenciales/google-sheets.js` | URL del Google Apps Script |

## Funcionalidades

### Idénticas a Chiva77 1.25

- Meta Pixel diferido (PageView + Contact enriquecido con advanced matching).
- Selección aleatoria de números (`FIXED_PHONES` + `Math.random()`).
- Promo code único por click.
- Mensaje aleatorio.
- Anti doble-click (`__waInFlight`).
- GEO detection (ipapi.co, timeout 900ms).
- Envío a Google Sheets via `/api/xz3v2q` con `keepalive:true`.
- Critical CSS inline + CSS async + preconnect + preload.
- Accesibilidad y responsive.
- fbp / fbc recolección y envío.

### Diferencia principal: integración con CRM (Whapify)

En Chiva77 1.25, la automatización de WhatsApp extrae el promo code limpio del mensaje y lo envía al Apps Script. En DuckLuck, el CRM (Whapify) envía el **mensaje completo** como texto crudo.

| | Chiva77 1.25 | DuckLuck 1.25 |
|---|---|---|
| **CRM recibe** | `promo_code: "CT1-3a8f2bc91d4e"` | `promo_code: "Hola! Vi este anuncio DL3-3a8f2bc91d4e"` |
| **Extracción** | No necesita (ya viene limpio) | Usa `extractPromoCode()` con regex |

### `extractPromoCode()` — Apps Script

```javascript
function extractPromoCode(raw) {
  if (!raw) return "";
  var m = String(raw).match(/([A-Za-z0-9]{2,10}-[a-f0-9]{12})\b/i);
  return m ? m[1] : "";
}
```

Extrae el patrón `{TAG}-{12 hex chars}` de cualquier texto libre.

### Modo D — `handleWhapifyIdentity` (exclusivo de DuckLuck)

El CRM envía un POST con el phone del usuario y el mensaje crudo:

```json
{
  "phone": "5493513131230",
  "promo_code": "Hola! Vi anuncio DL3-abc123def456"
}
```

El Apps Script:
1. Extrae el promo code limpio del texto crudo.
2. Busca la fila en la Sheet que tiene ese promo_code.
3. Actualiza el campo `phone` de esa fila.
4. **NO envía CAPI. NO cambia estado.**

Esto resuelve la identidad: "este promo_code pertenece a este teléfono". Es necesario porque cuando el landing guardó la fila, solo tenía el promo_code pero no el phone real del usuario.

### Búsqueda por phone (Lead/Purchase)

A diferencia de Chiva77 (que busca por `promo_code`), DuckLuck busca por **phone**:

- `handleActionLead`: busca la última fila con ese phone → cambia estado a "lead" → envía Lead CAPI.
- `handleActionPurchase`: busca la última fila con ese phone → actualiza estado/valor → envía Purchase CAPI.

Esto es posible porque el CRM ya resolvió la identidad (promo_code → phone) en el paso de Whapify.

### Apps Script — Modos de entrada

| Modo | Trigger | Qué hace |
|------|---------|----------|
| **A) Contact** | POST sin `action` ni `phone` | Guarda fila con estado "contact" |
| **B1) Lead** | POST con `action: "LEAD"` | Busca por phone, actualiza → Lead CAPI |
| **B2) Purchase** | POST con `action: "PURCHASE"` | Busca por phone, actualiza → Purchase CAPI |
| **C) Simple Purchase** | POST con phone + amount | Crea fila, hereda identidad → Purchase CAPI |
| **D) Whapify Identity** | POST con phone + promo_code (texto crudo) | Extrae promo, resuelve identidad phone↔promo_code |

### CAPI — Datos enviados a Meta

Cada evento Lead/Purchase incluye `user_data` hasheado:

| Campo | Normalización | Hashing |
|-------|---------------|---------|
| `em` | `trim().toLowerCase()` | SHA-256 |
| `ph` | Solo dígitos, prefijo `54` si 10 dígitos | SHA-256 |
| `fn` | `trim().toLowerCase().replace(/\s+/g, " ")` | SHA-256 |
| `ln` | `trim().toLowerCase().replace(/\s+/g, " ")` | SHA-256 |
| `ct` | `normForMeta(value, "ct")` — sin acentos, lowercase, sin espacios | SHA-256 |
| `st` | `normForMeta(value, "st")` — sin acentos, lowercase, sin espacios | SHA-256 |
| `zp` | `normForMeta(value, "zp")` | SHA-256 |
| `country` | `normForMeta(value, "country")` — lowercase 2 chars | SHA-256 |
| `external_id` | — | SHA-256 |
| `fbp` / `fbc` | — | Sin hashear |
| `client_ip_address` | Soporta IPv4 e IPv6 | Sin hashear |
| `client_user_agent` | — | Sin hashear |

API version: `v24.0`.

## Configuración

### `index.html`

```javascript
const LANDING_CONFIG = {
  BRAND_NAME: "",
  MODE: "ads",
  PROMO: { ENABLED: true, LANDING_TAG: "DL3" },
  GEO: { ENABLED: true, PROVIDER_URL: "https://ipapi.co/json/", TIMEOUT_MS: 900 }
};

const FIXED_PHONES = [
  "5493513131230"
];
```

### Pixel ID

- `fbq("init", "816146647896037", {...})`
- `<noscript>` img con el mismo ID.

### `credenciales/google-sheets.js`

```javascript
export const CONFIG_SHEETS = {
  GOOGLE_SHEETS_URL: 'https://script.google.com/macros/s/.../exec'
};
```

### Apps Script

- `PIXEL_ID`: ID del Pixel de Meta.
- `ACCESS_TOKEN`: Token de la API de Conversiones.
- `API_VERSION`: `v24.0`.

## Deploy

Vercel (con serverless functions).

```bash
git add -A && git commit -m "cambios" && git push
```

**Importante**: Autorizar la URL del deploy en Google Apps Script.
