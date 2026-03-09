# ◈ FinControl — Master Reference
> Documento completo para retomar el proyecto en cualquier sesión de Claude AI.
> Última actualización: 8 de marzo 2026

---

## 1. CONTEXTO DEL USUARIO

**Perfil:** Trabajador en Argentina, 2026. Dos hijos (manutención en proceso de reactivación).

**Situación financiera real:**
- Ingresos: ~$550.000/mes (adelantos de sueldo)
- Gastos fijos: ~$425.000/mes
- Saldo libre: ~$125.000/mes
- Deuda crítica activa: EPE (luz) ~$435.801 · TGI ~$144.169
- Manutención reactivada genera déficit adicional de ~$345.000/mes

**Uso principal del proyecto:**
Gestión personal de finanzas para Argentina con inflación alta: control de gastos, deudas, ahorro por objetivos y proyección anual. No es un proyecto comercial, es uso personal directo.

---

## 2. PROYECTO: FinControl

### Deploy
- **URL pública:** `https://codepitter.github.io/FinControl`
- **Repo:** `https://github.com/codePitter/FinControl`
- **Deploy automático:** GitHub Actions (workflow "Deploy FinControl to GitHub Pages")

### Stack
| Capa | Tecnología |
|------|-----------|
| Frontend | HTML5 + CSS3 + JS ES2022 — sin frameworks |
| Gráficos | Chart.js 4.4.1 (CDN) |
| Auth | Supabase Auth (email/password + Google OAuth) |
| Base de datos | Supabase (PostgreSQL, tabla `user_data` con JSONB) |
| Persistencia offline | localStorage |
| Deploy | GitHub Pages |

### Estructura de archivos
```
FinControl/
├── index.html          ← Toda la UI (797 líneas)
├── css/style.css       ← Estilos (1105 líneas)
├── js/main.js          ← Toda la lógica (1834 líneas, 102 funciones)
├── js/config.js        ← Credenciales (gitignored)
├── js/config.example.js
├── fc-rescue.js        ← Script de rescate para consola del navegador
├── supabase_schema.sql
├── .gitignore
├── README.md
└── STRUCTURE.md
```

> ⚠️ **Regla de trabajo:** Siempre subir los 3 archivos (main.js, index.html, style.css) al inicio de cada sesión de Claude. Los outputs de Claude no persisten entre sesiones.

---

## 3. CREDENCIALES (NO subir al repo)

```js
// js/config.js
window.APP_CONFIG = {
  supabaseUrl:    'https://vqlbxuoowzgnlqyahink.supabase.co',
  supabaseKey:    'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InZxbGJ4dW9vd3pnbmxxeWFoaW5rIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzI2ODU5NzMsImV4cCI6MjA4ODI2MTk3M30.CC-JUlO662O46HRuR75Ia9AUc8GrusyQ00aIZDoQYvk',
  googleClientId: '175409993499-v38h85aqn2of4qqem2dfrl2ahu19rmfq.apps.googleusercontent.com',
};
```

**Supabase dashboard:** `supabase.com/dashboard/project/vqlbxuoowzgnlqyahink`

---

## 4. BASE DE DATOS SUPABASE

### Tabla `user_data` ✅ (creada y verificada)
```sql
create table user_data (
  id         uuid primary key references auth.users(id) on delete cascade,
  data       jsonb not null default '{}',
  updated_at timestamptz not null default now()
);
-- RLS habilitado
-- 4 policies: select/insert/update/delete por auth.uid() = id
-- Trigger: actualiza updated_at automáticamente
```

**Verificar:** `select id from user_data limit 1;` → "Success. No rows returned" es correcto (tabla vacía).

---

## 5. ARQUITECTURA DE LA APP

