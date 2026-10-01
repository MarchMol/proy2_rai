# Stack y despliegue — AI Assisted News App


## 1. Resumen de decisiones

| Pieza | Tecnología | Dónde vive | Costo |
|---|---|---|---|
| App móvil | Flutter (Dart) | APK Android + instalación nativa iOS desde Mac | $0 |
| Portal admin | Next.js (TypeScript) | Vercel (Hobby) | $0 |
| API REST + servidor MCP | Python (FastAPI + SDK oficial `mcp`) | Google Cloud Run (un solo servicio) | $0 dentro del free tier |
| Base de datos | Postgres | Supabase (Free) | $0 |
| Imágenes | Object storage | Supabase Storage (Free) | $0 |
| Autenticación | Google Sign-In | Firebase Authentication | $0 |
| IA (LLM / imágenes) | Por definir | API del proveedor | **≤ USD 20 en total** |

### Por qué estas decisiones

- **Cloud Run en lugar de Railway:** Railway ya no tiene un plan gratuito real para un servicio siempre encendido. Cloud Run escala a cero, su free tier (~2M requests/mes, 180k vCPU‑s) sobra para el curso y vive en el mismo proyecto de Google Cloud que Firebase. Desventajas: hay que vincular una tarjeta (con alerta de presupuesto de USD 1) y la primera petición después de un rato sin uso tarda ~1–3 s (*cold start*).
- **Flutter se mantiene:** la restricción de iOS es de Apple, no del framework. Con una Mac + iPhone + Apple ID gratuito se instala la app nativa sin pagar. Los compañeros prueban en Android vía APK.
- **MCP dentro del mismo servicio que la API:** expone los artículos al modelo mediante herramientas pequeñas (buscar → resúmenes cortos; obtener → texto completo solo si hace falta). Esto reduce los tokens que lee el modelo y es la evidencia de optimización de costo que pide el proyecto. Al estar en el mismo contenedor que la API, se despliega una sola vez y no hay un segundo servicio.
- **Solo la API habla con Supabase:** la app y el portal nunca tienen credenciales de la base de datos. Así evitamos sincronizar usuarios de Firebase con el auth/RLS de Supabase.
- **Un solo ambiente (prod):** sin staging, para no duplicar configuración.

---

## 2. Arquitectura

```
App Flutter (Android APK / iOS nativo) ─┐
                                        ├─► FastAPI en Cloud Run ──► Supabase Postgres (noticias)
Portal Next.js (Vercel) ────────────────┘     ├── REST  /api/...       Supabase Storage (imágenes)
          │                                   └── MCP   /mcp  ◄── LLM (proveedor por definir)
          └── ambos inician sesión con Firebase Auth (Google)
```

### Flujo de autenticación

1. La app (Flutter) o el portal (Next.js) inicia sesión con Google mediante Firebase Auth.
2. El cliente obtiene el **Firebase ID token** y lo envía a la API en `Authorization: Bearer <token>`.
3. La API lo verifica con `firebase-admin`.
4. Los administradores tienen el *custom claim* `admin: true`, que se asigna con un script (sección 5.1). Las rutas `/api/admin/*` exigen ese claim.

### Flujo del chat (genérico, independiente del proveedor)

1. La app envía la pregunta a `POST /api/chat` con su token.
2. La API llama al LLM y le da acceso a las herramientas MCP en `/mcp`, protegidas con un secreto compartido (`MCP_SECRET`).
3. El modelo usa las herramientas para buscar y leer artículos. La API devuelve la respuesta junto con las fuentes (IDs de artículos).

### Flujo de imágenes

El portal sube la imagen a la API (`POST /api/admin/images`). La API la guarda en el bucket público `news-images` de Supabase y registra la URL pública en la noticia. La app carga las imágenes desde esa URL.

### Endpoints base (esqueleto, se ajusta en la implementación)

| Método | Ruta | Auth | Uso |
|---|---|---|---|
| GET | `/health` | ninguna | chequeo / calentar el servicio |
| GET | `/api/news` | usuario | feed |
| GET | `/api/news/{id}` | usuario | lectura de una noticia |
| POST | `/api/chat` | usuario | chat de noticias |
| POST | `/api/admin/news` | admin | crear/publicar noticia |
| PUT | `/api/admin/news/{id}` | admin | editar noticia |
| POST | `/api/admin/images` | admin | subir imagen |
| — | `/mcp` | `MCP_SECRET` | servidor MCP (streamable HTTP) |

