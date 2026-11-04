# Runbook de Incidentes — CUNITECH

Guía para diagnosticar y resolver fallos de CUNITECH en **menos de 10 minutos**, escrita para alguien que no conoce el proyecto. Siga el orden indicado y no cambie nada hasta tener el diagnóstico.

> En Windows (PowerShell) use `curl.exe` en lugar de `curl`.
> Cada paso indica su nivel de acceso:
> - **[LOCAL]** se hace solo con el código y el computador.
> - **[SUPABASE]** requiere acceso al panel del proyecto Supabase (`aswhtehpmmeetplnskmz`) o a su CLI.

---

## 0. Mapa del sistema

```
Cliente Flutter (localhost:60144)
   │
   ├─► Edge Function "login" ───────► Supabase Auth
   ├─► Edge Functions IA / usuarios ► Gemini / Resend
   └─► PostgreSQL (tablas con RLS) ─► Supabase
```

| Elemento | Dónde está | Dato clave |
|---|---|---|
| Configuración de conexión | `lib/main.dart` → `Supabase.initialize(...)` | URL `https://aswhtehpmmeetplnskmz.supabase.co` |
| Puerto de la app web | `flutter run -d chrome --web-port=60144` | Debe ser **60144** (CORS) |
| Inicio de sesión | Edge Function `login` | Sin ella no hay login |
| Funciones | `login`, `create-user`, `generate-insights`, `identify-breed`, `chat-assistant`, `health` | Desplegadas en Supabase |
| Roles | tabla `usuario` (`id_usuario`, `rol`) | `admin`, `veterinario`, `ayudante` |
| Políticas de datos | migración `018_rls_compartida_criadero.sql` | Todos los autenticados comparten datos |

---

## 1. Diagnóstico rápido (primeros 2 minutos)

```bash
# 1. [LOCAL] ¿Qué archivos fueron modificados? (los fallos inyectados casi siempre aparecen aquí)
git status
git diff

# 2. [LOCAL] ¿El entorno Flutter está sano?
flutter --version
flutter doctor

# 3. [LOCAL] Estado del backend en la nube
curl.exe -i https://aswhtehpmmeetplnskmz.supabase.co/functions/v1/health

# 4. [LOCAL] Compilar y mirar el primer error real
flutter analyze
flutter run -d chrome --web-port=60144
```

Además, en Chrome abra las herramientas de desarrollo (**F12**) y mire la pestaña **Console** y **Network**: ahí aparecen los errores de conexión, CORS y de permisos.

**Cómo leer el resultado:**

| Lo que ve | Vaya a |
|---|---|
| `flutter pub get` o la compilación falla con errores de paquetes o de versión | INC-01 |
| Error de compilación en un archivo `.dart` | INC-02 |
| No aparece `Supabase init completed`, o todo falla por red | INC-03 |
| El login funciona pero la IA / crear usuario fallan y la consola dice *CORS* | INC-04 |
| No se puede iniciar sesión | INC-05 |
| `/health` no responde 200 o dice `"status":"error"` | INC-06 |
| Función de IA, chat o crear usuario devuelve error 401/403/500 | INC-07 |
| Listas vacías o error de permisos al leer/guardar datos | INC-08 |
| Faltan menús o botones de administración | INC-09 |
| Micrófono, cámara o notificaciones no funcionan | INC-10 |

> **Atajo:** si `git diff` muestra cambios en `lib/main.dart`, `pubspec.yaml` o cualquier archivo en `lib/`, ya encontró el origen. Pase a la sección 3 (Revertir cambios).

---

## 2. Incidentes

### INC-01 — La app no compila: dependencias o versión de Flutter

- **Síntomas:** `flutter pub get` o `flutter run` fallan con errores de paquetes, de resolución de versiones o *"requires SDK version"*.
- **Causa probable:** `pubspec.yaml` modificado, dependencias sin instalar o Flutter/Dart demasiado antiguo (el proyecto requiere **Dart ^3.11.4**).
- **Verificar [LOCAL]:**
  ```bash
  git diff -- pubspec.yaml pubspec.lock
  flutter --version
  ```
