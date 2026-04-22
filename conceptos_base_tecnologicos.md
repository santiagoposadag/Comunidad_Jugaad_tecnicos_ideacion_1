# Conceptos Base Tecnológicos — Previos al Desarrollo

## Para qué sirve este documento

Antes de ejecutar el plan de implementación del MVP (`plan_implementacion_mvp_flutter.md`), el equipo de la Comunidad Jugaad Técnicos debe compartir un vocabulario común sobre los elementos tecnológicos que vamos a usar. Este documento introduce cada concepto **desde primeros principios**, explica **por qué lo elegimos para este proyecto**, y sugiere **qué aprender** para contribuir al código.

No es un tutorial exhaustivo. Es un mapa — una vez que identifiques qué bloques no conoces, usa los recursos listados para profundizar. La idea es que nadie del equipo llegue a la primera sesión de código sin saber qué significan las palabras "Flutter", "Supabase Realtime", "RLS" o "tile vectorial".

Los conceptos están ordenados de más generales a más específicos. Al final hay un cuestionario de auto-evaluación para saber si estás listo para M0.

---

## 1. Desarrollo móvil: nativo, cross-platform y PWA

### Qué es
Una app móvil puede construirse de tres formas fundamentalmente distintas:

- **Nativo:** código específico para cada sistema operativo. iOS se escribe en Swift, Android en Kotlin. Dos codebases separados, dos equipos o una persona escribiendo todo dos veces.
- **Cross-platform:** un solo código que corre en ambos sistemas. Ejemplos: **Flutter** (Dart), **React Native** (JavaScript), Kotlin Multiplatform.
- **PWA (Progressive Web App):** es una página web que el navegador puede "instalar" como si fuera app. No pasa por App Store ni Play Store. Corre dentro del navegador con algunas capacidades extra (caché offline, notificaciones limitadas).

### Por qué Flutter para este proyecto
Nuestro app necesita acceder a GPS continuamente, renderizar un mapa fluido con múltiples marcadores en movimiento, y correr en Android + iOS. Evaluamos:

- **PWA** no alcanza: iOS suspende el tab del navegador al cambiar de app → perderíamos el tracking.
- **React Native** funciona pero su puente JavaScript ↔ nativo tiene overhead; además, para tracking en background necesitaríamos pagar una licencia ($400 USD) de `react-native-background-geolocation`.
- **Flutter** compila a código ARM nativo (sin puente), tiene mejor rendimiento gráfico, y en 2026 salió **Tracelet**, un plugin gratis y open-source de tracking en background que resuelve el escenario post-MVP sin licencias.

### Qué aprender
- Qué es un "framework cross-platform" y por qué existe
- Ciclo de vida de una app móvil (foreground, background, terminated)
- Diferencias de permisos entre Android e iOS (especialmente location)

