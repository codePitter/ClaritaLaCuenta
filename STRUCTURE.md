# Clarita la Cuenta — Referencia Técnica Completa

> Este documento describe en detalle la arquitectura, convenciones, flujos y decisiones de implementación del proyecto.
> Última actualización: 8 de marzo 2026

---

## Estructura de archivos

```
FinControl/              ← nombre de repo (el branding es "Clarita la Cuenta")
│
├── index.html           (~894 líneas)
│   ├── <head>
│   │   ├── Favicon SVG inline (verde esmeralda #059669 + ícono blanco)
│   │   ├── SEO meta tags (description, keywords, author, robots, canonical)
│   │   ├── Open Graph (og:type, og:url, og:title, og:description, og:locale, og:site_name)
│   │   ├── Twitter Card (summary)
│   │   ├── JSON-LD (schema.org WebApplication)
│   │   ├── Google Fonts (8 familias)
│   │   ├── Chart.js 4.4.1 (CDN)
│   │   ├── Supabase JS v2 (CDN)
│   │   └── Google AdSense (async — requiere Publisher ID real)
│   │
│   ├── #authScreen              ← pantalla de login/registro
│   │   ├── #loginForm           ← tab login
│   │   └── #registerForm        ← tab registro
│   │
│   └── #appShell  (hidden hasta login)
│       ├── <nav.topnav>         ← barra superior fija
│       ├── <aside.sidebar>      ← menú móvil
│       ├── <main.app-main>
│       │   ├── #sec-dashboard   (+ banner horizontal ad)
│       │   ├── #sec-gastos      (+ footer ad)
│       │   ├── #sec-presupuesto (+ footer ad)
│       │   ├── #sec-calendario
│       │   ├── #sec-deudas      (+ footer ad)
│       │   ├── #sec-ahorro
│       │   └── #sec-anual       (+ footer ad)
│       ├── <aside.ad-sidebar>   ← sidebar publicitario sticky (≥1200px)
│       ├── <aside.theme-panel>  ← panel de temas (slide-in)
│       └── Modals
│           ├── #txOverlay       ← nueva transacción
│           ├── #calOverlay      ← nuevo evento de calendario
│           ├── #debtOverlay     ← nueva deuda
│           └── #savingOverlay   ← nueva alcancía
│
├── css/style.css        (~1300 líneas)
│   ├── Reset & base
│   ├── Theme tokens (6 temas × 2 modos = 12 bloques de variables)
│   ├── Swatch classes para theme panel
│   ├── Efectos de fondo por tema (::before / ::after)
│   ├── Auth screen
│   ├── Topnav
│   ├── Sidebar
│   ├── App layout
│   ├── Componentes (cards, KPIs, buttons, chips, badges, tables,
│   │              forms, progress bars, calendar, debt cards,
│   │              calc result, goals, budget rows, charts, grids,
│   │              savings/alcancías, budget month nav)
│   ├── Modals
│   ├── Theme panel + mini preview
│   ├── Toast
│   ├── Publicidad (ad-banner-wrap, ad-footer-wrap, ad-sidebar, ad-label)
│   └── Responsive (1200px, 1024px, 768px, 600px, 480px)
│
├── js/main.js           (~1895 líneas, ~105 funciones)
│   └── [ver sección detallada abajo]
│
├── js/config.js         ← Credenciales Supabase + Google (gitignored)
├── js/config.example.js
├── fc-rescue.js         ← Script de rescate para consola del navegador
├── supabase_schema.sql
├── .gitignore
├── README.md
└── STRUCTURE.md
```

---

## SEO (index.html `<head>`)

### Meta tags implementados
```html
<!-- Favicon -->
<link rel="icon" type="image/svg+xml" href="data:image/svg+xml,...">   <!-- SVG verde + ícono -->
<link rel="apple-touch-icon" href="...">                                <!-- iOS -->
<meta name="theme-color" content="#059669">                             <!-- Android/Chrome -->

<!-- SEO primario -->
<title>Clarita la Cuenta | Finanzas personales sin drama 💚</title>
<meta name="description" content="...">   <!-- ~155 chars, keywords + Argentina -->
<meta name="keywords" content="finanzas personales, presupuesto personal, ...">
<meta name="author" content="Clarita la Cuenta">
<meta name="robots" content="index, follow">
<link rel="canonical" href="https://codepitter.github.io/FinControl/">

<!-- Open Graph -->
<meta property="og:type"        content="website">
<meta property="og:url"         content="https://codepitter.github.io/FinControl/">
<meta property="og:title"       content="...">
<meta property="og:description" content="...">
<meta property="og:locale"      content="es_AR">
<meta property="og:site_name"   content="Clarita la Cuenta">

<!-- Twitter/X Card -->
<meta name="twitter:card"        content="summary">
<meta name="twitter:title"       content="...">
<meta name="twitter:description" content="...">

<!-- JSON-LD (schema.org) -->
<script type="application/ld+json">
{ "@type": "WebApplication", "applicationCategory": "FinanceApplication",
  "offers": { "price": "0", "priceCurrency": "ARS" }, "inLanguage": "es-AR" }
</script>
```