- **Solución [LOCAL]:**
  ```bash
  git checkout -- pubspec.yaml pubspec.lock
  flutter clean
  flutter pub get
  ```
  Si la versión de Dart es menor a 3.11.4, actualice Flutter con `flutter upgrade`.
- **Confirmar:** `flutter pub get` termina sin errores y `flutter run -d chrome --web-port=60144` compila.

---

### INC-02 — Error de compilación en un archivo Dart

- **Síntomas:** la compilación se detiene con un error que apunta a un archivo y línea de `lib/` (por ejemplo, paréntesis sin cerrar, método inexistente, import roto).
- **Causa probable:** un archivo de `lib/` fue alterado.
- **Verificar [LOCAL]:**
  ```bash
  flutter analyze
  git diff -- lib
  ```
  El primer error de `flutter analyze` indica el archivo afectado.
- **Solución [LOCAL]:**
  ```bash
  git checkout -- <archivo-afectado>
  flutter run -d chrome --web-port=60144
  ```
- **Confirmar:** `flutter analyze` no muestra errores (los avisos `info`/`warning` son aceptables) y la app abre.

---

### INC-03 — La app no conecta con Supabase

- **Síntomas:** pantalla en blanco o errores de red al iniciar; **no** aparece `Supabase init completed`; en Network (F12) las peticiones fallan o devuelven 401.
- **Causa probable:** URL o llave alteradas en `lib/main.dart`, falta de internet, o el proyecto de Supabase está pausado.
- **Verificar [LOCAL]:**
  ```bash
  git diff -- lib/main.dart
  curl.exe -i https://aswhtehpmmeetplnskmz.supabase.co/functions/v1/health
  ```
  La URL en `Supabase.initialize` debe ser exactamente `https://aswhtehpmmeetplnskmz.supabase.co`. Si `/health` no responde, revise su conexión a internet.
- **Solución:**
  1. **[LOCAL]** Restaure la configuración: `git checkout -- lib/main.dart`
  2. **[SUPABASE]** Si `/health` sigue sin responder, entre al panel y verifique que el proyecto no esté **pausado** (plan gratuito); use *Restore project*.
  3. Reinicie con `flutter run -d chrome --web-port=60144` (reinicio completo, no *hot reload*).
- **Confirmar:** en consola aparece `supabase.supabase_flutter: INFO: ***** Supabase init completed *****`.

---

### INC-04 — La IA o crear usuarios fallan por CORS (puerto incorrecto)

- **Síntomas:** el login funciona, pero el chat, los insights, la identificación de raza o crear usuarios fallan. En la consola de Chrome (F12) aparece *"blocked by CORS policy"*.
- **Causa probable:** la app se ejecuta en un puerto distinto de **60144**. Esas funciones solo aceptan el origen `http://localhost:60144`.
- **Verificar [LOCAL]:** mire la barra de direcciones del navegador: debe decir `localhost:60144`.
- **Solución:**
  - **[LOCAL]** Detenga la app y ejecútela con el puerto correcto:
    ```bash
    flutter run -d chrome --web-port=60144
    ```
  - **[SUPABASE]** Alternativa: `supabase secrets set APP_URL=http://localhost:<puerto>` y desplegar de nuevo las funciones.
- **Confirmar:** el asistente de chat responde y no hay errores CORS en la consola.

---

### INC-05 — No se puede iniciar sesión

