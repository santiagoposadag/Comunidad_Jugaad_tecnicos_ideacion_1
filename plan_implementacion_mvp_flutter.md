# Plan de Implementación MVP — Red Colaborativa de Transporte Público (Flutter)

## Contexto

La Comunidad Jugaad Técnicos está construyendo una app colaborativa para que pasajeros de transporte público en Medellín compartan en tiempo real la ubicación del bus, nivel de ocupación y reportes de seguridad. El diseño original (Telegram bot + PWA, documentado en `posada-main-design-20260408-185918.md`) fue descartado por limitaciones de tracking continuo y UX fragmentada. La decisión post-investigación (context7 + web) es **Flutter + Supabase + MapLibre**, con tracking GPS en foreground para el MVP y app nativa instalable manualmente en dispositivos Android (APK) e iOS (TestFlight / build directo) de miembros de la comunidad.

Este plan establece la estructura del proyecto, entidades, lógica de negocio, contextos pendientes de refinar, y un roadmap de 7 milestones incrementales que permite empezar la construcción gradual sin bloquear la primera sesión de código por decisiones abiertas que no son críticas para arrancar.

El repo actualmente es greenfield: solo existen dos documentos de diseño en markdown, un `README.md` mínimo, y nada más — ni `.gitignore`, ni `pubspec.yaml`, ni tooling config. Todo el scaffold se construye desde cero.

---

## 1. Stack técnico consolidado

| Capa | Elección | Razón |
|------|----------|-------|
| Framework | **Flutter (Dart)** | Cross-platform, performance GPS-heavy, Tracelet gratis para background v2 |
| Backend | **Supabase** (Postgres + Auth + Realtime + RLS + PostGIS) | Free tier, RLS reemplaza backend custom |
| Mapa | **MapLibre GL** via `maplibre_gl` | Tiles gratuitos, sin Mapbox API keys |
| Tiles dev | **OpenFreeMap** (`https://tiles.openfreemap.org/styles/liberty`) | Sin cuenta, sin límites en dev |
| Tiles prod | **Protomaps PMTiles en Cloudflare R2** | Costo $0, infra propia, offline-friendly |
| GPS | **Foreground only (`geolocator`)** | Elimina review de Google Play por ACCESS_BACKGROUND_LOCATION |
| Realtime posiciones en vivo | **Supabase Broadcast** (canal `route:{id}`) | Efímero, no satura DB ni Postgres Changes |
| Realtime reportes | **Supabase Postgres Changes** | Persistentes, baja frecuencia |
| State management | **Riverpod** (`flutter_riverpod` + codegen) | Providers type-safe, `StreamProvider` para Broadcast, test-friendly |
| Navegación | **go_router** | Oficialmente recomendado por Flutter team |

---

## 2. Estructura de carpetas (feature-first)

```
lib/
  main.dart
  app.dart                          # MaterialApp.router + theme + ProviderScope
  bootstrap.dart                    # runZonedGuarded, dotenv.load, Supabase.initialize
  core/
    config/env.dart                 # SUPABASE_URL, ANON_KEY, TILE_STYLE_URL
    router/app_router.dart          # go_router config
    router/routes.dart              # enum de rutas nombradas
    supabase/supabase_providers.dart
    location/location_service.dart  # wrapper sobre geolocator + permisos
    location/location_providers.dart
    theme/app_theme.dart
    widgets/                        # botones, sheets, toasts reutilizables
    utils/                          # distance calc, throttle, rate limiter
  features/
    auth/
      data/auth_repository.dart
      application/auth_controller.dart
      presentation/splash_page.dart
    map/
      data/live_positions_repository.dart   # Broadcast subscriber
      application/live_positions_controller.dart
      presentation/map_page.dart
      presentation/widgets/bus_marker.dart
      presentation/widgets/occupancy_legend.dart
    routes_catalog/
      data/routes_repository.dart
      domain/route.dart
      domain/stop.dart
      presentation/route_picker_sheet.dart
    session/
      data/session_repository.dart
      domain/active_session.dart
      application/session_controller.dart   # start / heartbeat / stop
      presentation/start_session_sheet.dart
      presentation/active_session_banner.dart
    reports/
      data/reports_repository.dart
      domain/report.dart
      application/reports_controller.dart
      presentation/report_sheet.dart
assets/
  routes/                           # bootstrap GeoJSON (ruta-203.geojson, ...)
  map_styles/                       # fallback offline
supabase/
  migrations/                       # SQL versionado
  seed.sql
test/
  features/<name>/...
```