> **Pendiente:** Agregar `<meta property="og:image">` con screenshot de la app guardado en `/og-image.png`.

---

## Publicidad (Google AdSense)

### Bloques implementados
| ID / clase | Ubicación | Tipo | Visible en |
|---|---|---|---|
| `#adBannerDash` `.ad-banner-wrap` | Dashboard, arriba de los KPIs | Horizontal auto | Todos |
| `.ad-footer-wrap` | Pie de Gastos | Horizontal auto | Todos |
| `.ad-footer-wrap` | Pie de Presupuesto | Horizontal auto | Todos |
| `.ad-footer-wrap` | Pie de Deudas | Horizontal auto | Todos |
| `.ad-footer-wrap` | Pie de Anual | Horizontal auto | Todos |
| `#adSidebar` `.ad-sidebar-slot` | Sidebar derecha fija (sticky) | 160×600 | ≥ 1200px |

### Activación
1. Reemplazar `ca-pub-XXXXXXXXXX` con el Publisher ID real de AdSense
2. Reemplazar cada `data-ad-slot="XXXXXXXXXX"` con el Slot ID de cada unidad
3. La etiqueta `<span class="ad-label">Publicidad</span>` es requerida por las políticas de AdSense

### CSS de publicidad (al final de style.css)
```css
.ad-label         /* etiqueta "Publicidad" discreta — 9px, muted, opacity 0.6 */
.ad-banner-wrap   /* contenedor horizontal, min-height 90px, flex column */
.ad-footer-wrap   /* pie de sección, borde superior, min-height 70px */
.ad-sidebar       /* sticky top:80px, width:160px, oculto en <1200px */
.ad-sidebar-inner /* flex column, gap 4px */
.ad-sidebar-slot  /* 160px wide, min-height 600px */
```

---

## Variables CSS (Design Tokens)

Cada combinación `[data-theme][data-mode]` define el mismo conjunto de tokens:

```css
--clr-bg          /* fondo principal */
--clr-bg2         /* fondo secundario (cards, modals) */
--clr-bg3         /* fondo terciario (inputs, calendarios) */
--clr-border      /* bordes y separadores */
--clr-text        /* texto principal */
--clr-muted       /* texto secundario / labels */
--clr-accent      /* color de acento primario */
--clr-accent2     /* color de acento secundario */
--clr-success     /* verde positivo */
--clr-danger      /* rojo negativo */
--clr-warn        /* amarillo advertencia */
--clr-purple      /* acento adicional */
--clr-nav-bg      /* fondo de la barra de navegación */
--clr-card        /* fondo de tarjetas */
--clr-input       /* fondo de inputs */
--clr-hover       /* fondo en estado hover */
--card-radius     /* radio de bordes de tarjetas */
--font-display    /* tipografía para títulos y KPIs */
--font-mono       /* tipografía monoespaciada */
--font-body       /* tipografía de cuerpo */
--nav-height      /* altura de la barra de navegación */
--shadow-card     /* sombra de tarjetas */
--shadow-accent   /* sombra con color de acento */
--shadow-modal    /* sombra de modales y paneles */
--glow-text       /* text-shadow para temas con glow */
--border-top-bar  /* gradiente decorativo superior */
```

---

## Modelo de datos

### Transaction
```js
{
  id:     number,   // autoincremental desde State.txIdCounter
  type:   'gasto' | 'ingreso',
  date:   'YYYY-MM-DD',
  desc:   string,
  amount: number,   // siempre positivo
  cat:    string,   // id de CATS[]
  freq:   'único' | 'diario' | 'semanal' | 'mensual' | 'anual',
  notes:  string
}
```

### CalEvent
```js
{
  id:     number,
  date:   'YYYY-MM-DD',
  type:   'vencimiento' | 'cobro' | 'recordatorio' | 'gasto',
  desc:   string,
  amount: number
}
```

### Debt
```js
{
  id:        number,
  name:      string,
  total:     number,    // monto original
  remaining: number,    // saldo pendiente
  cuotas:    number,
  interest:  number,    // tasa mensual en %
  due:       'YYYY-MM-DD',
  paid:      number     // total pagado
}
```