- **Síntomas:** mensaje *"Credenciales incorrectas"*, *"No se pudo iniciar sesión. Intenta de nuevo."* o la app no avanza tras el login.
- **Causa probable:** credenciales mal escritas, usuario inexistente o sin confirmar, función `login` caída, o el usuario no tiene fila en la tabla `usuario`.
- **Verificar:**
  1. **[LOCAL]** Reescriba correo y contraseña de la cuenta de prueba (sección 4.5 del manual). Revise que no haya espacios.
  2. **[LOCAL]** Compruebe el backend: `curl.exe -i https://aswhtehpmmeetplnskmz.supabase.co/functions/v1/health`.
  3. **[LOCAL]** Con F12 → Network, mire la respuesta de la petición a `login`: 401 = credenciales; 500 = error del servidor.
  4. **[SUPABASE]** *Authentication → Users*: ¿existe el correo y está confirmado? *Table Editor → `usuario`*: ¿hay una fila con su `id_usuario` y un `rol`?
- **Solución:**
  - **[SUPABASE]** Contraseña olvidada: *Authentication → Users →* los tres puntos del usuario → **Send password recovery**.
  - **[SUPABASE]** Falta la fila de rol: en el *SQL Editor* ejecute
    ```sql
    update public.usuario set rol = 'admin'
    where id_usuario = (select id from auth.users where lower(email) = lower('<correo>'));
    ```
    (si la fila no existe, debe insertarse con el mismo `id` del usuario de `auth.users`).
  - **[SUPABASE]** Función `login` caída: `supabase functions deploy login` y revise `supabase functions logs login`.
- **Confirmar:** el usuario entra al panel principal.

---

### INC-06 — `/health` no responde o dice `status: error`

- **Síntomas:** `curl.exe -i https://aswhtehpmmeetplnskmz.supabase.co/functions/v1/health` devuelve error, `404`, o un JSON con `"status":"error"` / `"database":"disconnected"`.
- **Causa probable:** función `health` no desplegada, proyecto pausado, o la tabla `system_logs` inaccesible.
- **Verificar:**
  1. **[LOCAL]** Repita el `curl` y lea el código HTTP y el JSON.
  2. **[SUPABASE]** Panel → *Edge Functions → health → Logs*; confirme que el proyecto esté activo y que exista la tabla `system_logs`.
- **Solución [SUPABASE]:**
  ```bash
  supabase functions deploy health
  ```
  y, si el proyecto estaba pausado, restáurelo desde el panel.
- **Confirmar:** el `curl` devuelve `200` con `"status":"ok"` y `"database":"connected"`.

---

### INC-07 — Falla una Edge Function (IA, chat, crear usuario)

- **Síntomas:** error al generar insights, al usar el chat, al identificar la raza o al crear un usuario desde el panel de administración.
- **Causa probable:** función no desplegada, secreto faltante (`GEMINI_API_KEY`, `RESEND_API_KEY`, `APP_URL`), sesión vencida o permisos insuficientes.
- **Verificar:** en F12 → Network, abra la petición fallida y lea el código:

  | Respuesta | Significado y acción |
  |---|---|
  | `401 No autenticado` / `Token inválido` | La sesión venció: cierre sesión e ingrese de nuevo. |
  | `403 Acceso denegado: se requiere rol admin` | Solo el rol `admin` puede crear usuarios: use la cuenta de administrador. |
  | `400` (correo, nombre o rol inválidos) | Corrija los datos del formulario. |
  | Error de CORS | Ver INC-04. |
  | `500` o respuesta vacía de IA | Posible secreto faltante o cuota de Gemini agotada. |

  **[SUPABASE]** Para ver la causa exacta: `supabase functions logs <nombre>` o el panel *Edge Functions → Logs*, y `supabase secrets list`.
- **Solución [SUPABASE]:**
  ```bash
  supabase secrets set GEMINI_API_KEY=<valor>
  supabase functions deploy <nombre>
  ```
- **Confirmar:** repita la acción en la app: responde sin error y la invocación aparece exitosa en los logs.

---

### INC-08 — Las listas salen vacías o hay errores de permisos