**¿Por qué feature-first?** El MVP tiene 4 slices verticales (map, session, reports, routes catalog). Un layout feature-first permite que cada milestone envíe código end-to-end sin tocar todo el codebase. `lib/core/` contiene código cross-cutting; `lib/features/<name>/` contiene una vertical.

---

## 3. Dependencias (`pubspec.yaml`)

Versiones aproximadas mid-2026 (verificar con `flutter pub outdated` antes de ejecutar):

| Paquete | Propósito |
|---------|-----------|
| `supabase_flutter: ^2.8.0` | Auth + Postgres + Realtime + Broadcast |
| `flutter_riverpod: ^2.6.0` | State management |
| `riverpod_annotation` + `riverpod_generator` (dev) | Codegen de providers |
| `go_router: ^14.6.0` | Navegación declarativa |
| `maplibre_gl: ^0.21.0` | Mapa vectorial |
| `geolocator: ^13.0.0` | GPS |
| `permission_handler: ^11.3.0` | Permisos runtime |
| `flutter_dotenv: ^5.2.0` | Env vars en dev (híbrido con `--dart-define` en release) |
| `freezed: ^2.5.0` + `json_serializable` | Modelos inmutables |
| `intl: ^0.19.0` | Locale `es_CO` |
| `logger: ^2.4.0` | Logs estructurados |
| `connectivity_plus: ^6.1.0` | Banner offline |
| `shared_preferences` | Onboarding flag, último map view |

Dev: `build_runner`, `custom_lint`, `riverpod_lint`, `flutter_test`, `mocktail`.

**Env handling híbrido:** `core/config/env.dart` usa `const String.fromEnvironment` (release via `--dart-define`) con fallback a `dotenv.env[...]` (dev). Así `.env` nunca entra al APK.

---

## 4. Entidades — Dart + Supabase

### Clases Dart (freezed en `features/<feature>/domain/`)

| Clase | Campos principales |
|-------|--------------------|
| `BusRoute` | `id`, `number`, `name`, `polyline: List<LatLng>`, `stops`, `directions` |
| `BusStop` | `id`, `name`, `lat`, `lng`, `sequence` |
| `ActiveSession` | `id`, `userId`, `routeId`, `direction`, `startedAt`, `lastLat`, `lastLng`, `lastReportedAt`, `occupancy`, `safety` |
| `Report` | `id`, `userId`, `routeId`, `sessionId?`, `type: ReportType`, `value`, `lat`, `lng`, `createdAt` |
| `LivePosition` (payload Broadcast, NO persistido) | `sessionId`, `routeId`, `direction`, `lat`, `lng`, `heading?`, `occupancy`, `safety`, `sentAt` |
| `Profile` | `userId`, `displayName`, `createdAt`, `trustScore` |

### Supabase SQL (`supabase/migrations/0001_init.sql`)

```sql
create extension if not exists postgis;

create table routes (
  id uuid primary key default gen_random_uuid(),
  number text not null,
  name text not null,
  polyline geography(LineString,4326) not null,
  directions text[] not null default array['ida','vuelta'],
  created_at timestamptz default now()
);
create index routes_polyline_gix on routes using gist(polyline);

create table stops (
  id uuid primary key default gen_random_uuid(),
  route_id uuid not null references routes(id) on delete cascade,
  name text not null,
  location geography(Point,4326) not null,
  sequence int not null
);
create index stops_location_gix on stops using gist(location);
create index stops_route_idx on stops(route_id);

create table profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  display_name text,
  trust_score int not null default 0,
  created_at timestamptz default now()
);
create or replace function handle_new_user() returns trigger language plpgsql security definer as $$
begin insert into profiles(user_id) values (new.id); return new; end; $$;
create trigger on_auth_user_created after insert on auth.users
  for each row execute function handle_new_user();

create table active_sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  route_id uuid not null references routes(id),
  direction text not null,
  started_at timestamptz default now(),
  ended_at timestamptz,
  last_location geography(Point,4326),
  last_reported_at timestamptz,
  occupancy text check (occupancy in ('vacio','medio','ocupado')),
  safety text check (safety in ('seguro','inseguro'))
);
create unique index one_active_session_per_user
  on active_sessions(user_id) where ended_at is null;
create index active_sessions_route_live_idx
  on active_sessions(route_id) where ended_at is null;

create table reports (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  route_id uuid not null references routes(id),
  session_id uuid references active_sessions(id) on delete set null,
  type text not null check (type in ('occupancy','safety')),
  value text not null,
  location geography(Point,4326),
  created_at timestamptz default now()
);
create index reports_route_time_idx on reports(route_id, created_at desc);
create index reports_user_time_idx on reports(user_id, created_at desc);
```