### BudgetItem
```js
{
  id:     number,
  name:   string,
  amount: number,   // presupuestado
  actual: number,   // gasto real (manual o auto-calculado si cat asignada)
  cat:    string    // categoría de transacciones para auto-cálculo ('' = manual)
}
```

> `cat` fue agregado para la feature de auto-cálculo del presupuesto. Si `cat` está asignado, el campo "Real" se calcula automáticamente sumando transacciones del mes/año del presupuesto con esa categoría, y se muestra bloqueado con `🔒`.

### Goal (objetivo simple — dentro de Presupuesto)
```js
{
  id:     number,
  name:   string,
  target: number,
  saved:  number
}
```

### Saving (alcancía — sección Ahorro)
```js
{
  id:        number,
  name:      string,
  goal:      number,
  dueDate:   'YYYY-MM-DD',
  notes:     string,
  saved:     number,
  deposits:  [{ date, amount, note }],
  createdAt: 'YYYY-MM-DD'
}
```

### User (State.user)
```js
{
  name:        string,
  email:       string,
  picture:     string | undefined,  // URL foto de Google
  avatar:      string,              // inicial para fallback
  provider:    'google' | 'local' | 'demo',
  supabaseId?: string               // UUID de Supabase Auth
}
```

---

## Flujo de autenticación

```
window.load
  ├── initSupabase() → crea cliente sb
  ├── sb.auth.onAuthStateChange() → si session → enterApp()
  └── sb.auth.getSession() → si session activa → enterApp() directamente
        └── sin sesión Supabase → localStorage['fc-session'] → enterApp()

enterApp(user)
  ├── Guard: si ya inicializado → return
  ├── Guarda en State.user
  ├── localStorage['fc-session'] = user
  ├── Oculta #authScreen, muestra #appShell
  └── initApp() → loadState() → showSection('dashboard')
```

---

## Presupuesto: auto-cálculo del campo "Real"

Si un BudgetItem tiene `cat` asignado (no vacío):
- `getItemActual(type, item, year, month)` suma todas las transacciones del mes/año cuya `cat === item.cat` y cuyo `type` corresponde ('income' → 'ingreso', 'fixed'/'variable' → 'gasto')
- El campo "Real" se muestra como `🔒 $xxx` (read-only, estilo verde)
- El dropdown de categoría se muestra en cada fila

Si `cat` está vacío:
- El campo "Real" es editable normalmente (`<input>`)

### Navegador de mes del presupuesto
- Encabezado: `◀ Marzo 2026 ▶`
- State: `State.budgetYear` / `State.budgetMonth`
- Funciones: `budgetPrevMonth()`, `budgetNextMonth()`
- El auto-cálculo filtra por el mes/año seleccionado en el navegador

---

## Cálculo de cuotas (interés compuesto)

Fórmula de amortización francesa (cuota fija):

```
       r × (1 + r)^n
C = D × ─────────────
        (1 + r)^n - 1
```

Si tasa = 0%: `C = D / n`. Implementado en `calcPayment()` y `renderDebts()`.

---

## Proyección anual

Array de variación estacional aplicado a los totales del presupuesto:

```js
const variation = [0.9, 0.92, 1, 1.02, 0.95, 1.05, 1.1, 0.98, 1, 1.02, 1.15, 1.3];
//                 Ene  Feb  Mar Abr   May   Jun  Jul  Ago  Sep  Oct  Nov  Dic
```

Meses pasados/actual → datos reales de transacciones. Meses futuros → proyección desde presupuesto × variación.

---

## Gráficos (Chart.js)

| Canvas ID     | Tipo     | Sección     | Descripción |
|---------------|----------|-------------|-------------|
| `chartPie`    | doughnut | Dashboard   | Distribución de gastos por categoría |
| `chartBar`    | bar      | Dashboard   | Ingresos vs gastos últimos 6 meses |
| `chartDaily`  | bar      | Gastos      | Gasto diario del mes actual |
| `chartBudget` | bar      | Presupuesto | Presupuestado vs real por ítem |
| `chartAnual`  | line     | Anual       | Proyección 12 meses (ingresos, egresos, saldo) |

Todos usan `makeChart()` / `destroyChart()` para evitar memory leaks.

---

## Funciones de main.js (por módulo)