- **Síntomas:** *Conejos*, *Reproducción* u otras pantallas salen vacías, o al guardar aparece un error de permisos / *row-level security*.
- **Causa probable:** sesión no iniciada o vencida, o las políticas de acceso (RLS) fueron modificadas o eliminadas.
- **Verificar:**
  1. **[LOCAL]** Cierre sesión e ingrese de nuevo; confirme que está autenticado.
  2. **[SUPABASE]** *Authentication → Policies*: las tablas `conejo`, `alimentacion`, `reproduccion`, `births`, `kits` y `metas_mensuales` deben tener la política `authenticated_all`.
- **Solución [SUPABASE]:** ejecute en el *SQL Editor* el contenido de `supabase/migrations/018_rls_compartida_criadero.sql` (es idempotente: se puede repetir sin riesgo).
- **Confirmar:** las listas cargan y se pueden crear, editar y eliminar registros.

---

### INC-09 — No aparecen las opciones de administración o de edición

- **Síntomas:** el usuario no ve la gestión de usuarios, el botón de agregar conejo u otras acciones.
- **Causa probable:** el rol del usuario en la tabla `usuario` no es el esperado (`admin`, `veterinario`, `ayudante`).
- **Verificar [SUPABASE]:**
  ```sql
  select u.email, p.rol
  from auth.users u join public.usuario p on p.id_usuario = u.id
  where lower(u.email) = lower('<correo>');
  ```
- **Solución [SUPABASE]:**
  ```sql
  update public.usuario set rol = 'admin'
  where id_usuario = (select id from auth.users where lower(email) = lower('<correo>'));
  ```
  Después cierre sesión y vuelva a entrar.
- **Confirmar:** aparecen las pantallas y botones correspondientes al rol.

---

### INC-10 — Micrófono, cámara o notificaciones no funcionan

- **Síntomas:** el asistente de voz no escucha, la identificación por foto no abre la cámara o no llegan notificaciones.
- **Causa probable:** permisos del navegador o del dispositivo denegados.
- **Verificar y solucionar [LOCAL]:**
  1. En Chrome, haga clic en el candado junto a la URL → permita **Micrófono**, **Cámara** y **Notificaciones** para `localhost`.
  2. Recargue la página (o reinicie la app con `flutter run`).
  3. En Android: *Ajustes → Aplicaciones → CUNITECH → Permisos*.
- **Confirmar:** el asistente de voz transcribe y la cámara o galería se abre.

---

## 3. Revertir cambios (acción más rápida ante un fallo inyectado)

```bash
git status
git diff                              # ver qué se modificó
git checkout -- lib/main.dart pubspec.yaml pubspec.lock
git checkout -- <otro-archivo>        # restaurar un archivo concreto
flutter clean
flutter pub get
flutter run -d chrome --web-port=60144
```

Si necesita conservar los cambios para analizarlos antes: `git stash` (y luego `git stash pop`).

---

## 4. Registro del incidente

Complete durante la atención:

| Campo | Valor |
|---|---|
| Hora de inicio | |
| Síntoma observado | |
| Componente afectado | |
| Causa encontrada | |
| Acción aplicada | |
| Hora de resolución | |
| Tiempo total (meta: < 10 min) | |

---

## 5. Comandos de referencia

| Acción | Comando |
|---|---|
| Ver cambios locales | `git status` / `git diff` |
| Restaurar un archivo | `git checkout -- <archivo>` |
| Analizar el código | `flutter analyze` |
| Reinstalar dependencias | `flutter clean` y `flutter pub get` |
| Ejecutar la app web | `flutter run -d chrome --web-port=60144` |
| Estado del backend | `curl -i https://aswhtehpmmeetplnskmz.supabase.co/functions/v1/health` |
| Logs de una función **[SUPABASE]** | `supabase functions logs <nombre>` |
| Desplegar una función **[SUPABASE]** | `supabase functions deploy <nombre>` |
| Ver / definir secretos **[SUPABASE]** | `supabase secrets list` / `supabase secrets set NOMBRE=valor` |