**Recursos sugeridos:**
- [Flutter showcase](https://flutter.dev/showcase) — para ver apps reales hechas en Flutter
- [Mobile app architecture 101 (Google)](https://developer.android.com/topic/architecture) — conceptos que aplican también a Flutter

---

## 2. Flutter y Dart

### Qué es
**Flutter** es un framework creado por Google que permite construir interfaces para móvil, web y escritorio con un solo código. Lo escribes en **Dart**, un lenguaje diseñado también por Google, fuertemente tipado (como Java o TypeScript), con sintaxis familiar para quien sabe JavaScript, Java o C#.

Todo en Flutter es un **Widget** — botones, textos, pantallas, layouts. Los widgets se componen como árboles: un `Scaffold` contiene un `AppBar` y un `Body`, el `Body` contiene un `Column`, el `Column` contiene un `Text` y un `Button`. La UI se describe declarativamente: tú dices "así quiero que se vea" y Flutter renderiza.

Flutter **no usa los componentes nativos del sistema operativo** (a diferencia de React Native). Pinta su propia UI con un motor gráfico (Skia / Impeller) sobre un canvas, como un videojuego. Esto le da performance predecible y consistente entre plataformas.

### Por qué Dart
- Compila a código nativo ARM → rápido al ejecutar
- Compila también a JavaScript → la misma app puede correr en la web
- Hot reload: guardas código, la app se actualiza en ~1 segundo sin reiniciar → iteración rapidísima
- Null safety integrado al lenguaje → menos bugs

### Qué aprender
- Sintaxis básica de Dart: clases, constructores, `async/await`, streams
- Widget tree y diferencia entre `StatelessWidget` y `StatefulWidget`
- Hot reload vs hot restart
- Pub.dev (el repositorio de paquetes de Dart, equivalente a npm o PyPI)

**Recursos sugeridos:**
- [Dart tour en 30 minutos](https://dart.dev/language) — oficial
- [Flutter codelab de Google](https://docs.flutter.dev/get-started/codelab) — construye tu primera app
- [Flutter in Action (YouTube channel oficial)](https://www.youtube.com/@flutterdev)

---

## 3. Backend-as-a-Service (BaaS) y Supabase

### Qué es
Tradicionalmente, construir un backend requiere: elegir un lenguaje (Node.js, Python, Go...), escribir un servidor HTTP, conectarlo a una base de datos, implementar autenticación, manejar sesiones, desplegar en algún servidor, configurar TLS, monitoreo, etc. Semanas de trabajo antes de escribir una sola línea de lógica de negocio.

Un **BaaS (Backend-as-a-Service)** te da todo eso pre-armado como un servicio. Los más conocidos: Firebase (Google), Supabase, AWS Amplify, Appwrite.

**Supabase** es un BaaS open-source construido sobre Postgres (la base de datos más sólida del mundo open source). Te da:

- Base de datos **Postgres** con interfaz visual (Supabase Studio) para ver y editar datos
- **Autenticación** con email, phone OTP, OAuth providers, anonymous — sin escribir código
- **APIs auto-generadas** — cada tabla que creas tiene endpoints REST y GraphQL listos para usar
- **Realtime** — suscríbete a cambios en la DB desde el cliente vía WebSockets
- **Storage** para archivos (imágenes, PDFs, etc.)
- **Edge Functions** — equivalente a AWS Lambda, corre código server-side cuando lo necesitas
- **Row Level Security (RLS)** — reglas declarativas que controlan quién puede leer/escribir cada fila (ver sección 5)

### Por qué Supabase para este proyecto
Nuestro equipo es pequeño, el presupuesto es cero, y necesitamos: base de datos, auth, realtime, y reglas de seguridad. Hacerlo manualmente tomaría semanas; Supabase lo entrega en horas. Además el free tier aguanta el volumen esperado del MVP (decenas a cientos de usuarios).

### Qué aprender
- Qué es una API REST
- Diferencia entre frontend, backend y base de datos
- Qué es Postgres y por qué es relevante (sección siguiente)
- Cómo funciona la dashboard de Supabase (Studio)

**Recursos sugeridos:**
- [Supabase docs — Getting started](https://supabase.com/docs/guides/getting-started)
- [Supabase Crash Course (YouTube)](https://www.youtube.com/@Supabase)

---

## 4. Bases de datos relacionales: Postgres y SQL

### Qué es
Una **base de datos relacional** organiza la información en **tablas** con filas (registros) y columnas (atributos). Las tablas se relacionan entre sí por **llaves foráneas** (foreign keys).

Ejemplo en nuestro proyecto: la tabla `reports` tiene una columna `user_id` que referencia a `auth.users(id)`. Esto significa: "cada reporte pertenece a un usuario específico".

**SQL (Structured Query Language)** es el lenguaje para interactuar con la base de datos. Tiene unas pocas operaciones fundamentales:

```sql
SELECT * FROM reports WHERE route_id = '203' AND created_at > now() - interval '15 minutes';
INSERT INTO reports (route_id, type, value) VALUES ('203', 'occupancy', 'lleno');
UPDATE active_sessions SET ended_at = now() WHERE user_id = 'abc';
DELETE FROM reports WHERE id = 'xyz';
```

**Postgres** (también llamado PostgreSQL) es una implementación específica de base de datos relacional, open-source, extremadamente robusta, usada por Apple, Instagram, Reddit, Spotify. Supabase está construido sobre Postgres.

### Conceptos clave que vamos a usar
- **Tablas y columnas tipadas** (text, int, uuid, timestamptz, geography)
- **Primary keys** (identificador único de cada fila)
- **Foreign keys** (relaciones entre tablas)
- **Índices** (aceleran queries; los creamos en columnas que se consultan mucho)
- **Transacciones** (un conjunto de operaciones que todas se aplican o ninguna)
- **Funciones / RPC** (procedimientos almacenados escritos en PL/pgSQL — código que corre dentro de la DB)
- **Triggers** (funciones que se disparan automáticamente cuando pasa algo, ej: crear un perfil cuando se registra un usuario)
- **Extensiones** (módulos adicionales que amplían Postgres, ej: PostGIS para datos geoespaciales, pg_cron para tareas programadas)

### Qué aprender
- SQL básico: SELECT, INSERT, UPDATE, DELETE, JOIN, WHERE, GROUP BY
- Qué es un schema, una tabla, una columna, una row
- Qué es un índice y cuándo agregarlo

**Recursos sugeridos:**
- [SQLBolt (interactivo)](https://sqlbolt.com/) — aprende SQL en una tarde
- [Postgres in 100 seconds (Fireship, YouTube)](https://www.youtube.com/@Fireship)
- [Use The Index, Luke](https://use-the-index-luke.com/) — libro gratis sobre índices

---

## 5. PostGIS y datos geoespaciales

### Qué es
**PostGIS** es una extensión de Postgres que agrega soporte nativo para datos geoespaciales: puntos, líneas, polígonos, distancias reales en la superficie de la Tierra.

Sin PostGIS, para preguntar "¿qué rutas están a menos de 500 metros de mi ubicación?" tendrías que programar cálculos matemáticos complejos en tu aplicación. Con PostGIS:

```sql
SELECT * FROM routes
WHERE ST_DWithin(polyline, ST_MakePoint(-75.56, 6.24)::geography, 500);
```

Esa query devuelve rutas cuya polilínea está a menos de 500 metros del punto `(-75.56, 6.24)` — una ubicación en Medellín.

### Conceptos clave
- **Geometry vs Geography:** Geometry es 2D plano (rápido, pero impreciso para distancias grandes). Geography usa coordenadas sobre una esfera (más lento pero exacto). Para nuestro caso usamos Geography porque las distancias importan.
- **SRID 4326:** es el sistema de coordenadas estándar del mundo — el que usan Google Maps, GPS, todos. Latitud y longitud en grados decimales.
- **Tipos espaciales:** `Point` (un punto), `LineString` (una línea con varios puntos, útil para polilíneas de rutas), `Polygon` (un área cerrada).
- **Índices GiST:** índices especializados para queries espaciales. Sin ellos, preguntar "qué está cerca" sería lentísimo.

### Por qué lo usamos
- La tabla `routes` guarda la polilínea de cada ruta de bus como `geography(LineString, 4326)`
- La tabla `stops` guarda cada paradero como `geography(Point, 4326)`
- La tabla `active_sessions` guarda la última ubicación del bus reportada
- El picker de rutas usa `ST_DWithin` para mostrar las rutas más cercanas al usuario

### Qué aprender
- Qué es una coordenada lat/lng
- Diferencia entre Point, LineString, Polygon
- Funciones más usadas: `ST_DWithin`, `ST_Distance`, `ST_MakePoint`, `ST_AsGeoJSON`

**Recursos sugeridos:**
- [PostGIS tutorial (oficial)](https://postgis.net/workshops/postgis-intro/) — el workshop oficial, práctico
- [Supabase + PostGIS guide](https://supabase.com/docs/guides/database/extensions/postgis)

---

## 6. Row Level Security (RLS)

### Qué es
Tradicionalmente, en una app, la lógica de "quién puede ver qué" vive en el backend: el servidor pregunta a la base de datos por todos los registros, luego filtra en código antes de enviar al cliente. Esto funciona pero requiere escribir mucho backend.

**Row Level Security (RLS)** mueve esa lógica a la base de datos. Escribes reglas en SQL que dicen "esta fila solo es visible/editable por tal usuario bajo tales condiciones", y la DB las aplica automáticamente en cada query.

Ejemplo real de nuestro proyecto:

```sql
-- Un usuario solo puede actualizar su propia sesión activa
create policy "users can update own session"
  on active_sessions
  for update
  using (auth.uid() = user_id);

-- Cualquiera autenticado puede LEER sesiones no cerradas
create policy "anyone can read active sessions"
  on active_sessions
  for select
  using (ended_at is null);
```

Con estas políticas, **el cliente puede hablar directo con la DB** (vía la API de Supabase) sin miedo de que alguien haga trampa. Si un usuario malicioso intenta modificar la sesión de otro, la DB lo rechaza.

### Por qué lo usamos
RLS nos permite no escribir un backend custom. El cliente Flutter se conecta directo a Supabase, envía queries, y Supabase (gracias a RLS) garantiza que nadie vea o modifique datos que no le corresponden. Esto reduce drásticamente el código que tenemos que mantener.

### Qué aprender
- Cómo activar RLS en una tabla
- Cómo escribir una `policy` (select, insert, update, delete)
- Qué es `auth.uid()` y cómo la DB sabe qué usuario está haciendo la query
- Cuándo RLS no alcanza y necesitas una RPC con `security definer`

**Recursos sugeridos:**
- [Supabase RLS guide](https://supabase.com/docs/guides/database/postgres/row-level-security) — explicación clarísima con ejemplos
- [Postgres RLS (oficial)](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

---

## 7. Realtime: WebSockets, Broadcast, Postgres Changes, Presence

### Qué es
Una app tradicional hace **polling**: cada X segundos pregunta al servidor "¿hay algo nuevo?". Eso es ineficiente y no es verdaderamente en tiempo real.

**Realtime** usa **WebSockets** — una conexión permanente entre cliente y servidor donde cualquiera puede enviar mensajes en cualquier momento. El cliente no pregunta; el servidor le avisa cuando hay algo.

Supabase Realtime provee **tres primitivos** distintos:

#### Broadcast
Mensajes efímeros cliente-a-cliente. Se envían, se reciben, y desaparecen. No se guardan en DB. Ideales para alta frecuencia.

```
Cliente A → channel.sendBroadcastMessage({event: 'position', payload: {lat, lng}})
Cliente B (suscrito al mismo canal) → recibe el mensaje en ~50ms
```

En nuestro proyecto: **las posiciones en vivo de los buses viajan por Broadcast**. No persistimos cada tick de GPS; solo el último punto se guarda en `active_sessions.last_location` para que clientes nuevos tengan algo que mostrar al abrir el app.

#### Postgres Changes
Te suscribes a cambios en una tabla específica. Cuando alguien hace `INSERT`, `UPDATE` o `DELETE`, todos los clientes suscritos reciben el evento.

En nuestro proyecto: **los reportes de ocupación y seguridad usan Postgres Changes**. Son baja frecuencia (máximo 1 por minuto por usuario) y sí nos importa persistirlos para historial.

#### Presence
Saber quién está conectado al canal en tiempo real. Útil para "N usuarios activos ahora mismo en esta ruta".

No lo usamos en MVP, pero puede aparecer en v2.

### Por qué esta mezcla
Si usáramos Postgres Changes para todas las ubicaciones, sobrecargaríamos la DB con escrituras y a los clientes con mensajes filtrados. Broadcast es más barato y ligero para data efímera. La doc oficial de Supabase lo dice explícitamente: "si usas Postgres Changes a escala, considera Broadcast en su lugar".

### Qué aprender
- Qué es un WebSocket (vs HTTP)
- Diferencia entre pub/sub y request/response
- Cuándo usar Broadcast vs Postgres Changes

**Recursos sugeridos:**
- [Supabase Realtime docs](https://supabase.com/docs/guides/realtime)
- [WebSockets explained (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)

---

## 8. Autenticación: JWT y usuarios anónimos

### Qué es
**Autenticación** es el proceso de "¿quién eres?". Tradicionalmente: email + password.

**JWT (JSON Web Token)** es un formato estándar para representar la identidad de un usuario como un string firmado criptográficamente. Después del login, el servidor te devuelve un JWT; tú lo adjuntas a cada request siguiente, y el servidor confía en él sin tener que consultar la DB cada vez.

En Supabase, cuando un usuario se autentica, recibe un JWT que incluye su `user_id` (un UUID). Ese `user_id` es lo que RLS usa para decidir qué datos le muestra y deja modificar.

**Anonymous auth** (lo que usamos en MVP): Supabase crea un usuario con `user_id` aleatorio sin pedir email ni nada. El app lo guarda. Siguiente vez que se abre, usa el mismo `user_id`. Si desinstalas, perdiste tu identidad y recibes una nueva.

Esto es perfecto para adopción rápida: el usuario abre el app y ya puede reportar sin fricciones. Si después quiere "vincular" su cuenta anónima a un email o teléfono, Supabase soporta esa migración.

### Por qué anónimo para MVP
La fricción de pedir email/teléfono antes del primer uso mata la adopción. Queremos que el usuario entre y reporte. Si en v2 agregamos features que requieren identidad (leaderboards, trust score verificable, reputación), ahí introducimos phone OTP.

### Qué aprender
- Diferencia entre autenticación (quién eres) y autorización (qué puedes hacer)
- Qué es un JWT y cómo se verifica
- Qué son los claims (`sub`, `iat`, `exp`, etc.)
- Cómo Supabase integra JWT con Postgres (la función `auth.uid()`)

**Recursos sugeridos:**
- [JWT.io](https://jwt.io/) — página interactiva para explorar JWTs reales
- [Supabase Auth docs](https://supabase.com/docs/guides/auth)

---

## 9. Mapas: tiles vectoriales vs raster, GeoJSON, OpenStreetMap

### Qué es
Los mapas que ves en tu teléfono están hechos de miles de pequeñas imágenes cuadradas ("tiles") que el app descarga y compone. Hay dos tipos:

- **Raster tiles:** imágenes PNG/JPG pre-renderizadas. El servidor las genera una vez y el cliente las descarga. Gran tamaño, no se puede cambiar el estilo, no se rota bien.
- **Vector tiles:** datos vectoriales (geometrías + atributos). El cliente los renderiza en tiempo real usando WebGL. Mucho más pequeños, estilo customizable, rotación y zoom fluido.

**MapLibre GL** (lo que usamos) renderiza tiles vectoriales. Es el fork open-source de Mapbox GL.

**OpenStreetMap (OSM)** es la "Wikipedia de los mapas" — una base de datos global, gratuita, editable por voluntarios. La mayoría de tiles vectoriales gratuitos (OpenFreeMap, MapTiler, Protomaps) se generan a partir de OSM.

### GeoJSON
**GeoJSON** es un formato JSON para representar datos geoespaciales. Ejemplo:

```json
{
  "type": "LineString",
  "coordinates": [
    [-75.5681, 6.2476],
    [-75.5660, 6.2490],
    [-75.5612, 6.2521]
  ]
}
```

Eso es la polilínea de una ruta de bus. Lo que vamos a hacer en la semana 0 es dibujar las 5 rutas piloto en [geojson.io](https://geojson.io) y commitearlas al repo como archivos `.geojson`. Luego un script los carga a la tabla `routes` de Supabase.

### PMTiles
**PMTiles** es un formato relativamente nuevo (Protomaps) que mete todos los tiles de una región en **un solo archivo**. En vez de correr un servidor de tiles, subes el `.pmtiles` a un bucket S3 o Cloudflare R2, y MapLibre lo lee con byte-range requests. Cero infraestructura.

Para producción podemos generar un `.pmtiles` solo del Valle de Aburrá (~30-50 MB) y servirlo desde Cloudflare R2 gratis.

### Qué aprender
- Qué es un tile en un mapa, por qué se divide el mundo en tiles
- Sistema de coordenadas del mundo (latitud, longitud, SRID 4326)
- Cómo se estructura un GeoJSON (Point, LineString, Polygon, Feature, FeatureCollection)
- Qué es OpenStreetMap y por qué es importante

**Recursos sugeridos:**
- [OSM in 100 seconds (Fireship, YouTube)](https://www.youtube.com/@Fireship)
- [geojson.io](https://geojson.io) — dibuja y exporta GeoJSON visualmente
- [MapLibre GL docs](https://maplibre.org/maplibre-gl-js/docs/)
- [Protomaps blog](https://protomaps.com/blog/)

---

## 10. GPS y geolocalización móvil

### Qué es
El **GPS (Global Positioning System)** es una red de satélites que permite al teléfono calcular su ubicación triangulando señales. Adicionalmente, los teléfonos modernos combinan:

- Satélites GPS, GLONASS, Galileo
- Torres de celular
- Redes Wi-Fi conocidas (por MAC address)
- Sensores: acelerómetro, giroscopio, barómetro

Todo esto lo mezcla el sistema operativo en una sola API: "dame mi ubicación con tal precisión".

### Conceptos clave
- **Accuracy (precisión):** en metros. Afuera con cielo despejado: 3-5m. En un edificio: 50m+. En un bus con buses Wi-Fi cerca: a veces mejora.
- **Foreground vs Background:** en foreground (app abierta, pantalla encendida) el OS te da ubicación sin problema. En background (app minimizada o teléfono bloqueado) hay restricciones fuertes — especialmente en iOS.
- **Permisos:** el usuario debe autorizar cada tipo de acceso. En iOS los niveles son "When In Use" y "Always". En Android: Coarse, Fine, Background.
- **Battery drain:** pedir GPS cada segundo quema batería. Buenas prácticas: bajar frecuencia cuando está quieto, subir cuando está en movimiento. Tracelet y `react-native-background-geolocation` hacen esto con acelerómetro.

### Por qué foreground-only en MVP
- Evita la revisión de Google Play por `ACCESS_BACKGROUND_LOCATION`
- Evita pedirle al usuario permiso "Always" en iOS (que la gente rechaza)
- Simplifica el consent copy
- Real UX: el usuario abre el app al subir al bus, lo usa durante el viaje (~20 min), lo cierra al bajar

Post-MVP podemos agregar background con Tracelet cuando el producto demuestre tracción.

### Qué aprender
- Diferencia entre `getCurrentPosition` y `getPositionStream` (en Dart: `Geolocator.getPositionStream`)
- Cómo pedir permisos con `permission_handler`
- Qué es el estado "permiso denegado permanentemente" y cómo redirigir a Settings

**Recursos sugeridos:**
- [Flutter geolocator package](https://pub.dev/packages/geolocator)
- [Android location best practices](https://developer.android.com/training/location)

---

## 11. State management y Riverpod

### Qué es
En una app de Flutter, múltiples pantallas necesitan compartir información: "¿quién es el usuario actual?", "¿cuáles son los buses activos cerca?", "¿estoy en una sesión o no?".

Pasar esa información de widget en widget manualmente se vuelve caótico rápidamente. **State management** es el patrón para centralizar y distribuir esa información.

**Riverpod** es una librería de state management que define "providers" — pequeños bloques reactivos que otros widgets pueden escuchar. Ejemplo:

```dart
// Definimos un provider
final activeSessionProvider = StreamProvider<ActiveSession?>((ref) {
  final supabase = ref.watch(supabaseClientProvider);
  return supabase.from('active_sessions')
    .stream(primaryKey: ['id'])
    .eq('user_id', supabase.auth.currentUser!.id)
    .map((rows) => rows.isEmpty ? null : ActiveSession.fromJson(rows.first));
});

// En un widget, simplemente lo lees
class ActiveSessionBanner extends ConsumerWidget {
  Widget build(BuildContext context, WidgetRef ref) {
    final session = ref.watch(activeSessionProvider);
    return session.when(
      data: (s) => s == null ? SizedBox.shrink() : Text('Viajando en ruta ${s.routeId}'),
      loading: () => CircularProgressIndicator(),
      error: (e, _) => Text('Error: $e'),
    );
  }
}
```

Riverpod maneja automáticamente: recalcular cuando los datos cambian, limpiar recursos cuando nadie escucha, testing, dependencias entre providers.

### Por qué Riverpod (no Bloc, no Provider)
- **Provider** (el original) es más viejo, menos type-safe
- **Bloc** es potente pero verboso — mucho boilerplate para features simples
- **Riverpod** es compile-safe, compacto, y tiene `StreamProvider` que encaja perfecto con las subscripciones de Supabase Realtime

### Qué aprender
- Qué es un `Provider`, `FutureProvider`, `StreamProvider`, `NotifierProvider`
- `ref.watch` vs `ref.read` vs `ref.listen`
- Codegen con `riverpod_annotation`

**Recursos sugeridos:**
- [Riverpod docs oficiales](https://riverpod.dev/)
- [Code With Andrea — Riverpod series](https://codewithandrea.com/)

---

## 12. Distribución: APK, TestFlight, sideloading

### Qué es
Una app se distribuye a los usuarios de varias formas:

**Android:**
- **Google Play Store** — el canal oficial. Requiere cuenta de developer ($25 one-time), revisión de la app.
- **APK directo** — un archivo `.apk` que compartes por WhatsApp, Drive, email. El usuario lo instala manualmente. Requiere que active "Apps de fuentes desconocidas". Cero revisión, cero costo. **Lo que vamos a usar en MVP.**
- **Firebase App Distribution** — subes el APK, mandas link a testers, reciben notificaciones cuando hay nueva versión. Gratis.
- **Play Internal Testing** — un track especial del Play Store para testers, más rápido que producción.

**iOS:**
- **App Store** — canal oficial, $99/año Apple Developer, revisión estricta.
- **TestFlight** — canal beta oficial, viene con la cuenta de $99/año. Hasta 10,000 testers, builds válidos 90 días. **Lo que usaremos si tenemos >5 testers iOS.**
- **Ad-hoc builds** — instalables directamente a iPhones con su UDID registrado. Requiere cuenta de developer. Válidos ~1 año.
- **Free provisioning** — con Apple ID personal (gratis), puedes instalar builds firmados en **tu propio** iPhone. Pero expiran cada 7 días.

### Plan de distribución para MVP
1. **M0 a M6:** desarrollamos localmente con `flutter run` conectado por USB a nuestro Android/iOS
2. **M7:** generamos APK firmado + TestFlight build
3. Distribuimos APK al grupo Jugaad por Telegram/Drive
4. Distribuimos TestFlight a los miembros con iPhone

### Qué aprender
- Qué es un bundle ID / package name (ej: `co.jugaad.transpo`)
- Qué es "firmar" una app (signing) y por qué es necesario
- Qué es ADB (Android Debug Bridge) y cómo conectar tu phone por USB

**Recursos sugeridos:**
- [Flutter: Building and releasing Android](https://docs.flutter.dev/deployment/android)
- [Flutter: Building and releasing iOS](https://docs.flutter.dev/deployment/ios)

---

## 13. Git y flujo de trabajo en equipo

### Qué es
**Git** es un sistema de control de versiones. Permite múltiples personas editar el mismo código sin pisarse. Los conceptos mínimos:

- **Repositorio (repo):** la carpeta con el código + su historial
- **Commit:** una foto del estado del código en un momento
- **Branch (rama):** una línea paralela de desarrollo. `main` es la rama principal
- **Merge:** combinar cambios de una rama en otra
- **Pull Request (PR):** una propuesta de merge, con revisión de código antes de integrar

### Flujo propuesto para este proyecto
1. Cada milestone o feature → crear una rama (`feature/M1-map-and-gps`)
2. Commitear cambios pequeños y frecuentes
3. Abrir PR a `main` cuando el milestone esté completo
4. Al menos otra persona del equipo revisa antes de mergear
5. Mergear → borrar la rama

Evitar:
- Commits gigantes con 50 archivos de cambios
- Trabajar directamente en `main` sin PR
- Mensajes de commit inútiles como "fix" o "update"

### Qué aprender
- Comandos básicos: `clone`, `pull`, `add`, `commit`, `push`, `branch`, `checkout`, `merge`
- Resolución de conflictos
- Cómo funciona GitHub (issues, PRs, reviews)

**Recursos sugeridos:**
- [Pro Git (libro gratis)](https://git-scm.com/book/en/v2) — los primeros 3 capítulos bastan
- [Learn Git Branching (interactivo)](https://learngitbranching.js.org/)

---

## 14. Seguridad básica: rate limiting y validaciones

### Qué es
En cualquier app que reciba input de usuarios, hay que protegerse de abusos:

- **Rate limiting:** limitar cuántas veces un usuario puede hacer algo en un periodo. Ej: "máximo 1 reporte por minuto".
- **Validación:** chequear que los datos recibidos tengan forma correcta y valores razonables. Ej: lat entre -90 y 90, lng entre -180 y 180.
- **Anti-spoofing:** detectar datos físicamente imposibles. Ej: un bus no puede moverse a 300 km/h.

### En nuestro proyecto
- Rate limiting de reportes: RPC de Supabase rechaza si el usuario reportó hace <60s
- Validación de coordenadas: check constraints en la tabla
- Anti-spoofing de ubicación: RPC `update_session_location` calcula velocidad implícita y rechaza si > 120 km/h
- RLS: garantiza que un usuario no puede modificar sesiones/reportes de otros

### Qué aprender
- Por qué el cliente nunca es confiable
- Qué es una "race condition" y por qué importa en rate limiting
- Conceptos básicos de OWASP top 10

---

## Cuestionario de auto-evaluación

Antes de empezar M0, deberías poder responder estas preguntas sin buscar:

1. ¿Qué diferencia hay entre Flutter y una PWA?
2. ¿Por qué no vamos a persistir cada tick de GPS en la base de datos?
3. ¿Qué es RLS y por qué nos permite no escribir backend custom?
4. ¿Qué es un `FeatureProvider` en Riverpod y cómo lo usan los widgets?
5. ¿Qué es un tile vectorial y por qué OpenFreeMap es gratis?
6. ¿Cómo se distribuye un APK sin pasar por Play Store?
7. ¿Qué hace `ST_DWithin` y por qué lo necesitamos?
8. ¿Cuál es la diferencia entre Broadcast y Postgres Changes de Supabase Realtime?
9. ¿Qué es `auth.uid()` y cómo la DB sabe quién está haciendo una query?
10. ¿Por qué usamos auth anónimo en MVP y no email+password?

Si hay preguntas que no puedes responder, vuelve a la sección correspondiente y/o revisa los recursos sugeridos antes de unirte a la primera sesión de código.

---

## Formato sugerido de sesión de nivelación

Para acelerar la puesta en común, propongo:

- **Sesión 1 (1.5h):** secciones 1-4 — Qué es Flutter, qué es Supabase, qué es SQL básico
- **Sesión 2 (1.5h):** secciones 5-8 — PostGIS, RLS, Realtime, Auth
- **Sesión 3 (1h):** secciones 9-10 — Mapas y GPS en detalle
- **Sesión 4 (1h):** secciones 11-14 — Riverpod, distribución, git, seguridad
- **Hands-on (2h):** cada miembro instala Flutter, corre una app de ejemplo, commitea una mejora

Con 4 sesiones de ~1h y una de hands-on, el equipo debería estar listo para arrancar M0 con claridad compartida.
