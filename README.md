# Plataneros de Corozal — Performance OS

App de bienestar y monitoreo de carga para los **Plataneros de Corozal**, equipo profesional de la Liga de Voleibol Superior Masculina de Puerto Rico (LVSM). Diseñada para una temporada profesional real: dos partidos por semana, sin torneos de fin de semana ni "modo torneo" manual.

Es una PWA de un solo archivo (`index.html`, HTML + CSS + JS, sin build step) con backend en [Supabase](https://supabase.com) (Postgres + Auth + RLS).

## Qué hace

- **Modo jugador** (enlace personal por token, sin login): check-in diario de bienestar (sueño, dolor muscular, estrés, ánimo, fatiga, motivación, síntomas de enfermedad), y hasta **dos registros de entrenamiento separados el mismo día** — gimnasio (físico) y cancha (técnico/táctico) — cada uno con su propio RPE × duración, ya que en formato profesional un día puede tener sesión de gym primero y práctica de cancha después. El bienestar se contesta una sola vez al día y se comparte entre ambos registros. También: check-in de partido, recuperación post-partido, reporte semanal de sobreuso (OSTRC-O), y seguimiento de peso corporal.
- **Modo cuerpo técnico** (login con email/contraseña): panel de carga y ACWR por jugador y equipo, disponibilidad, **calendario de Partidos** (calendario real de la LVSM: rival, sede, jornada, resultado — no un torneo de fin de semana, totalmente editable desde el app), página **Game Day** con estado del roster de cara al próximo partido (siempre visible, ya no depende de activar un "modo torneo" a mano), página **Microciclo** con el plan de la semana en curso (gimnasio/cancha/partido/descanso día por día, cuánto del roster ya registró cada uno), un **panel de periodización física** que muestra automáticamente, según la fecha de hoy, la semana (1 a 20), la fase, el objetivo físico principal, el % de intensidad recomendado y el objetivo del período — tomado de la planificación de 20 semanas del cuerpo técnico (Coach Freddy Vázquez, LVSM 2K26-27, del 1 de septiembre 2026 al 16 de enero 2027) — una **plantilla semanal recurrente** editable desde el mismo Microciclo (qué días tocan gimnasio/cancha por defecto — actualmente martes, viernes y domingo — aplicada automáticamente semana tras semana) con posibilidad de **corregir un día puntual** (viaje, feriado, cambio de última hora) tocando su tarjeta sin desarmar el patrón general, y un **taper corto (3–5 días) que solo aparece cuando hay un partido de playoffs/final cerca** — los partidos de jornada regular no disparan taper, se manejan con el microciclo semanal, alertas de riesgo, exportes (reporte diario/semanal), roster con teléfono y recordatorio directo por WhatsApp.

## Estructura del repositorio

```
index.html        La app completa (single-file)
manifest.json      Manifest de la PWA
icons/              Íconos de la PWA (192, 512, maskable, apple-touch, favicon)
favicon.ico
vercel.json         Config de despliegue en Vercel (headers de manifest/íconos)
```

No hay `package.json` ni build step: es HTML estático servido tal cual.

## Backend (Supabase)

La app se conecta a un proyecto de Supabase propio de Plataneros (URL + anon key ya están en el `<script>` de `index.html`, sección `SUPABASE`). Ese proyecto tiene:

- 10 tablas (`players`, `entries`, `team_settings`, `practice_days`, `tournaments`, `availability`, `body_weight`, `overuse_reports`, `entry_edits`, `backups`) con Row Level Security activo. La tabla `tournaments` sigue llamándose así por compatibilidad con las funciones RPC ya desplegadas, pero ahora almacena el calendario real de partidos de la LVSM (no torneos escolares). `entries.session_label` distingue `fisico` (gimnasio), `tecnica`/`scrimmage` (cancha), `partido` y `recuperacion`; `entries.illness` guarda el autorreporte diario de síntomas. `team_settings.training_template` (jsonb) guarda la plantilla semanal por defecto (día de la semana → gimnasio/cancha); una fila en `practice_days` para una fecha exacta siempre le gana a esa plantilla, incluso en `gym:false, court:false` (un descanso a propósito).
- Funciones RPC `SECURITY DEFINER` para el modo jugador: `player_bootstrap`, `player_submit_checkin` (ahora reemplaza solo el registro de la misma fecha **y el mismo tipo de sesión**, para no borrar el gimnasio del día al guardar la cancha, o viceversa), `player_submit_overuse`, `player_mark_welcomed`. El anon key nunca puede leer/escribir las tablas directamente; todo pasa por estas funciones validando el token.
- `make_backup()` para respaldos manuales.

**Pendiente de configurar en el Dashboard de Supabase** (no se puede hacer por SQL de forma segura): crear el usuario de acceso del cuerpo técnico en *Authentication → Add User* (email + contraseña).

Si se necesita rotar credenciales o apuntar a otro proyecto de Supabase, se edita directamente el bloque `const sb = window.supabase.createClient(...)` en `index.html`.

## Desplegar en Vercel

1. Importar este repositorio de GitHub en [vercel.com/new](https://vercel.com/new).
2. Framework preset: **Other** (sitio estático, sin build command, sin output directory — Vercel sirve `index.html` en la raíz tal cual).
3. Deploy. No hace falta ninguna variable de entorno: las credenciales de Supabase están embebidas en el cliente (son públicas por diseño — la seguridad real la da RLS + las funciones RPC en la base de datos).

## Desarrollo local

No requiere instalar nada. Basta con abrir `index.html` en un navegador, o servirlo con cualquier servidor estático, por ejemplo:

```bash
python3 -m http.server 8080
```

y visitar `http://localhost:8080`.

## Notas

- Los enlaces personales de jugador llevan el token en el fragmento de la URL (`#token`), nunca en la ruta ni en query string visible a servidores/proxies, para que no quede en logs.
- El manifest de la PWA se inyecta por JavaScript solo cuando **no** hay token de jugador en la URL, para que "Añadir a pantalla de inicio" en modo jugador no sobreescriba el enlace personal con la URL raíz del sitio.
- Los números de camiseta del roster están pendientes de confirmación por el cuerpo técnico (aparecen como "S/N"). Los teléfonos están confirmados para 13 de los 16 jugadores; Pedro Nieves, Antonio Elías y Samuel Jackman quedan pendientes.
- El panel de periodización física (`PERIODIZATION_PLAN` en `index.html`) es un dato fijo transcrito de la planificación del cuerpo técnico, no una tabla de Supabase — para ajustarlo (otra temporada, otro plan) se edita ese arreglo directamente en el código.
- Formato de temporada: no existe un "modo torneo" que el staff prenda y apague a mano. Game Day y Microciclo se calculan siempre a partir del calendario de Partidos y de los días de gimnasio/cancha marcados, porque con dos partidos por semana esas páginas deben ser útiles todas las semanas.
- Todo el monitoreo actual es autorreportado (subjetivo): no hay integración con wearables, HRV ni pruebas de salto. Si más adelante el equipo adquiere esa tecnología, es la siguiente extensión natural del modelo de carga.