**Nota clave:** las ubicaciones en vivo **NO viven en DB**. Viajan por Broadcast `route:{routeId}` con evento `position`. La fila de `active_sessions` guarda solo el último snapshot para clientes que se unen tarde.

---

## 5. Lógica de negocio — dónde vive cada regla

| Concern | Dónde | Cómo |
|---------|-------|------|
| Auth | Cliente | `supabase.auth.signInAnonymously()` en splash. Upgrade a phone OTP post-MVP. |
| Publicar posición | Cliente | Cada 12s con sesión activa: (a) `channel.sendBroadcastMessage`, (b) RPC `update_session_location` que persiste el último punto |
| Consumir posiciones | Cliente | `channel('route:$id')` + prefill inicial desde `active_sessions where ended_at is null` |
| Rate limit de reportes | Postgres | RPC `create_report` con `security definer`, rechaza si hay reportes del mismo usuario en los últimos 60s. Max 1 reporte/min. |
| Una sesión activa por usuario | Postgres | Partial unique index + RPC `start_session` que cierra dangling antes de insertar |
| TTL de posiciones | Scheduled | `pg_cron` cada 2 min: cierra sesiones con `last_reported_at` mayor a 2 min |
| Anti-spoofing | Postgres RPC | `update_session_location` calcula velocidad implícita vs `last_location`; rechaza si > 120 km/h |
| RLS | Postgres | `profiles`: own read/update. `active_sessions`: read públicas no-ended; write solo `user_id = auth.uid()`. `reports`: read público, insert own. `routes`/`stops`: read público, write solo service role |
| Bootstrap catálogo | Admin script | Script Dart one-shot que lee `assets/routes/*.geojson` y hace insert via service role |

---

## 6. Contextos que hay que refinar antes de codear

Estos son los puntos abiertos que deben decidirse (o listarse conscientemente como "diferidos") antes de empezar:

1. **Nombre del app + bundle id** — afecta `flutter create --org ... --project-name ...`. Propuesta: `co.jugaad.transpo`, `transpo_comunidad`. Confirmar.
2. **Método de auth** — recomendado: `signInAnonymously` (cero fricción). Decidir si el path de upgrade a phone OTP va en MVP o queda en v2.
3. **Fuente del catálogo de rutas** — revisar si el Área Metropolitana de Medellín publica GTFS (estático o RT) en 2025-2026. Si sí → script de importación. Si no → trazado manual de 5 rutas piloto en geojson.io (~2h/ruta). **Esto desbloquea M3.**
4. **Lista de 5 rutas piloto** — nombrar las 5 rutas concretas y asignar un owner por ruta (alguien que la use diariamente).
5. **Anonimato vs accountability** — ¿`display_name` se muestra públicamente o todos los reportes son anónimos? Afecta UI de trust-score.
6. **Distribución iOS** — TestFlight ($99/año Apple Dev) vs builds ad-hoc a UDIDs registrados. TestFlight vale la pena si >5 testers iOS.
7. **Signing Android** — keystore upload para APK. ¿Play Internal Testing o APK directo por WhatsApp/Drive al grupo Jugaad?
8. **Tiles de producción** — bounding box del `.pmtiles` de Medellín (sugerencia: Valle de Aburrá, ~6.1–6.4 N, −75.7 a −75.4 W). Se puede generar en paralelo al código.
9. **Telemetría** — ¿Sentry Flutter SDK? Útil para debugging remoto con testers comunitarios.
10. **Copy de privacidad** — texto en español explicando qué se comparte: ubicación foreground mientras hay sesión activa, nada cuando está inactivo. **Requerido antes del primer TestFlight/APK.**

De los 10, los **realmente bloqueantes para empezar M0 son: #1 (naming) y #2 (auth method)**. El resto puede diferirse al milestone donde aplica.

---

## 7. Roadmap gradual (7 milestones shippables)

