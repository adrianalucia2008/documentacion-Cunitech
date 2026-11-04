# Runbook de Incidentes — CUNITECH

Guía para diagnosticar y resolver fallos del sistema CUNITECH en **menos de 10 minutos**. Está pensada para alguien que no conoce el proyecto: siga el orden indicado y no cambie nada hasta tener el diagnóstico.

> Reemplace `<api>` y `<nginx>` por los nombres reales de los servicios que muestra `docker compose ps`.
> En Windows (PowerShell) use `curl.exe` en lugar de `curl`.

---

## 0. Mapa de la arquitectura

| Componente | Tecnología | Puerto | Dónde se ve |
|---|---|---|---|
| Cliente | Flutter (Web / Android) | 60144 en desarrollo (`flutter run -d chrome`) | Navegador / dispositivo |
| Proxy inverso | Nginx | **80** | `http://localhost` |
| API de negocio | Python + FastAPI en Docker | **8000** (interno, detrás de Nginx) | A través de Nginx |
| Logs | Dozzle | **8080** | `http://localhost:8080` |
| Pruebas de carga | Locust | **8089** | `http://localhost:8089` |
| Base de datos y Auth | Supabase (PostgreSQL en la nube) | — | Panel de Supabase |
| Lógica de servidor / IA | Supabase Edge Functions (`create-user`, `generate-insights`, `identify-breed`, `chat-assistant`) | — | Supabase → Edge Functions → Logs |

Flujo normal: **Cliente → Nginx (80) → API (8000) → Supabase**.
Las Edge Functions las llama el cliente directamente a Supabase.

---

## 1. Diagnóstico rápido (primeros 2 minutos)

Ejecute esto **en orden** y anote qué falla:

```bash
# 1. ¿Alguien modificó archivos? (los fallos inyectados casi siempre aparecen aquí)
git status
git diff

# 2. Estado de los contenedores
docker compose ps

# 3. ¿La configuración del compose es válida?
docker compose config

# 4. Salud del sistema a través de Nginx
curl.exe -i http://localhost/health

# 5. Logs recientes
docker compose logs --tail=50 <api>
docker compose logs --tail=50 <nginx>

# 6. Consumo de recursos
docker stats --no-stream
```

**Cómo leer el resultado:**

| Lo que ve | Vaya a |
|---|---|
| Un contenedor en `Exited`, `Restarting` o `unhealthy` | INC-01 |
| `/health` responde error o no responde | INC-02 |
| Nginx devuelve 502 / 504 | INC-03 |
| Nada responde en el puerto 80 | INC-04 |
| Solo algunas réplicas están caídas | INC-05 |
| Dozzle o Locust no abren | INC-06 |
| La app no conecta o no inicia sesión | INC-07, INC-09 |
| Una función de IA o creación de usuarios falla | INC-08 |
| `flutter` no compila | INC-10 |

> **Atajo:** si `git diff` muestra cambios en `docker-compose.yml`, `nginx.conf` o el código de la API, ya encontró el origen. Pase directo a la sección 3 (Revertir cambios).

---

## 2. Incidentes

### INC-01 — Contenedor de la API caído, reiniciándose o sin memoria

- **Síntomas:** `docker compose ps` muestra `Exited`, `Restarting` o `unhealthy`. La API deja de responder. En Dozzle se ven reinicios continuos.
- **Causa probable:** límite de memoria demasiado bajo (el contenedor muere por OOM), error en el comando de arranque o variable de entorno faltante.
- **Verificar:**
  ```bash
  docker inspect <api> --format "OOMKilled={{.State.OOMKilled}} Memoria={{.HostConfig.Memory}} ExitCode={{.State.ExitCode}}"
  docker compose logs --tail=80 <api>
  ```
  Si `OOMKilled=true`, el límite de RAM es insuficiente.
- **Solución:**
  1. Abra `docker-compose.yml` y busque `mem_limit` o `deploy.resources.limits.memory`.
  2. Corrija el valor (o restaure el original con `git checkout -- docker-compose.yml`).
  3. Reaplique:
     ```bash
     docker compose up -d --force-recreate <api>
     ```
- **Confirmar:** `docker compose ps` muestra la API en `Up` / `healthy` y `curl.exe -i http://localhost/health` responde 200.

---

### INC-02 — `/health` falla o devuelve error