---

## 3. Estructura del repositorio (monorepo)

```
/mobile      App Flutter
/admin       Portal Next.js (TypeScript)
/api         FastAPI + servidor MCP + Dockerfile
/supabase    Migraciones SQL
/docs        Specs, decisiones, bitácora de costos (docs/costos.md)
```

**Nunca se suben al repo:** `.env`, `google-services.json` y `GoogleService-Info.plist` (opcional, son semipúblicos), el keystore de Android, `key.properties` ni ninguna llave de servicio. Todo eso va en `.gitignore`.

---

## 4. Variables de entorno

### API (`/api`, Cloud Run)

| Variable | Origen | Secreto |
|---|---|---|
| `SUPABASE_URL` | Supabase → Project Settings → API | no |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase → Project Settings → API keys (service_role / secret key) | **sí** |
| `SUPABASE_BUCKET` | `news-images` | no |
| `FIREBASE_PROJECT_ID` | Firebase → Configuración del proyecto | no |
| `MCP_SECRET` | generar: `python -c "import secrets;print(secrets.token_urlsafe(32))"` | **sí** |
| `ALLOWED_ORIGINS` | dominio de Vercel + `http://localhost:3000` | no |
| `LLM_API_KEY` | proveedor de IA (cuando se defina) | **sí** |

En Cloud Run, `firebase-admin` usa las credenciales por defecto del servicio, así que **no hace falta un archivo de service account**. En local se usa `gcloud auth application-default login`.

### Portal (`/admin`, Vercel)

| Variable | Origen |
|---|---|
| `NEXT_PUBLIC_FIREBASE_API_KEY` | Firebase → app Web |
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | Firebase → app Web |
| `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | Firebase → app Web |
| `NEXT_PUBLIC_FIREBASE_APP_ID` | Firebase → app Web |
| `NEXT_PUBLIC_API_URL` | URL del servicio de Cloud Run |

### App (`/mobile`)

La URL de la API se pasa al compilar: `--dart-define=API_URL=https://...run.app`. La configuración de Firebase sale de `flutterfire configure`.

---

## 5. Instrucciones de configuración (una sola vez)

> Orden recomendado: Firebase → Google Cloud / Cloud Run → Supabase → Vercel → App móvil.

### 5.1 Firebase (Auth)

1. Crear el proyecto en <https://console.firebase.google.com>. Esto también crea el proyecto de Google Cloud que usará Cloud Run.
2. **Authentication → Sign-in method → Google → Habilitar.**
3. **Agregar app Web** (para el portal) y copiar la configuración a las variables `NEXT_PUBLIC_FIREBASE_*`.
4. **Authentication → Settings → Authorized domains:** agregar el dominio de Vercel (`<proyecto>.vercel.app`).
5. Las apps Android e iOS se registran desde Flutter con `flutterfire configure` (ver 5.5).
6. **Script para asignar admin** (en `/api/scripts/set_admin.py`):
   ```python
   import sys, firebase_admin
   from firebase_admin import auth
   firebase_admin.initialize_app()
   user = auth.get_user_by_email(sys.argv[1])
   auth.set_custom_user_claims(user.uid, {"admin": True})
   print(f"{user.email} ahora es admin")
   ```
   Uso: `python scripts/set_admin.py correo@gmail.com`. El usuario debe haber iniciado sesión al menos una vez, y después debe cerrar sesión y volver a entrar para refrescar el token.

> Nota: al vincular facturación al proyecto (paso 5.2), Firebase pasa al plan Blaze. Las cuotas gratuitas de Auth son las mismas.

### 5.2 Google Cloud / Cloud Run (API + MCP)

1. Instalar Google Cloud CLI y ejecutar `gcloud init`. Seleccionar el proyecto de Firebase.
2. **Vincular una cuenta de facturación** al proyecto.
3. **Billing → Budgets & alerts → crear un presupuesto de USD 1** con alertas al 50%, 90% y 100%.
4. Habilitar las APIs:
   ```bash
   gcloud services enable run.googleapis.com cloudbuild.googleapis.com \
     artifactregistry.googleapis.com secretmanager.googleapis.com
   ```
