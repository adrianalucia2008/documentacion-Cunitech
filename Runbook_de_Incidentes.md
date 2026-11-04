# Runbook de Operaciones e Incidentes — CUNITECH

**Propósito:** Este documento establece los procedimientos operativos estándar (SOP) para el equipo de soporte y desarrollo de CUNITECH. Proporciona una guía de acción rápida para el diagnóstico y resolución de incidentes en el entorno del sistema, contemplando el cliente Flutter, la base de datos PostgreSQL y las Edge Functions de Supabase.

---

## Índice de Incidentes
1. Incidente 1: Ausencia de datos o errores 401/403 en Flutter
2. Incidente 2: Fallo en la creación de usuarios (Edge Function)
3. Incidente 3: Fallo en la carga del módulo de IA (Insights)
4. Incidente 4: Pantalla en blanco en Flutter Web (CORS/Puertos)
5. Incidente 5: Timeout en la pantalla de inicio de sesión
6. Protocolo de Contingencia y Modo Supervivencia

---

### Incidente 1: Ausencia de datos o errores 401/403 en Flutter
> **Contexto:** El usuario completa el inicio de sesión exitosamente, pero al acceder a los módulos de datos (ej. lista de conejos), la interfaz se muestra vacía o despliega un mensaje de "No autorizado". Esto se debe típicamente a configuraciones de Row Level Security (RLS) o problemas con el token de sesión.

**Diagnóstico:**
1. Abrir la consola de Flutter (VS Code/Android Studio) o las herramientas de desarrollador del navegador (F12 en entorno Web).
2. Identificar excepciones de tipo `PostgrestException` con códigos HTTP `401` (No autenticado) o `403` (Prohibido).
3. Verificar los logs de la API en el panel de Supabase (Logs > API Edge Logs) para aislar el origen del rechazo.

**Plan de Acción:**
*   **Código 401:** Indica expiración o invalidez del token JWT. El usuario debe cerrar sesión y volver a autenticarse para que Supabase emita un nuevo token.
*   **Código 403 o consultas vacías:** Corresponde a un bloqueo por RLS. Validar en Supabase (Authentication > Policies) que las tablas principales (`conejo`, `alimentacion`, `reproduccion`, `births`, `kits`, `metas_mensuales`) cuenten con la política `authenticated_all` activa, permitiendo la lectura mediante la regla `auth.uid() is not null`. Adicionalmente, asegurar que el frontend no esté aplicando filtros restrictivos por `user_id`.

---

### Incidente 2: Fallo en la creación de usuarios (Edge Function)
> **Contexto:** Un administrador intenta registrar un nuevo usuario a través del módulo de gestión, pero la operación falla y la interfaz notifica "No se pudo crear el usuario".

**Diagnóstico:**
1. Revisar el mensaje emitido por el `AppSnackbar` en la consola de Flutter (el error "User already registered" indica duplicidad).
2. Acceder a Supabase > Edge Functions > `create-user` > Logs para revisar el rastro del servidor.
3. Confirmar que la variable de entorno `SUPABASE_SERVICE_ROLE_KEY` esté correctamente inyectada en los Secrets de la función.

**Plan de Acción:**
*   **Duplicidad de datos:** Verificar el módulo de usuarios y depurar el registro existente, o proceder con un correo electrónico alternativo.
*   **Fallo de permisos:** Validar la vigencia del token del administrador. Si el token expiró, reiniciar la sesión.
*   **Función no desplegada:** En caso de que la función no exista en el entorno, ejecutar el despliegue mediante el comando `supabase functions deploy create-user` en la terminal.

---

### Incidente 3: Fallo en la carga del módulo de IA (Insights)
> **Contexto:** Al invocar el asistente de IA (botón flotante Insight), la interfaz permanece en estado de carga o retorna un error de servidor (401/500).

**Diagnóstico:**
1. Inspeccionar la consola del navegador o los logs de Flutter en busca de excepciones `FunctionsHttpException`.
2. Acceder a Supabase > Edge Functions > `generate-insights` > Logs para evaluar la respuesta del servicio.
3. Si el registro reporta "Google Gemini API error" o "fallback", el origen es la comunicación con el proveedor externo.

**Plan de Acción:**
*   **Error 401:** La sesión del usuario ha expirado; se requiere reautenticación.
*   **Error de modelo:** Verificar el código fuente de `generate-insights` y asegurar que se esté invocando un modelo válido (ej. `gemini-2.0-flash`), descartando referencias a versiones inexistentes.
*   **Error persistente:** Validar en Supabase > Edge Functions > Secrets que la clave `GEMINI_API_KEY` esté configurada y vigente.

---

### Incidente 4: Pantalla en blanco en Flutter Web (CORS/Puertos)
> **Contexto:** La aplicación compila y funciona en dispositivos Android, pero al ejecutarse en el navegador Web, la interfaz no renderiza o presenta fallos de comunicación.

**Diagnóstico:**
1. Abrir las herramientas de desarrollo del navegador (F12) y acceder a la pestaña Console.
2. Verificar errores de política CORS en las peticiones dirigidas a Supabase.
3. Identificar bloqueos en el puerto de ejecución local (ej. error `errno = 10048`).

**Plan de Acción:**
*   **Configuración CORS:** Modificar los headers en las Edge Functions (`create-user`, `generate-insights`) para incluir `'Access-Control-Allow-Origin': '*'` en el código fuente.
*   **Conflicto de puertos:** Ejecutar la aplicación forzando un puerto alternativo mediante el comando `flutter run -d chrome --web-port=8080`.
*   **Persistencia de caché:** Ejecutar un *Hot Restart* desde la terminal de Flutter (tecla `R`) o limpiar la caché del navegador.

---

### Incidente 5: Timeout en la pantalla de inicio de sesión
> **Contexto:** El usuario ingresa credenciales válidas y solicita el acceso, pero el botón de inicio de sesión se mantiene en estado de carga de forma indefinida.

**Diagnóstico:**
1. Inspeccionar la consola de Flutter para identificar excepciones de tipo `TimeoutException` o fallos de red.
2. Verificar en Supabase > Auth > Users que el usuario exista y que su estado de confirmación sea correcto.
3. Confirmar la operatividad del servicio de autenticación.

**Plan de Acción:**
*   **Usuario no registrado:** El administrador debe generar el registro correspondiente desde el módulo de usuarios.
*   **Correo no confirmado:** En Supabase (Authentication > Users), editar el perfil del usuario y forzar la confirmación del correo electrónico.
*   **Caída del servicio:** Verificar el estado global de la plataforma en `status.supabase.com`.

---

### Protocolo de Contingencia y Modo Supervivencia
> **Contexto:** Escenario de degradación crítica donde una o varias Edge Functions (ej. `create-user`, `generate-insights`) presentan fallos generalizados.

**Procedimiento:**
1. **Aislamiento:** No se requiere la suspensión total de la aplicación. Las operaciones transaccionales y de lectura básica (gestión de conejos, historial de alimentación, consultas a PostgreSQL) continuarán operando con normalidad gracias a la conexión directa con la base de datos.
2. **Comunicación al usuario:** La interfaz de Flutter debe interceptar los errores y desplegar un mensaje informativo estandarizado: *"El servicio de Inteligencia Artificial está temporalmente fuera de línea. Las funciones de gestión operativa continúan disponibles"*.
3. **Restablecimiento:** El equipo de desarrollo debe analizar los logs, aplicar las correcciones pertinentes en el entorno local y ejecutar el redespliegue de la función mediante el comando:
   `supabase functions deploy [nombre-de-la-funcion]`
