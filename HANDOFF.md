Documento de entrega para el siguiente desarrollador. Cubre todo lo necesario para retomar el proyecto, probarlo, publicarlo en Play Store y mantenerlo. Lectura estimada: 20 min.

> **Fecha de este handoff:** 2026-06-04
> **Estado:** Listo para subir a Play Store. AAB de produccion generado, falta crear cuenta Play Console y publicar.
> **Cliente / Owner:** Technology Solutions and Services (TSS PTY) · techsspty.com · rmgd30@gmail.com

---

## TABLA DE CONTENIDO

1. [Que es AutoAlerta](#1-que-es-autoalerta)
2. [Stack tecnico](#2-stack-tecnico)
3. [Arquitectura del codigo](#3-arquitectura-del-codigo)
4. [Como configurar el entorno local](#4-como-configurar-el-entorno-local)
5. [Como correr la app en desarrollo](#5-como-correr-la-app-en-desarrollo)
6. [Como buildear APK / AAB](#6-como-buildear-apk--aab)
7. [Como probar el APK](#7-como-probar-el-apk)
8. [Como subir a Play Store](#8-como-subir-a-play-store)
9. [Cambios recientes (sesion 2026-06-04)](#9-cambios-recientes-sesion-2026-06-04)
10. [Pendientes y riesgos](#10-pendientes-y-riesgos)
11. [Recursos externos y accesos](#11-recursos-externos-y-accesos)
12. [Contactos](#12-contactos)

---

## 1. Que es AutoAlerta

App movil Android (no iOS por ahora) para que dueños de autos en **Panama** registren sus vehiculos y choferes, y reciban **alertas locales** antes de que venzan los documentos legales: licencia, revisado vehicular, calcomania municipal, placa metalica, poliza de seguro, cedula.

### Diferenciadores

- **Sin backend propio.** Los datos del usuario se guardan en SU Google Drive (scope `drive.file`, solo accedemos a la carpeta que creamos). Cero costo operativo, cero responsabilidad de almacenar datos personales.
- **Alertas locales** (no requieren push server) usando `expo-notifications`.
- **100% espanol**, leyes y plazos especificos de Panama (ATTT, Sertracen, Tribunal Electoral).
- **Tema oscuro nativo** alineado al design system de TSS PTY (navy + naranja).

### Funcionalidad principal

- Login con Google (OAuth implicito, scope `drive.file` + `userinfo`).
- Registrar hasta 10 autos con foto y datos basicos.
- Registrar choferes (licencia + cedula).
- Para cada documento: fecha de vencimiento + dias de aviso configurables.
- Motor de alertas calcula `vencido / urgente / proximo / al dia` (semaforo).
- Notificaciones locales programadas (180, 90, 60, 30, 15 dias antes — configurable).
- Sincronizacion con Google Drive (lectura/escritura del JSON de datos en la carpeta de la app).
- Modo offline-first: app funciona sin Internet, sincroniza cuando hay red.

---

## 2. Stack tecnico

| Capa | Tecnologia | Notas criticas |
|---|---|---|
| Runtime | **Expo SDK 54** (React Native 0.81 + React 19) | NewArchitecture habilitada (`newArchEnabled: true` en app.json) |
| Lenguaje | TypeScript estricto | `npx tsc --noEmit` debe pasar en 0 errores antes de build |
| Routing | `expo-router` v6 file-based | Carpetas `app/(auth)`, `app/(tabs)`, `app/auto/[id]`, `app/chofer/[id]` |
| UI | `react-native-paper` (Material Design 3) | Tema dark forzado en `app/_layout.tsx` |
| Auth | `@react-native-google-signin/google-signin` + `expo-auth-session` | OAuth implicit flow, scopes `drive.file` + `userinfo.email/profile` |
| Storage backend | **Google Drive API v3** | Scope `drive.file` — la app solo accede a archivos que ella misma cree |
| Storage local | `expo-secure-store` (tokens) + `@react-native-async-storage/async-storage` (cache) | |
| Notifications | `expo-notifications` (locales) | NO usa push server, no necesita FCM credentials para alertas |
| Fonts | `@expo-google-fonts/{bebas-neue,rajdhani,dm-sans,jetbrains-mono}` | Cargadas en `app/_layout.tsx` antes del primer render |
| Build | **EAS Build** (cloud) | Perfiles `development`, `preview` (APK), `production` (AAB) |
| Versionado | Git (master branch) + EAS branches `preview` / `production` | OTA updates via `expo-updates` con runtime version `appVersion` |

### Dependencias bloqueadas/sensibles

- `babel-preset-expo@~54.0.10` — DEBE coincidir con SDK 54. Si se actualiza Expo, actualizar tambien esto en sintonia. Incidente documentado en [MEMORIA-CAMBIOS.md](MEMORIA-CAMBIOS.md) seccion "2026-05-17 (parte 2)".
- `expo-font` — Peer dep de `@expo/vector-icons`. Si se quita, el APK crashea en el primer render (no en Expo Go).
- `react-native-chart-kit` ya NO se usa — fue reemplazado por SVG custom (commit `8d1dbbf`) por incompatibilidad con New Architecture.

---

## 3. Arquitectura del codigo

```
AutoAlerta/
├── app/                          # Rutas (expo-router)
│   ├── _layout.tsx              # Root: SafeAreaProvider, PaperProvider, fonts, dark theme
│   ├── index.tsx                # Splash + redirect a auth o tabs
│   ├── (auth)/
│   │   ├── _layout.tsx          # Layout flujo no autenticado
│   │   └── welcome.tsx          # Login con Google
│   ├── (tabs)/
│   │   ├── _layout.tsx          # Tab bar (con safe-area inset para Android gestures)
│   │   ├── index.tsx            # Inicio (KPIs + alertas por severidad)
│   │   ├── autos.tsx            # Lista de autos
│   │   ├── choferes.tsx         # Lista de choferes
│   │   ├── reportes.tsx         # Reportes y graficos
│   │   └── config.tsx           # Ajustes (perfil, sync, notificaciones, cerrar sesion)
│   ├── auto/
│   │   ├── [id].tsx             # Detalle de auto + documentos
│   │   └── nuevo.tsx            # Modal nuevo auto
│   └── chofer/
│       ├── [id].tsx             # Detalle de chofer
│       └── nuevo.tsx            # Modal nuevo chofer
│
├── components/                   # Componentes reutilizables
│   ├── AlertBanner.tsx          # Banner agrupador de alertas (semaforo)
│   ├── DocumentCard.tsx         # Tarjeta de documento con badge
│   ├── EmptyState.tsx           # Estado vacio
│   ├── Form.tsx                 # TextField, etc.
│   ├── FotoCapture.tsx          # Capturar foto con expo-camera
│   └── SyncIndicator.tsx        # Indicador de estado de sync
│
├── services/                     # Logica de negocio + integraciones externas
│   ├── alertEngine.ts           # Calcula severidad y dias restantes
│   ├── config.ts                # Configuracion (env vars, OAuth client IDs)
│   ├── googleAuth.ts            # Login/logout Google + token management
│   ├── googleDrive.ts           # CRUD del JSON en Drive
│   ├── imagenes.ts              # Compresion <800KB
│   ├── mutations.ts             # Operaciones de actualizacion sobre el datos store
│   ├── notifications.ts         # Schedule + cancel de notifications locales
│   ├── reportes.ts              # Generacion de reportes PDF/CSV
│   └── storage.ts               # AsyncStorage + SecureStore wrappers
│
├── hooks/
│   └── AppContext.tsx           # Context provider: usuario, datos, autenticado, sincronizar, cerrarSesion
│
├── constants/
│   └── tema.ts                  # PALETA, DS, ESPACIADO, RADIO, TIPO, FUENTES, TEMA_OSCURO
│
├── types/                        # Tipos TypeScript del dominio (Auto, Chofer, Alerta, etc.)
│
├── assets/                       # Iconos, splash, adaptive-icon
│
├── play-store/                   # Metadata para publicacion (creada en sesion 2026-06-04)
│   ├── README.md                # Indice del proceso de publicacion
│   ├── 01-duns-number.md        # Tramite D-U-N-S
│   ├── 02-play-console-signup.md
│   ├── 03-app-listing.md        # Textos de ficha listos para copy/paste
│   ├── 04-content-rating.md     # Respuestas IARC
│   ├── 05-data-safety.md        # Declaracion privacidad
│   ├── 06-screenshots-checklist.md
│   ├── 07-store-graphics.md     # Specs icon y feature graphic
│   ├── 08-eas-submit-setup.md   # Service Account + eas submit
│   └── 09-release-checklist.md
│
├── releases/                     # APKs y AAB generados (no commiteados, > 100 MB)
├── web/                          # privacidad.html y terminos.html para hostear
├── design/                       # Mockups y design system de referencia
├── MEMORIA-CAMBIOS.md           # **CRITICO** — bitacora de incidentes y root causes
├── HANDOFF.md                   # Este archivo
└── README.md                    # Vision general
```

### Modelo de datos en Drive

Cuando el usuario inicia sesion por primera vez, la app crea en su Drive:

```
Mi unidad/AutoAlerta/
├── datos.json                   # Estado completo serializado (usuario, autos, choferes, preferencias)
└── fotos/
    ├── auto-{id}-{nombre}.jpg
    └── doc-{id}-{tipo}.jpg
```

Solo accedemos a archivos creados por nosotros (scope `drive.file`). El usuario puede borrar la carpeta y la app se reinicia limpia.

---

## 4. Como configurar el entorno local

### Requisitos

- **Node.js 20+** (recomendado 22 LTS)
- **npm** (viene con Node) o pnpm
- **Git**
- **Cuenta Expo** (gratis en https://expo.dev) — para builds EAS
- **Android Studio** (opcional, solo si quieres emulador local). Para builds y APKs no es necesario porque EAS Build es cloud.
- **VSCode** con extensiones: `Expo Tools`, `ESLint`, `Prettier`.

### Clonar e instalar

```powershell
git clone <repo-url>
cd AutoAlerta
npm install
```

### Variables de entorno

Crear `.env` en la raiz (NO commiteado, ver `.gitignore`):

```env
EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID=856777328728-qb8lhpoe1lvm7mo750h66bm5v05gj7uj.apps.googleusercontent.com
EXPO_PUBLIC_GOOGLE_IOS_CLIENT_ID=<no-aplica-por-ahora>
EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID=856777328728-qnbm8rd92jtm2hqsa5f15226r60nrnf7.apps.googleusercontent.com
```

> **Si no tienes acceso al Google Cloud Console actual:** habria que crear OAuth client IDs nuevos. Ver `services/googleAuth.ts` y `app.json:extra.googleAndroidClientId`. Pero los actuales ya estan en `app.json` para los APKs firmados con el keystore EAS — si cambias los OAuth IDs, hay que rebuildear y resubir.

### Login a EAS

```powershell
npx eas login
# Usuario: ronalds30
# (pedir password al owner)
```

### Validar el entorno

```powershell
npx expo-doctor     # Debe pasar 17/17
npx tsc --noEmit    # Debe pasar con 0 errores
```

---

## 5. Como correr la app en desarrollo

### Opcion 1 — Expo Go (mas rapido, NO sirve para probar login Google ni notificaciones reales)

```powershell
npm start
```

Escanear QR con la app Expo Go en un Android real. Limitaciones: el login Google falla porque Expo Go no tiene el `@react-native-google-signin/google-signin` nativo. Solo sirve para iterar UI.

### Opcion 2 — Development build (preferido)

```powershell
npx eas build --profile development --platform android
```

Una vez que termine (~15 min), descargar el APK desde el link de EAS, instalarlo en el telefono, y correr:

```powershell
npx expo start --dev-client
```

El development build SI tiene Google Sign-In nativo, asi que el login funciona como en produccion. Es la mejor forma de desarrollar.

### Opcion 3 — Emulador Android

Requiere Android Studio + emulador con Google Play Services. El login Google requiere que el emulador tenga "Google Play" en el AVD (no "Android Open Source"). Recomendable solo si vas a hacer mucho debug nativo.

### Hot reload y debugging

- Cmd+M (o sacudir el telefono) abre el dev menu.
- Logs en la terminal donde corre `expo start`.
- React DevTools: `npx react-devtools` en otra terminal.

---

## 6. Como buildear APK / AAB

### Perfiles EAS (configurados en `eas.json`)

| Perfil | Output | Para que |
|---|---|---|
| `development` | APK con dev client | Desarrollo con hot reload |
| `preview` | APK firmado de release | Internal testing, sideload, distribucion fuera de Play Store |
| `production` | AAB firmado | Subir a Play Store (Play Store NO acepta APK) |

### Build preview (APK para probar en tu telefono)

```powershell
npx eas build --platform android --profile preview --non-interactive
```

Sale un APK descargable desde https://expo.dev/accounts/ronalds30/projects/AutoAlerta/builds

### Build production (AAB para Play Store)

```powershell
npx eas build --platform android --profile production --non-interactive
```

Sale un AAB descargable. Esto es lo que se sube a Play Store.

### Que ya tenemos generado al 2026-06-04

| Tipo | Build ID | Archivo local | Tamano |
|---|---|---|---|
| APK preview | `44570170-2e2c-420a-aa16-e55cb013158d` | [releases/AutoAlerta-v1.0-dark-redesign.apk](releases/AutoAlerta-v1.0-dark-redesign.apk) | 110 MB |
| AAB production | `0d04ddde-9d59-478e-aac8-b8b32a0a5f7a` | [releases/AutoAlerta-v1.0-production.aab](releases/AutoAlerta-v1.0-production.aab) | 72 MB |

Ambos firmados con el keystore EAS `Build Credentials g84zrUxkr3`. **No cambiar el keystore entre releases** o Play Store rechaza el upload.

### Cuanto demora un build

10-20 min en cola + ejecucion en EAS Build cloud. Si la cola esta vacia, ~8 min. La hora puede variar segun la carga del servicio.

### Bumpear version para siguiente release

Antes de cada nuevo build production:

1. Editar `app.json`:
   ```json
   "version": "1.0.1",          // bump aqui
   "android": { ... }
   ```
2. Cada release a Play Store DEBE tener un `versionCode` ascendente. EAS Build lo maneja automaticamente si tienes `appVersionSource: "local"` (que es lo configurado). Si lo quieres explicito, pon `"versionCode": 2` en `android` de app.json.

---

## 7. Como probar el APK

### Instalar el APK en un Android real

**Opcion A — Cable USB:**
1. Conectar telefono al PC.
2. En el telefono: notificacion USB → "Transferencia de archivos".
3. Copiar el `.apk` a la carpeta `Download`.
4. En el telefono: explorador → tap en el APK → "Instalar" (puede pedir permitir fuentes desconocidas).

**Opcion B — ADB:**
```powershell
.\platform-tools-extracted\platform-tools\adb.exe install -r releases\AutoAlerta-v1.0-dark-redesign.apk
```
(`-r` reemplaza la version anterior si ya estaba instalada)

**Opcion C — Drive o WhatsApp:**
1. Subir el APK a Drive (drag & drop).
2. En el telefono abrir Drive → descargar → tap → instalar.

### Plan de pruebas manual (golden path)

| # | Caso | Como probar | Resultado esperado |
|---|---|---|---|
| 1 | Splash | Abrir la app desde cero | Logo navy + AUTOALERTA en Bebas Neue, transicion suave a Welcome o Tabs |
| 2 | Login Google | Welcome → "Continuar con Google" | Picker de cuentas Google, autoriza scope drive, redirect a Inicio |
| 3 | Sin Internet | Activar modo avion → abrir app ya logueada | App abre, muestra datos locales |
| 4 | Crear auto | Mis autos → + → llenar campos + foto → Guardar | Auto aparece en lista, persiste tras reinicio |
| 5 | Crear chofer | Choferes → + → llenar + licencia | Aparece en lista |
| 6 | Documento vencido | Editar fecha de calcomania a hace 5 dias | Badge rojo "VENCIDO" en card |
| 7 | Documento proximo | Fecha a +25 dias | Badge naranja "URGENTE" |
| 8 | Alertas en Inicio | Volver a Inicio | Aparecen las alertas agrupadas por severidad |
| 9 | Notificacion local | Settings → activar notifs → esperar el horario disparador | Notificacion aparece en barra de estado |
| 10 | Sincronizar Drive | Ajustes → "Sincronizar con Drive" | Spinner → "Sincronizado", carpeta `AutoAlerta/` existe en Drive |
| 11 | Cerrar sesion | Ajustes → "Cerrar sesion" | Vuelve a Welcome, sesion limpia |
| 12 | Tab bar Android | Probar en Samsung con gestos | Los 5 tabs se pueden tocar sin que los tape la barra de gestos |

### Cosas a NO romper accidentalmente

- Compatibilidad con datos viejos: si cambias el formato de `datos.json` en Drive, agrega migracion en `services/storage.ts` para no romper usuarios actuales.
- OAuth client IDs: si cambias los IDs en `app.json:extra`, los APKs viejos dejan de poder loguear.
- Keystore EAS: si pierdes el keystore o lo cambias, no puedes actualizar la app en Play Store y los usuarios no reciben updates.

---

## 8. Como subir a Play Store

**Resumen breve:** Carpeta [play-store/](play-store/) tiene todos los docs paso a paso. Leerlos en orden 01 → 09.

### Pre-requisitos personales (NO se pueden automatizar)

1. **D-U-N-S Number para TSS PTY** (gratis, 1-2 dias). Ver [play-store/01-duns-number.md](play-store/01-duns-number.md).
2. **Cuenta Play Console** como Organizacion (USD 25 unico + verificacion 1-7 dias). Ver [play-store/02-play-console-signup.md](play-store/02-play-console-signup.md).
3. **Politica de privacidad en URL HTTPS publica**. Tenemos el HTML en [web/privacidad.html](web/privacidad.html), falta hostearlo. Recomendado: subir a techsspty.com.

### Una vez aprobada la cuenta Play Console

1. **Crear app en Play Console** (10 min):
   - Nombre: AutoAlerta
   - Espanol (Panama)
   - Free
   - App

2. **Subir el AAB manualmente la primera vez**:
   - Descargar de https://expo.dev/artifacts/eas/2Bf7qBD9z8zLsCkAUTVw2h.aab (o usar el local en `releases/AutoAlerta-v1.0-production.aab`).
   - Play Console → Testing → Internal testing → Create new release → Upload.

3. **Llenar ficha completa**:
   - Copy/paste textos de [play-store/03-app-listing.md](play-store/03-app-listing.md).
   - Subir screenshots (tomar segun [play-store/06-screenshots-checklist.md](play-store/06-screenshots-checklist.md)).
   - Subir Feature Graphic 1024x500 (generar segun [play-store/07-store-graphics.md](play-store/07-store-graphics.md)).
   - Responder Content Rating con respuestas de [play-store/04-content-rating.md](play-store/04-content-rating.md).
   - Llenar Data Safety form con [play-store/05-data-safety.md](play-store/05-data-safety.md).

4. **Configurar `eas submit`** para futuros releases automaticos:
   - Seguir [play-store/08-eas-submit-setup.md](play-store/08-eas-submit-setup.md).
   - Crear Service Account en Google Cloud, dar permisos en Play Console.
   - Una vez configurado: `npx eas submit --platform android --profile production --latest` sube el ultimo build a Play Store.

5. **Promover a Production**:
   - Probar en Internal Testing primero (puedes invitar emails de testers).
   - Cuando este OK: Play Console → Production → Promote release.
   - Google revisa en 1-3 dias.

6. **Final**: app publicada en https://play.google.com/store/apps/details?id=com.tsspty.autoalerta

Ver checklist completo en [play-store/09-release-checklist.md](play-store/09-release-checklist.md).

---

## 9. Cambios recientes (sesion 2026-06-04)

Cambios aplicados en esta sesion (no documentados aun en MEMORIA-CAMBIOS.md, ver commits para detalle). En orden cronologico:

### 9.1 Fix tab bar tapado por barra de gestos Android

**Sintoma:** En Samsung Galaxy con navegacion por gestos (la barra `||| O <` abajo), el tab bar de la app quedaba debajo de la barra de gestos y los 5 botones no se podian tocar.

**Causa raiz:** `app.json:android.edgeToEdgeEnabled: true` pone el contenido bajo las system bars. El tab bar tenia `height: 64` fijo sin considerar el `safeAreaInsets.bottom`.

**Solucion:** En [app/(tabs)/_layout.tsx](app/(tabs)/_layout.tsx):
- Importar `useSafeAreaInsets` de `react-native-safe-area-context`.
- Calcular `bottomInset = Math.max(insets.bottom, 8)`.
- `tabBarStyle.height = 56 + bottomInset` y `paddingBottom: bottomInset`.

### 9.2 Boton "Cerrar sesion" demasiado grande

**Sintoma:** El boton rojo `mode="contained"` con `height: 52` ocupaba el ancho completo y dominaba visualmente Ajustes.

**Solucion:** En [app/(tabs)/config.tsx](app/(tabs)/config.tsx) cambiado a `mode="outlined"` con `compact`, height 40, `minWidth: 200`, `alignSelf: 'center'`.

### 9.3 Migracion a tema oscuro (Claude Design System)

**Motivacion:** El owner solicito aplicar el design system entregado por Claude Design (carpeta `Downloads/AutoAlerta Design System`). Es un tema oscuro navy + naranja con tipografias Bebas Neue / Rajdhani / DM Sans / JetBrains Mono.

**Cambios:**
- [constants/tema.ts](constants/tema.ts) reescrito:
  - `DS` (design system tokens): navy900-600, blue700-300, accent (FF6F00), red, amber, green, surface1-3, border.
  - `PALETA` (Material 3 mapping): primary = naranja, background = navy800, onSurface = white, etc.
  - `RADIO`: md=8, lg=12, xl=16, full=9999.
  - `FUENTES`: nombres de las fuentes cargadas en `_layout.tsx`.
  - `TIPO`: agregado `fontFamily` y nuevos sizes (displayLg=40 con Bebas Neue).
  - `TEMA_OSCURO`: tema unico (no hay claro). `TEMA_CLARO` queda como alias para retrocompatibilidad.
- 4 paquetes instalados: `@expo-google-fonts/bebas-neue`, `rajdhani`, `dm-sans`, `jetbrains-mono`.
- [app/_layout.tsx](app/_layout.tsx) reescrito:
  - `useFonts` carga las 4 familias.
  - Renderiza placeholder navy mientras cargan (evita flash blanco).
  - StatusBar `style="light"` + backgroundColor navy.
  - Stack con `contentStyle.backgroundColor = navy`.
- [app.json](app.json):
  - `userInterfaceStyle`: `"automatic"` → `"dark"`.
  - `splash.backgroundColor`: `"#1a237e"` → `"#0A1628"` (navy del DS).
  - `android.adaptiveIcon.backgroundColor`: idem.
  - `expo-notifications.color`: `"#1a237e"` → `"#FF6F00"` (naranja accent).
- Pantallas actualizadas para dark:
  - [app/index.tsx](app/index.tsx) (splash)
  - [app/(auth)/welcome.tsx](app/(auth)/welcome.tsx) (login + boton Google blanco con texto navy)
  - [app/(tabs)/_layout.tsx](app/(tabs)/_layout.tsx) (tab bar navy900)
  - [app/(tabs)/config.tsx](app/(tabs)/config.tsx) (boton cerrar sesion + import limpio)
  - [app/(tabs)/index.tsx](app/(tabs)/index.tsx) (kpiIcon con bg orange-tinted)
  - [components/DocumentCard.tsx](components/DocumentCard.tsx) (bordes dark + iconWrap orange-tinted)
  - [components/AlertBanner.tsx](components/AlertBanner.tsx) (bordes y radios actualizados)

### 9.4 Instalacion de Claude Code skills

Instalados 17 skills del repo `anthropics/skills` en `C:\Users\Admin\.claude\skills\`:
algorithmic-art, brand-guidelines, canvas-design, claude-api, doc-coauthoring, docx, frontend-design, internal-comms, mcp-builder, pdf, pptx, skill-creator, slack-gif-creator, theme-factory, web-artifacts-builder, webapp-testing, xlsx.

### 9.5 Builds nuevos

- APK preview (para sideload): `44570170-2e2c-420a-aa16-e55cb013158d` → `releases/AutoAlerta-v1.0-dark-redesign.apk` (110 MB).
- AAB production (para Play Store): `0d04ddde-9d59-478e-aac8-b8b32a0a5f7a` → `releases/AutoAlerta-v1.0-production.aab` (72 MB).

### 9.6 Metadata Play Store

Creada la carpeta [play-store/](play-store/) con 9 documentos guia para publicacion. Ver seccion 8 de este HANDOFF.

---

## 10. Pendientes y riesgos

### Bloqueante para Play Store

| # | Pendiente | Quien lo hace | ETA |
|---|---|---|---|
| 1 | **D-U-N-S Number para TSS PTY** | Owner (Ronald) | 1-2 dias gratis |
| 2 | **Cuenta Play Console aprobada** (con D-U-N-S) | Owner | 1-7 dias post pago USD 25 |
| 3 | **Hostear politica de privacidad** en HTTPS publica (`techsspty.com/autoalerta/privacidad`) | Programador con acceso al hosting | 30 min |
| 4 | **Generar Feature Graphic** 1024x500 PNG | Programador o disenador (con design system) | 1-2 horas |
| 5 | **Tomar 8 screenshots** desde el APK preview en Samsung | Owner | 30 min |
| 6 | **Crear Service Account** Google Cloud para `eas submit` | Programador (post cuenta aprobada) | 15 min |

### Riesgos tecnicos conocidos

| # | Riesgo | Mitigacion |
|---|---|---|
| 1 | Si el OAuth client ID se cambia, los APKs viejos dejan de loguear | NO tocar los IDs en `app.json:extra.google*ClientId`. Si hay que rotar, comunicar a usuarios. |
| 2 | Si se pierde el keystore EAS `g84zrUxkr3`, no podemos actualizar la app en Play Store | Backup del keystore: `npx eas credentials` → Android → Keystore → Download. Guardar en lugar seguro. |
| 3 | Tamano del APK (110 MB) es alto | Migrar a App Bundle (.aab) reduce a ~30-40 MB descargado por usuario gracias a Dynamic Delivery. Ya estamos generando AAB para Play Store. |
| 4 | New Architecture habilitada | Cualquier libreria que no soporte New Arch puede crashear. Caso ya ocurrido con `react-native-chart-kit`, resuelto reemplazando por SVG custom. Si agregas una nueva libreria, verificar New Arch compatibility primero. |
| 5 | Notificaciones locales solo Android | Para iOS habria que configurar push remoto o adaptar el scheduling. Por ahora la app es Android-only. |
| 6 | Drive scope `drive.file` no permite acceder a archivos creados por el usuario fuera de la app | Es el comportamiento deseado por privacidad. Si el usuario borra la carpeta en Drive, la app la recrea vacia. |

### Mejoras sugeridas (no urgentes)

- **Sentry / Crashlytics**: actualmente no hay telemetria de crashes. Considerar agregar para monitorear errores en produccion. Si se agrega, actualizar Data Safety form en Play Console.
- **iOS**: el codigo es 90% portable. Faltarian solo el OAuth client iOS, certificados Apple Developer (USD 99/ano), y testear.
- **Web**: ya hay configuracion para web (`expo start --web`) y carpeta `web/` con paginas legales. Podria evolucionar a un portal web companion.
- **i18n**: actualmente todo hardcoded en espanol. Si se expande fuera de Panama, usar `i18n-js` o `react-intl`.
- **Tests automatizados**: NO hay tests. Agregar `jest` + `react-native-testing-library` cuando el equipo crezca.

---

## 11. Recursos externos y accesos

### Cuentas y dashboards

| Servicio | URL | Usuario / Owner | Para que |
|---|---|---|---|
| Expo EAS | https://expo.dev/accounts/ronalds30/projects/AutoAlerta | `ronalds30` | Builds, OTA updates, credentials |
| Google Cloud Console | https://console.cloud.google.com | (cuenta del owner) | OAuth clients, futuras APIs |
| Play Console | https://play.google.com/console | **pendiente crear** | Publicacion app |
| Dun & Bradstreet | https://www.dnb.com | **pendiente** | D-U-N-S Number |
| Drive de testing | (cuenta del owner) | rmgd30@gmail.com | Ver carpeta `AutoAlerta/` que crea la app |

### Datos del proyecto en EAS

- Project ID: `2e2bb015-9a08-4a11-86c2-a3748613340a`
- Owner: `ronalds30`
- Update URL: `https://u.expo.dev/2e2bb015-9a08-4a11-86c2-a3748613340a`
- Keystore EAS: `Build Credentials g84zrUxkr3` (default Android, ya generado por EAS)

### OAuth client IDs (de app.json)

- Android: `856777328728-qb8lhpoe1lvm7mo750h66bm5v05gj7uj.apps.googleusercontent.com`
- Web: `856777328728-qnbm8rd92jtm2hqsa5f15226r60nrnf7.apps.googleusercontent.com`

Ambos del mismo proyecto Google Cloud `856777328728`. Pedir al owner acceso a ese proyecto si se necesita rotar credenciales.

### Branding

- Color primario (accent): `#FF6F00` (naranja)
- Fondo principal: `#0A1628` (navy 800)
- Logo SVG: embebido en `web/dashboard.html` y `design/` (ver tambien diseno entregado por Claude Design en `Downloads/AutoAlerta Design System/`).
- Fuentes: Bebas Neue (display), Rajdhani (heading), DM Sans (body), JetBrains Mono (mono).
- Aplicacion ID: `com.tsspty.autoalerta` (NO cambiar despues de subir a Play Store).

### Leyes Panama (reglas del motor de alertas)

Ver memoria documentada en `C:\Users\Admin\.claude\projects\c--Aplicaciones-AutoAlerta\memory\autoalerta-leyes-panama.md` o consultar:

- ATTT: https://attt.gob.pa — revisado vehicular anual
- Sertracen: https://sertracen.com.pa — licencia 4 anos
- Tribunal Electoral: https://tribunalcontigo.com — cedula 10 anos

---

## 12. Contactos

| Rol | Nombre | Email | Telefono |
|---|---|---|---|
| Owner / Founder | Ronald Gonzalez | rmgd30@gmail.com | (pedir al owner) |
| Empresa | Technology Solutions and Services (TSS PTY) | soporte@techsspty.com | (pedir al owner) |
| Sitio corporativo | https://techsspty.com | | |

### Para soporte tecnico

- Bugs / issues nuevos: revisar `MEMORIA-CAMBIOS.md` primero (incidentes resueltos historicos).
- Si la app crashea al abrir un APK: revisar [MEMORIA-CAMBIOS.md](MEMORIA-CAMBIOS.md) → "Lecciones para el proximo cambio". Casi siempre es peer dependency nativa faltante o babel version mismatch.
- Si OAuth falla: revisar `services/googleAuth.ts` + variables de entorno + OAuth client IDs en Google Cloud Console.
- Si EAS Build falla: revisar logs en https://expo.dev/accounts/ronalds30/projects/AutoAlerta/builds. Errores tipicos: dependencias mal versionadas, plugins de expo mal configurados.

---

## CHECKLIST DE ENTREGA

Marca esto cuando recibas el proyecto:

- [ ] Acceso a la cuenta EAS `ronalds30` (con password compartido por el owner)
- [ ] Acceso al Google Cloud Console del proyecto `856777328728` (owner debe agregarte)
- [ ] Acceso al hosting de `techsspty.com` (si vas a publicar la politica de privacidad)
- [ ] Clonaste el repo y `npm install` corre sin errores
- [ ] `npx tsc --noEmit` pasa con 0 errores
- [ ] `npx expo-doctor` pasa 17/17
- [ ] Puedes loguear con EAS (`npx eas whoami` muestra `ronalds30`)
- [ ] Tienes el APK actual instalado en un Android real y puedes loguear con Google
- [ ] Leiste este HANDOFF.md y MEMORIA-CAMBIOS.md completos
- [ ] Leiste los 9 archivos de `play-store/` para entender el proceso de publicacion
- [ ] Tienes contacto directo con el owner por WhatsApp o email para preguntas

Cuando todo lo de arriba este marcado, estas listo para continuar el proyecto.

---

**Fin del documento.** Cualquier pregunta, escribir a Ronald Gonzalez (rmgd30@gmail.com).