| Módulo | Funciones |
|--------|-----------|
| Utils | `fmt`, `today`, `getMonthTxs`, `calcTotals`, `showToast`, `makeChart`, `destroyChart`, `chartDefaults` |
| Theme | `setTheme`, `setMode`, `toggleMode`, `openThemePanel`, `closeThemePanel` |
| Auth | `initSupabase`, `loginWithGoogle`, `handleGoogleCredential`, `registerWithEmail`, `loginWithEmail`, `enterApp`, `switchAuthTab`, `loginDemo`, `logout`, `toggleUserMenu`, `showAuthError`, `clearAuthErrors` |
| App Init | `initApp`, `loadState`, `saveState`, `showSyncStatus` |
| Navigation | `showSection`, `rerenderCurrentSection`, `renderSection`, `toggleSidebar` |
| Dashboard | `renderDashboard`, `isThisMonth`, `renderTxRows` |
| Gastos | `renderGastos`, `clearFilters` |
| Transactions | `openTxModal`, `closeTxModal`, `saveTx`, `deleteTx` |
| Presupuesto | `renderBudget`, `renderBudgetSection`, `getItemActual`, `addBudgetRow`, `removeBudgetRow`, `updateBudgetName`, `updateBudgetAmt`, `updateBudgetCat`, `budgetPrevMonth`, `budgetNextMonth`, `renderBudgetSummary`, `renderBudgetChart`, `renderGoals`, `addGoal`, `removeGoal`, `updateGoalSaved`, `saveBudgetFeedback` |
| Calendario | `renderCalendar`, `prevMonth`, `nextMonth`, `openCalModal`, `closeCalModal`, `saveCalEvent`, `deleteCalEvent` |
| Deudas | `renderDebts`, `openDebtModal`, `closeDebtModal`, `saveDebt`, `deleteDebt`, `payDebt`, `calcPayment` |
| Ahorro | `renderSavings`, `openSavingModal`, `closeSavingModal`, `saveSaving`, `deleteSaving`, `addDeposit` |
| Anual | `setYear`, `renderAnual` |
| Chips | `setChipActive` |
| Export | `exportData` |
| Startup | IIFE (tema anti-flash), `window.onload` (sesión + Google) |

---

## Convenciones de código

| Convención | Descripción |
|---|---|
| `State.*` | Todo el estado mutable vive en el objeto global `State` |
| `render*()` | Funciones que actualizan el DOM. Siempre idempotentes |
| `open/close*Modal()` | Abrir/cerrar overlays de modales |
| `save*()` | Leer formulario, validar, pushear al State, cerrar modal, toast, re-render |
| `delete*(id)` | Filtrar el array del State, re-render |
| `fmt(n)` | Formatea número como `$1.234.567` (locale es-AR) |
| `fc-*` | Prefijo de todas las claves de `localStorage` del proyecto |

---

## LocalStorage keys

| Clave | Tipo | Descripción |
|---|---|---|
| `fc-theme` | string | Tema activo: `'windows'`, `'cyber'`, etc. |
| `fc-mode` | string | Modo: `'dark'` o `'light'` |
| `fc-session` | JSON | Usuario logueado (`User` object, sin supabaseId) |
| `fc-data-{email}` | JSON | Todos los datos del usuario (State serializado) |

---

## Temas visuales

| Tema | Estado | Default |
|------|--------|---------|
| `windows` | ✅ Completo dark + light | ✅ **`windows` / `light`** |
| `cyber` | ✅ Completo | — |
| `future` | ✅ Completo | — |
| `scifi` | ✅ Completo | — |
| `gold` | ✅ Completo | — |
| `mac` | ⚠️ Parcial | — |

El default se define en 4 lugares del código: `State`, `initApp()` fallback, IIFE anti-flash, y `<html data-theme="windows" data-mode="light">`.

---

## Responsive breakpoints

| Breakpoint | Cambios |
|---|---|
| `≤ 1200px` | Sidebar de publicidad oculto |
| `≤ 1024px` | KPI grid pasa a 2 columnas; grid-3 pasa a 2 columnas |
| `≤ 768px` | Nav central se oculta, aparece hamburger; grids a 1 columna |
| `≤ 600px` | Banners de ad con min-height reducida |
| `≤ 480px` | KPI grid a 1 columna; padding reducido |

---

## Checklist para nuevo desarrollo

- [ ] ¿El nuevo estado va en `State`?
- [ ] ¿La función de render es idempotente?
- [ ] ¿Los gráficos nuevos usan `makeChart()` / `destroyChart()`?
- [ ] ¿Los estilos van en `style.css` (sin `style=` inline en HTML)?
- [ ] ¿Las nuevas claves de `localStorage` usan el prefijo `fc-`?
- [ ] ¿Los nuevos BudgetItems incluyen el campo `cat: ''`?
- [ ] ¿Se corrió `node --check main.js`?
- [ ] ¿Se actualizó `STRUCTURE.md` y `FINCONTROL_MASTER.md`?