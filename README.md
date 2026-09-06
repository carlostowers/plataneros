# Plataneros de Corozal — Performance OS

App de bienestar y monitoreo de carga para los **Plataneros de Corozal**, equipo profesional de la Liga de Voleibol Superior Masculina de Puerto Rico (LVSM).

Es una PWA de un solo archivo (`index.html`, HTML + CSS + JS, sin build step) con backend en [Supabase](https://supabase.com) (Postgres + Auth + RLS).

## Qué hace

- **Modo jugador** (enlace personal por token, sin login): check-in diario de bienestar (sueño, dolor muscular, estrés, ánimo, fatiga), carga interna por sesión (RPE × duración), reporte de sobreuso (OSTRC-O), y seguimiento de peso corporal.
- **Modo cuerpo técnico** (login con email/contraseña): panel de carga y ACWR por jugador y equipo, disponibilidad, calendario de práctica/torneos, alertas de riesgo, tapering pre-torneo, exportes (reporte diario/semanal), roster.

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

- 10 tablas (`players`, `entries`, `team_settings`, `practice_days`, `tournaments`, `availability`, `body_weight`, `overuse_reports`, `entry_edits`, `backups`) con Row Level Security activo.
- Funciones RPC `SECURITY DEFINER` para el modo jugador: `player_bootstrap`, `player_submit_checkin`, `player_submit_overuse`, `player_mark_welcomed`. El anon key nunca puede leer/escribir las tablas directamente; todo pasa por estas funciones validando el token.
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
- Los números de camiseta del roster están pendientes de confirmación por el cuerpo técnico (aparecen como "S/N").
