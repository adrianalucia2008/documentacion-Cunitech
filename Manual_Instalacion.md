# 🛠️ Manual de Instalación y Despliegue — CUNITECH

## 1. Descripción del Proyecto
Este documento describe el proceso de instalación y despliegue de **CUNITECH**, un sistema de gestión integral para granjas de conejos. La arquitectura está basada en **Flutter** para el cliente (Web y Android), **Supabase** como Backend-as-a-Service (PostgreSQL + Auth), y **Supabase Edge Functions** para la lógica del servidor y la integración con IA (Gemini).

## 2. Arquitectura del Sistema
- **Frontend**: Flutter **(Dart) con GetX para la gestión de estado y navegación.
- **Base de Datos y Auth**: Supabase (PostgreSQL alojado en la nube).
- **Backend API (Lógica de Negocio**): Python + FastAPI (Puerto interno 8000). Empaquetado en Docker.
- **Proxy Inverso**: Nginx (Puerto 80) para redirigir tráfico y asegurar la API.
- **Observabilidad**: Dozzle (Puerto 8080) para monitorear los logs de los contenedores en tiempo real.
- **Pruebas de Carga**: Locust (Puerto 8089) para validar el rendimiento de la API.
- **Integración IA (Opcional)** : Supabase Edge Functions o conexión directa desde la API de Python a Gemin


## 3. Prerrequisitos de Software
Antes de iniciar, asegúrese de tener instalado:
* **Flutter SDK** (versión estable más reciente).
* **Dart** (incluido con Flutter).
* **Git**.
* **Visual Studio Code** o **Android Studio** (para desarrollo).
* **Docker Desktop** (necesario únicamente para desplegar las Edge Functions con Supabase CLI).
* **Supabase CLI** (v2.0 o superior).
* Un navegador web moderno (Chrome, Edge, etc.) o un dispositivo Android (físico o emulador).

## 4. Pasos de Instalación

### Link del repositorio

```bash
https://github.com/Sara-AprendizCBA/Cunitech.git
```

### Paso 1: Clonar el repositorio
```bash
git clone https://github.com/Sara-AprendizCBA/Cunitech.git
cd cunitech
```

### Paso 2: Instalar dependencias de Flutter
```bash
flutter pub get
```

### Paso 3: Configuración de Variables de Entorno
CUNITECH se conecta a Supabase mediante credenciales. Asegúrese de tener configurado el archivo `lib/core/config/supabase_config.dart` (o equivalente) con la URL del proyecto y la `anon_key`.

*(Nota: En producción, estas credenciales deben gestionarse mediante variables de entorno o archivos de configuración seguros).*

## 4. Despliegue en Producción

### 4.1. Despliegue del Frontend (Flutter Web)
Para compilar la versión web optimizada para producción:
```bash
flutter build web --release
```
El resultado se genera en la carpeta `build/web/`. Este contenido puede desplegarse en servicios como Firebase Hosting, Netlify, Vercel o un servidor propio.

### 4.2. Despliegue del Frontend (Android)
Para compilar el APK de Android:
```bash
flutter build apk --release
```
El archivo APK se genera en `build/app/outputs/flutter-apk/app-release.apk` y puede distribuirse directamente o publicarse en Google Play.

### 4.3. Despliegue del Backend (Edge Functions)
Las funciones del servidor (IA y creación de usuarios) se despliegan a Supabase mediante CLI. Asegúrese de estar autenticado y vinculado al proyecto:
```bash
supabase login
supabase link --project-ref [PROJECT_REF]
supabase functions deploy create-user
supabase functions deploy generate-insights
supabase functions deploy identify-breed
supabase functions deploy chat-assistant
```

### 4.4. Base de Datos (Migraciones)
Si se realizan cambios en la base de datos, aplique las migraciones correspondientes:
```bash
supabase db push
```
*(Nota: En este proyecto, las migraciones se aplican manualmente mediante SQL Editor cuando sea necesario).*

## 5. Verificación del Despliegue (Smoke Tests)

**1. Verificar que la app carga correctamente:**
Abra el navegador o la app en el dispositivo, e inicie sesión con un usuario válido (ej. admin).

**2. Verificar conexión con Supabase:**
En la consola de Flutter, debe aparecer el mensaje:
```
supabase.supabase_flutter: INFO: ***** Supabase init completed *****
```

**3. Verificar Edge Functions:**
Realice una acción que invoque una función (ej. crear un usuario desde el panel de admin o generar un insight desde el dashboard). En los logs de Supabase → Edge Functions, debe registrarse la invocación exitosa.

### Paso 6: Ejecutar la aplicación en entorno de desarrollo
Para ejecutar en el navegador web:
```bash
flutter run -d chrome
```

Para ejecutar en un dispositivo Android conectado por USB:
```bash
flutter run -d <ID_DEL_DISPOSITIVO>
```


## 7. Comandos de Operación Diaria
* **Ejecutar en desarrollo (Web):** `flutter run -d chrome`
* **Actualizar dependencias:** `flutter pub get`
* **Analizar el código en busca de errores:** `flutter analyze`
* **Compilar para producción (Web):** `flutter build web --release`
* **Compilar para producción (Android):** `flutter build apk --release`
* **Ver logs de las Edge Functions en tiempo real:**
  * Creación de usuarios: `supabase functions logs create-user` 
  * Generación de insights (IA): `supabase functions logs generate-insights`
  * Identificación de raza (IA): `supabase functions logs identify-breed`
  * Asistente de chat (IA): `supabase functions logs chat-assistant`
  * Inicio de sesión: `supabase functions logs login`
* **Desplegar Edge Functions específicas:**
  * Creación de usuarios: `supabase functions deploy create-user`
  * Generación de insights: `supabase functions deploy generate-insights`
  * Identificación de raza: `supabase functions deploy identify-breed`
  * Asistente de chat: `supabase functions deploy chat-assistant`
  * Inicio de sesión: `supabase functions deploy login`

  