5. Crear los secretos:
   ```bash
   printf "%s" "<service_role_key>" | gcloud secrets create supabase-service-key --data-file=-
   printf "%s" "<mcp_secret>"       | gcloud secrets create mcp-secret --data-file=-
   ```
6. Dar acceso a los secretos a la cuenta de servicio de Cloud Run:
   ```bash
   gcloud projects add-iam-policy-binding <PROJECT_ID> \
     --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" \
     --role="roles/secretmanager.secretAccessor"
   ```
7. **Desplegar** desde la raíz del repo (requiere `api/Dockerfile`; la app escucha en `$PORT`):
   ```bash
   gcloud run deploy noticias-api --source api --region us-central1 \
     --allow-unauthenticated --min-instances 0 --max-instances 2 \
     --set-env-vars SUPABASE_URL=<url>,SUPABASE_BUCKET=news-images,FIREBASE_PROJECT_ID=<id>,ALLOWED_ORIGINS=<origenes> \
     --set-secrets SUPABASE_SERVICE_ROLE_KEY=supabase-service-key:latest,MCP_SECRET=mcp-secret:latest
   ```
   `--allow-unauthenticated` es correcto aquí: la autenticación la hace nuestra API con tokens de Firebase.
8. Probar: `curl https://<url>.run.app/health`.
9. **Limpieza:** cada despliegue con `--source` deja una imagen en Artifact Registry, cuyo free tier es de 0.5 GB. Borrar las imágenes viejas de vez en cuando, o configurar una *cleanup policy* que conserve las últimas 2.

Para volver a desplegar basta con repetir el paso 7 (las variables y los secretos ya quedan guardados, así que se pueden omitir).

### 5.3 Supabase (Postgres + Storage)

1. Crear el proyecto en <https://supabase.com> (plan Free), en una región cercana a `us-central1` (p. ej. `us-east-1`).
2. Copiar `SUPABASE_URL` y la llave **service_role / secret** desde Project Settings → API. **Esta llave nunca va en la app ni en el portal.**
3. **Storage → New bucket → `news-images` → Public.**
4. Migraciones: instalar Supabase CLI y, desde la raíz:
   ```bash
   supabase init            # crea /supabase
   supabase link --project-ref <ref>
   supabase migration new init
   # editar el SQL generado en supabase/migrations/
   supabase db push
   ```
   (Alternativa rápida: pegar el SQL en el SQL Editor del dashboard, pero guardarlo también en `/supabase/migrations`.)
5. ⚠️ **El plan Free pausa el proyecto tras 7 días sin actividad.** Antes de la demo, abrir el dashboard y confirmar que el proyecto está activo.

Límites Free: 500 MB de base de datos y 1 GB de storage. Comprimir las imágenes antes de subirlas (p. ej. JPEG/WebP ≤ 300 KB).

### 5.4 Vercel (portal admin)

1. Subir el repo a GitHub.
2. En Vercel: **Add New → Project → importar el repo → Root Directory: `admin`** (el framework se detecta como Next.js).
3. Cargar las variables `NEXT_PUBLIC_*` (sección 4).
4. Cada push a `main` despliega solo.
5. Agregar el dominio de Vercel a: (a) Authorized domains de Firebase y (b) `ALLOWED_ORIGINS` de la API (y volver a desplegar la API).

### 5.5 App móvil (Flutter)

**Configuración común**

```bash
npm install -g firebase-tools && firebase login
dart pub global activate flutterfire_cli
cd mobile
flutterfire configure     # registra las apps Android/iOS y genera firebase_options.dart
flutter pub add firebase_core firebase_auth google_sign_in http
```

Usar el mismo identificador en ambas plataformas, p. ej. `com.equipo.noticias` (applicationId en Android, Bundle ID en iOS).

**Android (APK para el equipo y los compañeros)**

1. Crear un keystore de release **una sola vez** y compartirlo de forma privada con el equipo (no en el repo):
   ```bash
   keytool -genkey -v -keystore noticias-release.jks -keyalg RSA -keysize 2048 -validity 10000 -alias noticias
   ```
   Configurarlo en `android/key.properties` + `android/app/build.gradle` (guía oficial de Flutter: "Build and release an Android app").