### State (objeto global central)
```js
const State = {
  // Usuario
  user: null,           // { name, email, picture?, avatar, provider, supabaseId? }

  // UI
  theme: 'windows',     // 'windows' | 'cyber' | 'future' | 'scifi' | 'gold' | 'mac'
  mode: 'dark',         // 'dark' | 'light'
  currentSection: 'dashboard',
  dashPeriod: 'month',  // 'month' | 'week' | 'year'

  // Calendarios
  calYear: <año actual>,
  calMonth: <mes actual>,
  anualYear: 2026,

  // Contadores de ID
  txIdCounter: 1,
  calIdCounter: 1,
  debtIdCounter: 1,
  budgetIdCounter: 10,
  goalIdCounter: 1,
  savingIdCounter: 1,

  // Datos
  transactions: [],     // [{ id, type, date, desc, amount, cat, freq, notes }]
  calEvents: [],        // [{ id, date, type, desc, amount }]
  debts: [],            // [{ id, name, total, remaining, cuotas, interest, due, paid }]
  budget: {
    income:   [],       // [{ id, name, amount, actual }]
    fixed:    [],       // [{ id, name, amount, actual }]
    variable: []        // [{ id, name, amount, actual }]
  },
  goals: [],            // [{ id, name, target, saved }]  ← objetivos simples (slider)
  savings: [],          // [{ id, name, goal, dueDate, notes, saved, deposits[], createdAt }]
}
```

### Persistencia (doble capa)
```
saveState()
  ├── localStorage['fc-data-{email}']  ← SIEMPRE (offline first)
  └── Supabase user_data WHERE id = supabaseId  ← si hay sesión activa

loadState()
  ├── provider === 'local' → solo localStorage
  ├── supabaseId disponible → Supabase primero
  │     └── PGRST116 (no rows) → caer a localStorage
  └── fallback → localStorage
```

**localStorage keys:**
- `fc-theme` — tema activo
- `fc-mode` — dark/light
- `fc-session` — objeto usuario serializado (sin supabaseId)
- `fc-data-{email}` — todos los datos del usuario

### Flujo de autenticación
```
window.load
  ├── initSupabase() → crea cliente sb
  ├── sb.auth.onAuthStateChange() → si session → enterApp()
  └── sb.auth.getSession() → si session activa → enterApp() directamente
        └── sin sesión Supabase → localStorage['fc-session'] → enterApp()

enterApp(user)
  ├── Guarda en State.user
  ├── localStorage['fc-session'] = user (sin supabaseId)
  ├── Oculta #authScreen, muestra #appShell
  └── initApp() → loadState() → showSection('dashboard')
```

---

## 6. SECCIONES DE LA APP

| Sección | ID | Función render | Descripción |
|---------|-----|---------------|-------------|
| Dashboard | `sec-dashboard` | `renderDashboard()` | KPIs, alertas, gráfico torta, barras 6 meses, regla 50/30/20, transacciones recientes |
| Gastos | `sec-gastos` | `renderGastos()` | Listado con filtros por mes/categoría/tipo, totales |
| Presupuesto | `sec-presupuesto` | `renderBudget()` | Tabla editable ingresos/fijos/variables + resumen + objetivos + gráfico |
| Calendario | `sec-calendario` | `renderCalendar()` | Calendario mensual + eventos financieros + próximos vencimientos |
| Deudas | `sec-deudas` | `renderDebts()` | Gestión de deudas con calculadora de cuotas |
| Ahorro | `sec-ahorro` | `renderSavings()` | Alcancías con progreso, depósitos y fechas objetivo |
| Anual | `sec-anual` | `renderAnual()` | Proyección 12 meses con datos reales + presupuesto como base |

---

## 7. CÓMO FUNCIONA EL PRESUPUESTO

El presupuesto es **un plan mensual estático** separado del registro de transacciones reales.

### Estructura
Tiene 3 categorías editables:
- **Ingresos** — lo que esperás cobrar
- **Gastos Fijos** — alquiler, servicios, cuotas (importes constantes)
- **Variables / Estimados** — alimentación, transporte, ocio (importes aproximados)

Cada ítem tiene dos campos: **Pres.** (presupuestado) y **Real** (lo que realmente ocurrió).

### Flujo de uso
1. Al principio del mes: completás los campos "Pres." con los valores esperados
2. Durante el mes: actualizás "Real" con lo que realmente gastaste/cobraste
3. El **Resumen** muestra: `Real / Presupuestado` para cada categoría y el saldo resultante
4. El gráfico "Presupuestado vs Real" compara ambas columnas visualmente
5. Botón "💾 Guardar" → `saveState()`

