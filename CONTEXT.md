# Fintrack La Roca — Contexto del proyecto

Repo: https://github.com/jorgedelarocacalix-tech/fintrack-laroca  
App live: https://jorgedelarocacalix-tech.github.io/fintrack-laroca/  
Stack: Vanilla JS · HTML único (index.html) · Supabase · Edge Functions · GitHub Pages  
Supabase project ID: `upaenjotkocmdvfuobii`  
Supabase URL: `https://upaenjotkocmdvfuobii.supabase.co`

---

## Usuarios / PIN

| PIN  | Quién  |
|------|--------|
| 1234 | Jorge Daniel (default) |

PIN guardado en `localStorage` bajo la clave `ft_v1_pin`.  
Claude API key (opcional) bajo `ft_v1_claudekey`.

---

## Propósito

App de control financiero personal de Jorge Daniel (Corporación La Roca, Honduras).  
Maneja:
- **Tarjetas de crédito** de Jorge + Isis Johana Fonseca Vega (su esposa)
- **Pagos a proveedores** de La Roca

Objetivo principal: ver qué se debe pagar, cuándo, y cuánto — para no caer en mora.

---

## Supabase — Tablas

### `ft_tarjetas` — Catálogo de tarjetas
| campo | tipo | descripción |
|-------|------|-------------|
| id | uuid | PK |
| banco | text | Nombre del banco |
| cuenta | text | Últimos 4 dígitos |
| titular | text | 'jorge' o 'isis' |
| limite | numeric | Límite de crédito |

**Registros actuales (17 tarjetas):**
- Jorge: BAC 2755, BAC 8366, BAC 0799, BAC 5542, Atlántida 4901, Banrural (sin nro), Ficohsa 4484, Ficohsa 2127, Ficohsa 3630 (préstamo), Promerica (XF-5), Banpaís 9285 (MOVESA), Banpaís 9277 (El Baratillo), Banpaís 8231 (Italika/Diunsa/Jetstereo)
- Isis: BAC 4 tarjetas (pendiente de confirmar cuentas exactas)

### `ft_registros_mes` — Estado de cuenta por tarjeta/mes
| campo | tipo |
|-------|------|
| id | uuid |
| tarjeta_id | uuid FK |
| mes | text | 'junio 2026' |
| fecha_corte | date |
| fecha_pago | date |
| valor_pagar | numeric |
| pago_minimo | numeric |
| disponible | numeric |
| saldo_anterior | numeric |
| total_compras | numeric |
| total_pagos | numeric |
| total_intereses | numeric |
| moneda | text | 'L' o 'USD' |
| estado | text | 'pendiente', 'pagado', 'mora' |
| nota | text |
| pdf_url | text |

**Datos de junio 2026 ya cargados** (13 registros):
- BAC 2755: pago 30/jun, L67,833.95, mín L21,773 — URGENTE
- BAC 8366: pago 29/jun, L81,055.15, mín L31,801 — URGENTE
- BAC 0799: pago 29/jun, L0 — PAGADO
- BAC 5542: vencido 22/jun, L57,058.20 — MORA
- Atlántida 4901: vencido 17/jun, L175,904.09 — MORA
- Banrural: vencido 11/jun, L290 (solo seguros) — MORA
- Ficohsa 4484: pago 11/jul, L231,697.56, mín L14,183.11 — pendiente
- Ficohsa 2127: pago 11/jul, L34,360.50, mín L573.82 — pendiente
- Ficohsa 3630: L356,073.36 — PAGADO (préstamo cancelado 23/jun)
- Promerica: L518,056.70 — PAGADO (cancelación total 23/jun)
- Banpaís 9285: pago 14/jul, L523,225.90, mín L39,407.53 — pendiente
- Banpaís 9277: pago 14/jul, L71,893.71, mín L2,100 — pendiente
- Banpaís 8231: pago 14/jul, L98,166.31, mín L11,300 — pendiente

### `ft_proveedores_catalogo` — Catálogo de proveedores
Campos: id, nombre, categoria, contacto

### `ft_pagos_proveedor` — Pagos a proveedores
Campos: id, proveedor_id, mes, fecha_vence, monto, estado, nota

### Supabase Storage
Bucket: `estados-cuenta` — guarda PDFs de estados de cuenta subidos

---

## Edge Functions (Supabase)

### `analizar-estado` (proyecto fintrack)
- URL: `https://upaenjotkocmdvfuobii.supabase.co/functions/v1/analizar-estado`
- verify_jwt: false (usa apikey header del cliente)
- Recibe: `{ pdf_base64, banco, cuenta, mes }`
- Llama a: Claude Sonnet 4.6 con el PDF como document
- Devuelve: `{ fecha_corte, fecha_pago, valor_pagar, pago_minimo, saldo_anterior, total_compras, total_pagos, total_intereses, disponible, moneda }`
- API key Claude: en `ANTHROPIC_API_KEY` env var (misma que proyecto Cobranza)

