# 🛠️ Manual de Instalación y Despliegue — CUNITECH

## 1. Descripción del Proyecto
Este documento describe el proceso de instalación y despliegue de **CUNITECH**, un sistema de gestión integral para granjas de conejos. La arquitectura está basada en **Flutter** para el cliente (Web y Android), **Supabase** como Backend-as-a-Service (PostgreSQL + Auth), y **Supabase Edge Functions** para la lógica del servidor y la integración con IA (Gemini).

## 2. Arquitectura del Sistema
- **Frontend:** Flutter (Dart) con GetX para la gestión de estado y navegación.
- **Base de Datos y Auth:** Supabase (PostgreSQL alojado en la nube, con seguridad por filas RLS).
- **Backend (Lógica de Negocio):** Supabase Edge Functions: `login`, `create-user`, `generate-insights`, `identify-breed`, `chat-assistant` y `health`. Ya están desplegadas en la nube.
- **Integración IA:** Google Gemini, consumido desde las Edge Functions (la llave se guarda como secreto de Supabase, no en el código).
- **Roles de la aplicación:** `admin`, `veterinario` y `ayudante`.

> El backend ya está en funcionamiento en la nube. Para ejecutar el sistema solo hay que instalar y correr el cliente Flutter.

## 3. Prerrequisitos de Software
Antes de iniciar, asegúrese de tener instalado:
* **Flutter SDK** con **Dart 3.11.4 o superior** (verifique con `flutter --version`).
* **Git**.
* **Visual Studio Code** o **Android Studio** (para desarrollo).
* Un navegador web moderno (**Google Chrome**) o un dispositivo Android (físico o emulador).
* Conexión a internet (la app se conecta a Supabase en la nube).

*Solo para propietarios del proyecto Supabase (mantenimiento de las funciones, ver sección 6.3):* Docker Desktop y Supabase CLI (v2.0 o superior). **No son necesarios** para ejecutar la app.

## 4. Pasos de Instalación

### Link del repositorio
https://github.com/Sara-AprendizCBA/Cunitech.git

### Paso 1: Clonar el repositorio
```bash
git clone https://github.com/Sara-AprendizCBA/Cunitech.git
cd Cunitech
```

### Paso 2: Instalar dependencias de Flutter
```bash
flutter pub get
```
Debe terminar sin errores (`Got dependencies!`).

### Paso 3: Configuración de Variables de Entorno
**No requiere configuración adicional.** La URL del proyecto y la llave pública (anon key) de Supabase ya están en `lib/main.dart`, dentro de `Supabase.initialize(...)`. El archivo `.env.example` está vacío a propósito.

Las llaves secretas (Gemini, Resend, service_role) no están en el repositorio: viven como secretos dentro de Supabase.

*(Nota: En producción, estas credenciales deben gestionarse mediante variables de entorno o archivos de configuración seguros).*

### Paso 4: Cuenta de prueba
Use esta cuenta para iniciar sesión (rol administrador):

| Campo | Valor |
|---|---|
| Correo | `<CORREO-DE-PRUEBA>` |
| Contraseña | `<CONTRASEÑA-DE-PRUEBA>` |

### Paso 5: Ejecutar la aplicación en entorno de desarrollo
Para ejecutar en el navegador web:
```bash
flutter run -d chrome --web-port=60144
```
> **Importante:** use exactamente el puerto **60144**. Las funciones de IA y de creación de usuarios solo aceptan peticiones desde `http://localhost:60144` (política CORS). La primera compilación puede tardar varios minutos.

Para ejecutar en un dispositivo Android conectado por USB:
```bash
flutter devices
flutter run -d <ID_DEL_DISPOSITIVO>
```

## 5. Verificación del Despliegue (Smoke Tests)

**1. Verificar que la app carga correctamente:**
Abra la app en el navegador o en el dispositivo e inicie sesión con la cuenta de prueba del Paso 4. Debe abrir el panel principal.

**2. Verificar conexión con Supabase:**
En la consola de Flutter, debe aparecer el mensaje:
```
supabase.supabase_flutter: INFO: ***** Supabase init completed *****
```