- **Síntomas:** `curl.exe -i http://localhost/health` devuelve 404, 500 o no responde, aunque los contenedores estén `Up`.
- **Causa probable:** ruta `/health` modificada o eliminada en el código de la API, o Nginx no la reenvía correctamente.
- **Verificar:**
  ```bash
  git diff -- <ruta-del-codigo-de-la-api>
  docker compose logs --tail=50 <api>
  docker compose exec <api> python -c "import urllib.request;print(urllib.request.urlopen('http://localhost:8000/health').read())"
  ```
  Si responde dentro del contenedor pero no por el puerto 80, el problema está en Nginx (vaya a INC-03).
- **Solución:** restaure la ruta (`git checkout -- <archivo>`) o corrija el código y reconstruya:
  ```bash
  docker compose up -d --build <api>
  ```
- **Confirmar:** `curl.exe -i http://localhost/health` devuelve `200` con un JSON de estado `ok`.

---

### INC-03 — Nginx devuelve 502 Bad Gateway o 504 Gateway Timeout

- **Síntomas:** el navegador o `curl` muestran 502/504. Los logs de Nginx indican `connect() failed` o `upstream timed out`.
- **Causa probable:** el `upstream` de `nginx.conf` apunta a un puerto o nombre de servicio incorrecto (la API escucha en **8000**), o la API está caída.
- **Verificar:**
  ```bash
  docker compose logs --tail=50 <nginx>
  docker compose exec <nginx> nginx -t
  ```
  Revise en `nginx.conf` que el `upstream` / `proxy_pass` apunte al servicio de la API y al puerto **8000**.
- **Solución:**
  1. Corrija `nginx.conf` (o `git checkout -- nginx.conf`).
  2. Valide y recargue:
     ```bash
     docker compose exec <nginx> nginx -t
     docker compose restart <nginx>
     ```
- **Confirmar:** `curl.exe -i http://localhost/health` ya no devuelve 502/504.

---

### INC-04 — Nada responde en el puerto 80

- **Síntomas:** `curl.exe http://localhost` falla con "connection refused" o se queda sin respuesta, aunque el contenedor de Nginx esté `Up`.
- **Causa probable:** el mapeo de puertos del compose (`ports:`) o el `listen` de `nginx.conf` fue cambiado a otro puerto; o hay otro programa usando el 80.
- **Verificar:**
  ```bash
  docker compose ps                         # columna PORTS de Nginx
  docker compose config | findstr /i "ports published"
  netstat -ano | findstr :80
  ```
- **Solución:**
  - Si el mapeo es incorrecto, deje `"80:80"` en `docker-compose.yml` y el `listen 80;` en `nginx.conf`, luego `docker compose up -d --force-recreate <nginx>`.
  - Si otro proceso ocupa el 80, identifíquelo con `tasklist /FI "PID eq <PID>"` y deténgalo.
- **Confirmar:** `curl.exe -i http://localhost/health` responde 200.

---

### INC-05 — Una réplica de la API está caída (si el compose define réplicas)

- **Síntomas:** `docker compose ps` muestra menos instancias de la API que las definidas. Parte de las peticiones fallan o aumenta la latencia.
- **Causa probable:** una réplica murió por memoria, por error de arranque o fue detenida manualmente.
- **Verificar:**
  ```bash
  docker compose ps
  docker compose logs --tail=50 <api>
  ```
- **Solución:** restaure el número de réplicas y recree:
  ```bash
  docker compose up -d --scale <api>=<N> --no-recreate
  ```
  (use el número de réplicas definido en el compose). El sistema debe seguir sirviendo mientras tanto gracias a Nginx.
- **Confirmar:** `docker compose ps` muestra todas las réplicas en `Up` / `healthy`.

---

### INC-06 — Dozzle o Locust no abren

- **Síntomas:** `http://localhost:8080` (Dozzle) o `http://localhost:8089` (Locust) no cargan.
- **Causa probable:** contenedor detenido, puerto cambiado en el compose, o Dozzle sin acceso al socket de Docker.
- **Verificar:**
  ```bash
  docker compose ps
  docker compose logs --tail=30 <dozzle>
  docker compose logs --tail=30 <locust>
  ```
- **Solución:** corrija el mapeo de puertos (8080 para Dozzle, 8089 para Locust) y ejecute `docker compose up -d <servicio>`.
- **Confirmar:** ambas URL cargan en el navegador.

---

### INC-07 — La app no conecta con Supabase

- **Síntomas:** la app queda en blanco o con error de red al iniciar; en la consola de Flutter **no** aparece `Supabase init completed`.
- **Causa probable:** URL o `anonKey` incorrectas en `Supabase.initialize(...)`, proyecto pausado o sin internet.
- **Verificar:**
  1. Revise `lib/main.dart` (`Supabase.initialize`): la URL debe tener la forma `https://<project-ref>.supabase.co`.
  2. Compare URL y *publishable/anon key* con **Supabase → Project Settings → API**.
  3. Confirme que el proyecto no esté pausado en el panel.