---

## Funcionalidades implementadas

### 1. Login con PIN
- Pantalla de login con 4 puntos y teclado numérico
- PIN default: 1234

### 2. Tab Tarjetas
Sub-tabs: Resumen · Jorge · Isis · 📅 Fechas · 📤 Estado · ＋ Registrar

**Resumen:**
- 2 chips financieros: "Total por pagar" (L) + "Total pagado" (L) — en rojo y verde
- 3 chips de conteo: Mora/Urgente · Próximos · Pagados
- Lista de cards ordenadas por urgencia (mora → urgente → próximo → ok → pagado)

**Cada card muestra:**
- Banco + cuenta + badge Titular (Jorge/Isis)
- `Pago: DD/mes · Corte: DD/mes`
- `Disponible: L xxx · Mín: L xxx`
- Nota (si tiene)
- Badge de estado (MORA / URGENTE / PRÓXIMO / OK / PAGADO)
- Botones: ✓ Pagado + 📄 Subir estado

**Jorge / Isis tabs:** Misma vista filtrada por titular

**📅 Fechas:** Calendario consolidado con todos los pagos por fecha

**📤 Estado de cuenta (FLUJO PRINCIPAL):**
1. Seleccionar tarjeta
2. Subir PDF del banco
3. La app lo manda a Edge Function `analizar-estado`
4. Claude lee el PDF (incluso escaneados) y extrae todos los datos
5. El formulario se pre-llena automáticamente
6. Jorge revisa y da Guardar

Sin API key en el browser — la clave vive en Supabase.

**＋ Registrar:** Formulario manual para agregar/editar registro del mes

### 3. Tab Proveedores
Sub-tabs: Resumen · ＋ Agregar · Lista
Similar a tarjetas pero para proveedores

### 4. Botón ✦ IA
Análisis conversacional con Claude (requiere API key en Config si se quiere usar)

### 5. ⚙ Config
- Cambiar PIN
- Configurar Claude API key (opcional, para análisis IA)

---

## Responsive UI — EN PROCESO (pendiente completar)

**Estado actual (25/jun/2026):** Se empezó a implementar pero quedó incompleto.

**Lo que se hizo:**
- CSS agregado para breakpoint 768px
- Clases `.vista-cards` / `.vista-tabla` definidas en el CSS
- `fmtChip()` y `chips()` actualizados con montos

**Lo que FALTA implementar:**
- Función `renderTablaRegistros(registros)` — tabla HTML para desktop
- Envolver los cards en `<div class="vista-cards">` en `renderResumenTarjetas()`, `renderListaTarjetas()`
- Agregar `<div class="vista-tabla">` con la tabla en esas mismas funciones

**Diseño de la tabla desktop:**
```
| Estado | Banco | Cuenta | Titular | Corte | Fecha Pago | Total L | Mínimo L | Disponible L | Nota | Acciones |
```
Filas coloreadas según urgencia (mismos colores que las cards).

---

## Google Calendar — Alertas creadas

Eventos ya en el calendario de jorgedelaroca.calix@gmail.com (alertas 1 día antes, 4 PM Honduras):
- 28/jun → BAC 8366 (L81,055)
- 29/jun → BAC 2755 (L67,834)
- 10/jul → Ficohsa 4484 + 2127 (L266,058)
- 13/jul → Banpaís 9285 + 9277 + 8231 (L693,286)

**Pendiente:** Implementar en la app un botón "📅 Agregar a Calendar" que cree automáticamente el evento al guardar un estado de cuenta nuevo.

---

## Archivos clave

```
/Users/jorgecalix/fintrack-laroca/
  index.html   — App completa (~1600 líneas), único archivo
  CONTEXT.md   — Este archivo
```

GitHub: https://github.com/jorgedelarocacalix-tech/fintrack-laroca  
GitHub Pages: https://jorgedelarocacalix-tech.github.io/fintrack-laroca/

---

## Constantes en index.html

```js
const SB_URL = 'https://upaenjotkocmdvfuobii.supabase.co'
const SB_KEY = 'eyJhbGci...'   // anon key pública
const SK     = 'ft_v1'         // prefijo localStorage
```

---

## Notas técnicas

- App es un único `index.html` sin build ni bundler
- Supabase se usa con REST API directa (`fetch` al endpoint `/rest/v1/`)
- RLS: políticas abiertas `acceso_total` en todas las tablas `ft_`
- PDF.js incluido como CDN pero ya NO se usa para análisis (Edge Function reemplaza)
- SheetJS incluido como CDN para posible lectura de Excel (sin uso activo ahora)
- `mesActual()` devuelve el mes seleccionado, formato 'junio 2026'
- `urgencia(fechaISO, estado)` → 'mora' | 'urgente' | 'proximo' | 'ok' | 'sinfecha' | 'pagado'
- `fmtFecha(iso)` → '29/jun'
- `fmt(n)` → número con comas