**3. Verificar el estado del backend:**
Desde otra terminal (en Windows use `curl.exe`):
```bash
curl -i https://aswhtehpmmeetplnskmz.supabase.co/functions/v1/health
```
Debe responder `200` con un JSON que contiene `"status":"ok"` y `"database":"connected"`.

**4. Verificar las funciones de la app:**
En el panel verifique que cargan el Dashboard y la lista de *Conejos*. Cree un conejo de prueba, edítelo y elimínelo. Luego pruebe el asistente de chat (IA): envíe un mensaje y debe responder.

## 6. Despliegue en Producción

### 6.1. Despliegue del Frontend (Flutter Web)
Para compilar la versión web optimizada para producción:
```bash
flutter build web --release
```
El resultado se genera en la carpeta `build/web/`. Este contenido puede desplegarse en servicios como Firebase Hosting, Netlify, Vercel o un servidor propio.

> Si se publica en un dominio propio, el propietario del proyecto Supabase debe actualizar el secreto `APP_URL` con ese dominio y volver a desplegar las funciones; de lo contrario las funciones de IA y de creación de usuarios rechazarán las peticiones (CORS).

### 6.2. Despliegue del Frontend (Android)
Para compilar el APK de Android:
```bash
flutter build apk --release
```
El archivo APK se genera en `build/app/outputs/flutter-apk/app-release.apk` y puede distribuirse directamente o publicarse en Google Play.

### 6.3. Despliegue del Backend (Edge Functions) — solo propietarios del proyecto Supabase
Las funciones ya están desplegadas. Esta sección solo aplica a quien tenga acceso al proyecto de Supabase y necesite actualizarlas. Asegúrese de estar autenticado y vinculado al proyecto:
```bash
supabase login
supabase link --project-ref aswhtehpmmeetplnskmz
supabase functions deploy login
supabase functions deploy create-user
supabase functions deploy generate-insights
supabase functions deploy identify-breed
supabase functions deploy chat-assistant
supabase functions deploy health
```
Secretos requeridos: `GEMINI_API_KEY`, `RESEND_API_KEY` y `APP_URL` (`supabase secrets list` para verlos).

### 6.4. Base de Datos (Migraciones)
La base de datos de la nube ya está aprovisionada. Si se realizan cambios, las migraciones se aplican manualmente mediante el **SQL Editor** de Supabase, con los scripts de `supabase/migrations/` en orden (002 a 007 y 014 a 018).

*(Nota: las migraciones 001 y 008 a 013 no están en el repositorio; por eso la base de datos no puede reconstruirse desde cero solo con estos scripts.)*

## 7. Comandos de Operación Diaria
* **Ejecutar en desarrollo (Web):** `flutter run -d chrome --web-port=60144`
* **Actualizar dependencias:** `flutter pub get`
* **Reinicio completo:** `flutter clean` y luego `flutter pub get`
* **Analizar el código en busca de errores:** `flutter analyze`
* **Compilar para producción (Web):** `flutter build web --release`
* **Compilar para producción (Android):** `flutter build apk --release`
* **Estado del backend:** `curl -i https://aswhtehpmmeetplnskmz.supabase.co/functions/v1/health`
* **Ver logs de las Edge Functions en tiempo real (propietarios):**
  * Creación de usuarios: `supabase functions logs create-user`
  * Generación de insights (IA): `supabase functions logs generate-insights`
  * Identificación de raza (IA): `supabase functions logs identify-breed`
  * Asistente de chat (IA): `supabase functions logs chat-assistant`
  * Inicio de sesión: `supabase functions logs login`
* **Desplegar Edge Functions específicas (propietarios):**
  * Creación de usuarios: `supabase functions deploy create-user`
  * Generación de insights: `supabase functions deploy generate-insights`
  * Identificación de raza: `supabase functions deploy identify-breed`
  * Asistente de chat: `supabase functions deploy chat-assistant`
  * Inicio de sesión: `supabase functions deploy login`