### Relación con otras secciones
- **Vista Anual:** usa `budget.income`, `budget.fixed`, `budget.variable` como baseline de proyección para meses futuros sin transacciones reales
- **Regla 50/30/20** del Dashboard: usa los ingresos reales de transacciones (no del presupuesto)
- **Objetivos de Ahorro** (slider simple): están dentro del presupuesto como complemento visual, son distintos a las Alcancías de la sección Ahorro

### Importante: dos sistemas de ahorro
| | Objetivos (Presupuesto) | Alcancías (Ahorro) |
|--|------------------------|--------------------|
| Ubicación | Sección Presupuesto | Sección Ahorro |
| Control | Slider manual | Depósitos con fecha |
| Fecha objetivo | No | Sí |
| Historial | No | Sí (últimos movimientos) |
| Dashboard | No | Sí (KPI + alerta) |

---

## 8. TEMAS VISUALES

| Tema | Estilo | Fuentes | Estado |
|------|--------|---------|--------|
| `windows` | Windows 11 Fluent Design, acrylic blur | Inter / Share Tech Mono | ✅ **Default** |
| `cyber` | Cyberpunk, magenta/cian, scanlines | Orbitron / Share Tech Mono | ✅ |
| `future` | Futurista, azul profundo, HUD dots | Exo 2 / Share Tech Mono | ✅ |
| `scifi` | Sci-Fi, verde alienígena, hexágonos | Syncopate / Share Tech Mono | ✅ |
| `gold` | Sofisticado, dorado/negro | Cinzel / Cormorant Garamond | ✅ |
| `mac` | macOS style | (pendiente CSS completo) | ⚠️ Parcial |

Cada tema tiene modo **dark** y **light** (10 combinaciones totales).

---

## 9. CATEGORÍAS DE TRANSACCIONES

```
vivienda · servicios · transporte · alimentacion · salud
educacion · hijos · ocio · ropa · mascotas · impuestos · ahorro · deuda · otros
```

---

## 10. FEATURES ESPECIALES

### Math Input
Los campos de monto aceptan expresiones matemáticas:
- `550000 / 4` → evalúa a 137500
- `10000 + 5000 * 2` → evalúa a 20000
- Preview en tiempo real mientras escribís
- Se resuelve al perder el foco o presionar Enter

### Calculadora flotante
Botón azul FAB (bottom-right) → calculadora accesible desde cualquier sección.

### Pago de deudas integrado
Al registrar un pago en Deudas → se crea automáticamente una transacción de gasto con categoría `deuda`, impacta en el saldo del Dashboard.

### Vista Anual inteligente
- Meses pasados/actual: usa transacciones reales
- Meses futuros: proyecta desde el presupuesto con variaciones estacionales predefinidas
- Tabla detallada con totales anuales

### Dot de sync
Pequeño indicador verde/naranja en el nav que muestra si el último guardado fue exitoso en Supabase.

---

## 11. FIXES APLICADOS (historial)

| Fix | Descripción |
|-----|-------------|
| Fix 1 | `getSupabaseUid()` evita round-trip a `getUser()` usando `State.user.supabaseId` |
| Fix 2 | `onAuthStateChange` pasa `supabaseId: u.id` directamente |
| Fix 3 | Guard en `enterApp()` evita re-inicialización múltiple |
| Fix 4 | `PGRST116` ignorado en `loadState()` (primera vez, sin fila aún) |
| Fix 5 | `showSyncStatus()` — dot visual de estado de sync |
| Fix 6 | `getSession()` llama `enterApp()` directamente en vez de esperar `onAuthStateChange` |
| Fix 7 | Usuarios `provider: 'local'` saltan Supabase en `loadState()` |
| Fix 8 | Anti-flash: auth screen oculto antes de pintar si hay sesión en localStorage |
| Fix 9 | Sidebar: eliminado `toggleSidebar()` redundante en botones del menú |
| Fix 10 | `overflow-y: scroll` en html para scrollbar siempre visible |

---

## 12. SCRIPT DE RESCATE (consola del navegador)

Si los datos no cargan, ejecutar `fc-rescue.js` en la consola:
1. Abrí la app → logueate
2. F12 → Console
3. Pegá el contenido de `fc-rescue.js` → Enter

