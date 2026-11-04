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

## 5. Despliegue en Producción

### 5.1. Despliegue del Frontend (Flutter Web)
Para compilar la versión web optimizada para producción:
```bash
flutter build web --release
```
El resultado se genera en la carpeta `build/web/`. Este contenido puede desplegarse en servicios como Firebase Hosting, Netlify, Vercel o un servidor propio.


### 5.2. Despliegue del Frontend (Android)

Para compilar el APK de Android (versión de producción optimizada):
```bash
flutter build apk --release
```
El archivo APK se generará en la ruta `build/app/outputs/flutter-apk/app-release.apk`. Este archivo puede distribuirse directamente (por ejemplo, por WhatsApp o correo) o publicarse en Google Play.

**Instalación en un dispositivo Android (Modo Desarrollador):**
Si va a instalar el APK directamente en un celular (sin usar Google Play), el dispositivo debe tener habilitados los permisos de desarrollador. Siga estos pasos:

1. Ir a **Ajustes** > **Información del teléfono** (o "Acerca del teléfono").
2. Tocar 7 veces seguidas la opción **"Número de compilación"** hasta que aparezca el mensaje *"Ahora eres desarrollador"*.
3. Volver al menú principal de **Ajustes** > **Sistema** > **Opciones de desarrollador** (o "Ajustes adicionales" en algunas marcas).
4. Activar la opción **"Depuración USB"** (necesaria si va a instalar por cable desde el PC).
5. Activar la opción **"Instalar vía USB"** (en algunas marcas de celular aparece como "Instalación por USB" o "Permitir instalación desde fuentes desconocidas"). Esta opción es la que permite instalar el APK generado por Flutter.

Una vez habilitados estos permisos, puede conectar el celular por USB y ejecutar:
```bash
flutter install
```
*(Nota: Si el celular bloquea la instalación, desconecte y vuelva a conectar el cable USB, y acepte el mensaje de "Permitir depuración USB" que aparecerá en la pantalla del teléfono).*

### 5.3. Despliegue del Backend (Edge Functions)
Las funciones del servidor (IA y creación de usuarios) se despliegan a Supabase mediante la CLI. Para ello, siga estos pasos:

1. **Autenticación en Supabase:**
   Ejecute el siguiente comando para iniciar sesión en su cuenta de Supabase. Este paso es obligatorio la primera vez que se configura el proyecto en un equipo, ya que la CLI necesita un token de acceso para autorizar los despliegues.
   ```bash
   supabase login
   ```
   *(Nota: Se abrirá una ventana en el navegador para que ingrese sus credenciales. Si ya ha iniciado sesión previamente en este equipo, puede omitir este paso).*

2. **Vincular el proyecto local con el remoto:**
   Una vez autenticado, vincule la carpeta local con su proyecto en la nube. Reemplace con el ID de referencia del proyecto 

   ```bash
   upabase link --project-ref aswhtehpmmeetplnskmz
   ```

3. **Desplegar las funciones:**
   Finalmente, suba cada una de las Edge Functions a producción:
   ```bash
   supabase functions deploy create-user
   supabase functions deploy generate-insights
   supabase functions deploy identify-breed
   supabase functions deploy chat-assistant
   ```

## 6. Verificación del Despliegue (Smoke Tests)

**1. Verificar que la app carga correctamente:**
Abra el navegador o la app en el dispositivo, e inicie sesión con un usuario válido (ej. admin).

**2. Verificar conexión con Supabase:**
En la consola de Flutter, debe aparecer el mensaje:
```
supabase.supabase_flutter: INFO: ***** Supabase init completed *****
```

**3. Verificar Edge Functions:**
Realice una acción que invoque una función (ej. crear un usuario desde el panel de admin o generar un insight desde el dashboard). En los logs de Supabase → Edge Functions, debe registrarse la invocación exitosa.

### Paso 7: Ejecutar la aplicación en entorno de desarrollo
Para ejecutar en el navegador web:
```bash
flutter run -d chrome
```

Para ejecutar en un dispositivo Android conectado por USB:
```bash
flutter run -d <ID_DEL_DISPOSITIVO>
```


## 8. Comandos de Operación Diaria
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

  