2. ⚠️ **Registrar el SHA-1 en Firebase** (Configuración del proyecto → app Android → Agregar huella). Sin esto **Google Sign-In falla**.
   - SHA-1 de release: `keytool -list -v -keystore noticias-release.jks -alias noticias`
   - SHA-1 de debug **de cada integrante**: `keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android`
   - Después de agregar huellas, volver a descargar `google-services.json` (o volver a correr `flutterfire configure`).
3. Compilar:
   ```bash
   flutter build apk --release --dart-define=API_URL=https://<url>.run.app
   ```
   El APK queda en `build/app/outputs/flutter-apk/app-release.apk`.
4. Distribuir con un link (GitHub Releases o Drive) más un QR. Los compañeros deben permitir "instalar apps de fuentes desconocidas".

**iOS (nativo, solo el iPhone del equipo; requiere Mac)**

1. Instalar Xcode y CocoaPods. En Xcode → Settings → Accounts, iniciar sesión con un Apple ID normal (gratuito).
2. `open ios/Runner.xcworkspace` → target Runner → **Signing & Capabilities → Team: (Personal Team)**, con el Bundle ID igual al registrado en Firebase.
3. Para Google Sign-In: agregar el `REVERSED_CLIENT_ID` de `GoogleService-Info.plist` como URL Scheme en `ios/Runner/Info.plist` (seguir el README del paquete `google_sign_in`).
4. Conectar el iPhone por cable y activar **Modo desarrollador** (Ajustes → Privacidad y seguridad).
5. ```bash
   flutter run --release --dart-define=API_URL=https://<url>.run.app
   ```
6. La primera vez, en el iPhone: **Ajustes → General → VPN y gestión de dispositivos → confiar en el desarrollador**.
7. ⚠️ **La instalación expira a los 7 días.** Reinstalar (paso 5) en la semana de la demo.

---

## 6. Desarrollo local

```bash
# API
cd api && python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
gcloud auth application-default login     # para firebase-admin en local
uvicorn app.main:app --reload --port 8000

# Portal
cd admin && npm install && npm run dev    # http://localhost:3000

# App (emulador Android: el host se ve como 10.0.2.2)
cd mobile && flutter run --dart-define=API_URL=http://10.0.2.2:8000
# En un teléfono físico usar la IP LAN de la PC o directamente la URL de Cloud Run
```

---

## 7. Costos y control

| Servicio | Plan | Gasto esperado | Riesgo / control |
|---|---|---|---|
| Cloud Run | Free tier | $0 | Alerta de presupuesto de USD 1, `max-instances 2` |
| Artifact Registry | 0.5 GB gratis | $0 | Borrar imágenes viejas / cleanup policy |
| Supabase | Free | $0 | Se pausa a los 7 días sin uso |
| Vercel | Hobby | $0 | — |
| Firebase Auth | Blaze (cuotas gratis) | $0 | — |
| IA (LLM / imágenes) | Prepago | ≤ USD 20 | Bitácora en `docs/costos.md`: gasto acumulado, saldo y costo por función; **reservar saldo para la demo** |

---

## 8. Checklist previo a la demo

- [ ] Supabase activo (abrir el dashboard el día anterior).
- [ ] App iOS reinstalada hace menos de 7 días.
- [ ] APK final publicado con link + QR; probar el login con Google en un Android que no sea del equipo.
- [ ] Durante la demo: `gcloud run services update noticias-api --region us-central1 --min-instances 1` para evitar cold starts. **Regresarlo a 0 al terminar.**
- [ ] `curl /health` responde.
- [ ] Saldo de IA verificado y suficiente.
- [ ] Al menos una cuenta admin asignada y probada en el portal.

---

## 9. Pendientes (siguientes specs)

1. Elegir el proveedor de LLM y conectarlo a MCP.
2. Esquema de datos de noticias: fuentes, estado de verificación, alcance geográfico, imagen y etiqueta de IA.
3. Modelo de relevancia y ubicaciones simuladas.
4. Flujo de imágenes faltantes (buscar o generar, y cómo etiquetarlas).
5. Herramientas MCP concretas y estrategia de ahorro de tokens.