El script: lee localStorage → aplica al State → actualiza UI → sube a Supabase.

---

## 13. MEJORAS PROPUESTAS

### 🔴 Críticas (impacto directo en el uso diario)

**A. Presupuesto mensual con historial**
El presupuesto actual es estático — al cambiar de mes se pierden los valores del mes anterior. Falta:
- Guardar snapshot del presupuesto por mes (key: `YYYY-MM`)
- Comparativa mes anterior vs actual
- Autocompletar con valores del mes anterior como punto de partida

**B. Manutención hijos como módulo**
Dado el contexto del usuario (2 hijos, manutención reactivada), un módulo dedicado:
- Monto acordado / pagado / pendiente por mes
- Alertas de vencimiento
- Registro de pagos parciales
- Historial por hijo

**C. Importar/exportar datos**
`exportData()` existe en el código pero no se usa. Conectar con:
- Botón de exportar JSON completo
- Importar desde JSON (backup manual)
- Esto es crucial dado el riesgo de perder datos en localStorage

**D. Conectar presupuesto con transacciones reales**
Actualmente el campo "Real" del presupuesto se llena manualmente. Debería:
- Auto-calcular "Real" sumando transacciones del mes por categoría
- Mostrar diferencia presupuestado vs real automáticamente

### 🟡 Importantes

**E. Cuotas de deuda en presupuesto**
Las cuotas calculadas en Deudas no aparecen en el presupuesto. Deberían sugerirse automáticamente como gasto fijo.

**F. Ajuste por inflación en proyección anual**
La vista Anual usa variaciones estacionales hardcodeadas (array `variation`). Para Argentina debería permitir ingresar un % de inflación mensual esperado.

**G. Notificaciones / recordatorios**
- Alertas de vencimiento de deudas
- Recordatorio de alcancías próximas a vencer
- Aviso cuando se supera el presupuesto en una categoría

**H. Modo offline completo (PWA)**
Agregar `manifest.json` y un Service Worker básico para que funcione sin internet y se pueda instalar en el celular como app.

**I. Categoría "Hijos" más visible**
Dado el contexto: filtros rápidos por categoría `hijos`, resumen mensual de gastos en hijos en el Dashboard.

### 🟢 Mejoras de UX

**J. Agregar transacción rápida desde Dashboard**
El botón "+ Agregar" lleva a un modal. Podría tener campos precargados con la fecha de hoy y los tipos más usados del usuario.

**K. Presupuesto: campo "Real" auto-read-only con opción override**
Si se conecta con transacciones reales, el campo "Real" debería mostrarse calculado automáticamente con opción de override manual.

**L. Búsqueda en transacciones**
En Gastos solo hay filtros. Falta un campo de búsqueda por descripción.

**M. Gráfico de deuda en el tiempo**
Proyección de cuándo quedarán saldadas las deudas actuales si se paga la cuota sugerida.

**N. Tema `mac` completo**
El tema macOS tiene tokens pero le falta la definición completa dark/light similar al tema `windows`.

---

## 14. INSTRUCCIONES PARA CLAUDE (próxima sesión)

Al inicio de cada sesión:
1. El usuario sube `main.js`, `index.html`, `style.css`
2. Claude **siempre trabaja sobre los archivos subidos** como base
3. Nunca asumir que outputs de sesiones anteriores están disponibles
4. Verificar con `node --check main.js` antes de entregar
5. Siempre entregar los 3 archivos aunque solo se modificó uno

**Comandos de verificación rápida:**
```bash
node --check main.js
grep -c "function " main.js          # debe ser ~102+
grep "savings: \[\]" main.js         # debe existir
grep "sec-ahorro" index.html         # debe existir
grep "saving-card" style.css         # debe existir
```

**Estado esperado de los archivos correctos:**
- `main.js`: ~1834+ líneas, ~102+ funciones, sin errores de sintaxis
- `index.html`: ~797+ líneas, incluye secciones: dashboard, gastos, presupuesto, calendario, deudas, ahorro, anual
- `style.css`: ~1105+ líneas, incluye temas: cyber, future, scifi, gold, windows, mac