| # | Milestone | Objetivo | Verificación |
|---|-----------|----------|--------------|
| **M0** | Scaffold | App vacía corre en Android + iOS | `flutter run` en un Android + un iOS muestra la pantalla counter default; `flutter analyze` y `flutter test` pasan |
| **M1** | Shell + Mapa + GPS | Usuario abre app y se ve a sí mismo en el mapa | Caminar afuera con el teléfono → punto azul sigue al usuario. Denegar permiso → fallback a bounding box de Medellín |
| **M2** | Supabase + Auth anónimo + Schema | Cloud live, usuario autenticado | Abrir app fresh → fila aparece en `profiles`. Uninstall+reinstall → nueva fila |
| **M3** | Catálogo de rutas | Usuario puede seleccionar una ruta | Tap "Estoy en un bus" → sheet lista las 5 rutas más cercanas. Seleccionar dibuja polyline en el mapa |
| **M4** | Ciclo de vida de sesión + heartbeat | Un usuario solo puede transmitir su posición | `active_sessions` crece en `last_reported_at` cada 12s. Tap "Bajé del bus" → `ended_at` se setea |
| **M5** | Posiciones en vivo para otros usuarios | Un segundo usuario ve el bus del primero | 2 dispositivos: A transmite, B abre app → ve bus de A moviéndose. Al parar A → marker desaparece en <2 min (TTL) |
| **M6** | Reportes (ocupación + seguridad) | Flujo completo de primer reportero | A reporta "lleno + seguro" → B ve marker actualizado + chip. A intenta otro reporte <60s → error en español |
| **M7** | Pulido + Distribución | Shippable a 10 testers comunitarios | 10 miembros instalan, completan flow una vez, llenan form de feedback |

Cada milestone es shippable e independientemente verificable en un dispositivo real. La sesión de código en la **próxima interacción arranca por M0**, y M0 solo requiere tener resueltos los contextos #1 y #2 del bloque anterior.

---

## 8. Archivos críticos a crear

Greenfield — todos son creaciones nuevas:

- `pubspec.yaml`
- `.gitignore`
- `analysis_options.yaml`
- `lib/main.dart`
- `lib/bootstrap.dart`
- `lib/app.dart`
- `lib/core/config/env.dart`
- `lib/core/router/app_router.dart`
- `lib/core/supabase/supabase_providers.dart`
- `lib/features/session/application/session_controller.dart`
- `supabase/migrations/0001_init.sql`
- `supabase/seed.sql`
- `.env.example`
- `README.md` (actualizar con instrucciones de setup)

---

## 9. Plan de verificación end-to-end

### Automatizada
- `flutter analyze` limpio en cada commit
- Tests unitarios sobre `core/utils` (throttle, distancia geodésica) y repositorios contra Supabase local (`supabase start`)
- GitHub Actions: analyze + test en PR a main

### Matriz manual de dispositivos (mínima)
- 1 Android gama media (2 años atrás, tipo Moto G)
- 1 Android moderno (Pixel / Samsung reciente)
- 1 iOS (iPhone 12+)

Testing en datos móviles reales de Medellín, no solo Wi-Fi.

### Flujos por milestone (ejecutados en bus real cuando aplique)
- **M1:** permiso concedido → ubicación visible; modo avión → punto viejo + banner offline
- **M3:** parado cerca del corredor de una ruta piloto → aparece primera en el picker
- **M4:** un paradero de distancia → `last_location` actualiza cada ~12s; lock screen → heartbeat pausa (esperado en foreground-only)
- **M5:** dos miembros del equipo en la misma ruta, uno transmite, otro observa; medir latencia e2e (target: <3s p95 en 4G)
- **M6:** spam del botón de reporte → segundo intento rechazado con mensaje claro en español. Reportar "inseguro" → marker cambia color en <1s en el observador
- **M7:** viaje completo — usuario A se sube, selecciona ruta, reporta, viaja 10 min, se baja. Usuario B lo sigue desde lejos. Screen capture como regression reference

### Checks en Supabase Studio después de cada sesión
- Sin `active_sessions` huérfanas (pg_cron TTL funcionando)
- `reports` count coincide con taps
- RLS deniega writes cross-user (probar con un segundo usuario anon via SQL)

---

## 10. Próxima interacción — qué se ejecuta primero

1. Confirmar contextos #1 (naming) y #2 (auth method) del bloque 6
2. Ejecutar M0:
   - `flutter create --org <org> --platforms=android,ios --project-name <name> .`
   - `.gitignore` y `analysis_options.yaml`
   - `flutter pub add` de las dependencias del bloque 3
   - Commit inicial "M0: flutter scaffold"
3. Verificar M0: `flutter run` en un dispositivo físico Android (o simulador iOS) → counter app default corre.

Los milestones M1-M7 siguen en orden, cada uno en uno o varios commits, con validación en dispositivo real antes de avanzar al siguiente.

---

## Documento complementario

Ver `conceptos_base_tecnologicos.md` para una introducción desde primeros principios a los elementos tecnológicos que se usan en este plan (Flutter, Supabase, Realtime, RLS, PostGIS, tiles vectoriales, etc.). Ese documento está pensado para nivelar al equipo antes de que empecemos a construir.