- **Solución:** corrija los valores, guarde y reinicie con `flutter run -d chrome` (reinicio completo, no hot reload).
- **Confirmar:** en consola aparece `supabase.supabase_flutter: INFO: ***** Supabase init completed *****`.

---

### INC-08 — Una Edge Function falla (IA, crear usuario, chat)

- **Síntomas:** error al generar insights, identificar raza, usar el chat o crear un usuario desde el panel de admin.
- **Causa probable:** función no desplegada, secreto faltante (`GEMINI_API_KEY`, `RESEND_API_KEY`, `APP_URL`), o permisos insuficientes.
- **Verificar:**
  ```bash
  supabase functions logs <nombre-de-la-funcion>
  supabase secrets list
  ```
  Interprete el código:
  | Respuesta | Significado |
  |---|---|
  | `401 No autenticado / Token inválido` | La sesión expiró: cierre sesión e inicie de nuevo. |
  | `403 Acceso denegado: se requiere rol admin` | El usuario no es admin (solo aplica a `create-user`). |
  | Resultado vacío o sin IA | Falta `GEMINI_API_KEY`. |
  | Error de envío de correo | Falta o es inválida `RESEND_API_KEY`. |
- **Solución:**
  ```bash
  supabase secrets set GEMINI_API_KEY=<valor>
  supabase functions deploy <nombre-de-la-funcion>
  ```
- **Confirmar:** repita la acción en la app y verifique la invocación exitosa en Supabase → Edge Functions → Logs.

---

### INC-09 — No se puede iniciar sesión

- **Síntomas:** "Credenciales incorrectas" o la app no avanza tras el login.
- **Causa probable:** usuario inexistente, contraseña olvidada, correo sin confirmar, o el usuario no tiene fila en la tabla `usuario` (rol).
- **Verificar:**
  1. Supabase → **Authentication → Users**: ¿existe el correo? ¿está confirmado?
  2. Supabase → **Table Editor → `usuario`**: ¿existe una fila con ese `id_usuario` y un `rol` (`admin`, `veterinario`, `ayudante`)?
- **Solución:**
  - Contraseña olvidada: en Authentication → Users use **Send password recovery**.
  - Falta el usuario: créelo con **Add user → Create new user** (marque *Auto Confirm User*) y agregue su fila en `usuario`.
- **Confirmar:** el usuario ingresa y ve el panel acorde con su rol.

---

### INC-10 — Error de compilación o dependencias de Flutter

- **Síntomas:** `flutter run` falla con errores de paquetes, de Gradle o de versión.
- **Causa probable:** dependencias sin instalar tras un `git pull`, caché corrupta o cambio en `pubspec.yaml`.
- **Verificar:** `flutter doctor` y `flutter analyze`.
- **Solución:**
  ```bash
  flutter clean
  flutter pub get
  flutter run -d chrome
  ```
- **Confirmar:** la app compila y abre la pantalla de login.

---

## 3. Revertir cambios (acción más rápida ante un fallo inyectado)

```bash
git status
git diff                                        # ver qué se modificó
git checkout -- docker-compose.yml nginx.conf   # restaurar archivos de infraestructura
git checkout -- <archivo-de-la-api>             # restaurar código modificado
docker compose up -d --build --force-recreate   # reaplicar
```

Si necesita conservar los cambios para analizarlos: `git stash` (y después `git stash pop`).

---

## 4. Registro del incidente

Complete durante la atención (útil para la evidencia):

| Campo | Valor |
|---|---|
| Hora de inicio | |
| Síntoma observado | |
| Servicio afectado | |
| Causa encontrada | |
| Acción aplicada | |
| Hora de resolución | |
| Tiempo total (meta: < 10 min) | |

---

## 5. Comandos de referencia

| Acción | Comando |
|---|---|
| Ver contenedores | `docker compose ps` |
| Logs de un servicio | `docker compose logs --tail=50 <servicio>` |
| Reiniciar un servicio | `docker compose restart <servicio>` |
| Reconstruir y levantar | `docker compose up -d --build` |
| Recursos en uso | `docker stats --no-stream` |
| Validar compose | `docker compose config` |
| Validar Nginx | `docker compose exec <nginx> nginx -t` |
| Apagar todo | `docker compose down` |
| Logs de una Edge Function | `supabase functions logs <nombre>` |
| Desplegar una Edge Function | `supabase functions deploy <nombre>` |
