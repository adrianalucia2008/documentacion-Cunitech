# Reporte Post-Mortem del Incidente

**Equipo Afectado (Nosotros):** Cunitech
**Equipo Saboteador:** Smart Compost
**Tiempo Total de Resolución:** 30 min


## 1. Detección y Síntomas

¿Cómo se dieron cuenta de que el sistema estaba fallando?

El contenedor de Nginx no iniciaba correctamente. Al revisar los logs se identificó un error de configuración en la línea 25 del archivo `nginx.conf`:

```text
invalid value "http_505"
```

El valor `http_505` no es válido para la directiva `proxy_next_upstream`. Por este motivo, Nginx abortaba antes de levantar el servicio y las peticiones a `http://localhost/api/health` no podían resolverse correctamente.

**Síntoma en Locust:**  
No se utilizó Locust en esta validación. El síntoma equivalente se comprobó con `curl`: mientras Nginx no iniciaba, el endpoint `/api/health` no respondía correctamente.

**Síntoma en Dozzle / Terminal:**  
Nginx reportaba configuración inválida por `http_505`. Luego de corregir la configuración, la validación con `curl.exe http://localhost/api/health` devolvió:

```http
HTTP/1.1 200 OK
Server: nginx/1.31.3
Content-Type: application/json
```

Con cuerpo:

```json
{"status":"ok","timestamp":"2026-11-05T20:45:53.540094Z","uptime_seconds":2008.59,"database":{"status":"up","error":null}}
```

Posteriormente se ejecutaron 12 peticiones consecutivas al endpoint:

```powershell
1..12 | ForEach-Object { curl.exe -s http://localhost/api/health; "" }
```

Todas respondieron con `"status":"ok"` y `"database":{"status":"up"}`. Además, se observaron valores distintos de `uptime_seconds` —por ejemplo `1761.89` y `2168.77`—, lo que evidencia que Nginx estaba distribuyendo carga entre varias réplicas del backend.

---

## 2. Diagnóstico y Causa Raíz

¿Qué descubrieron al investigar? ¿Qué fue exactamente lo que el otro equipo dañó u ocultó en el código/infraestructura?

### Causa raíz técnica

En la línea 25 de `nginx.conf` se encontraba la siguiente directiva:

```nginx
proxy_next_upstream error timeout http_505 http_567;
```

Los valores `http_505` y `http_567` no son válidos para `proxy_next_upstream`. Nginx solo acepta condiciones y códigos HTTP soportados, por ejemplo:

```nginx
proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
```

Además, `http_567` no es un código HTTP estándar. Esto provocaba que Nginx fallara al iniciar con el error:

```text
invalid value "http_505"
```

También se detectó que los headers del proxy estaban definidos sin las variables de Nginx:

```nginx
proxy_set_header Host host;
proxy_set_header X-Real-IP remote_addr;
proxy_set_header X-Forwarded-For proxy_add_x_forwarded_for;
```

Lo correcto es:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

### Causa raíz documental

El `RUNBOOK_INCIDENTES.md` entregado no corresponde a la arquitectura desplegada de SmartCompost. El documento describe una aplicación CUNITECH basada en Flutter, Supabase y PostgreSQL, mientras que el despliegue real usa:

- Docker Compose.
- Nginx como balanceador.
- 4 réplicas del backend.
- MySQL.
- Endpoint de salud `http://localhost/api/health`.

El Runbook no contempla procedimientos para diagnosticar errores de configuración de Nginx, revisar contenedores Docker Compose ni recuperar el balanceador de carga.

**Evidencia comparativa:**

| Nuestro despliegue SmartCompost | Runbook entregado |
|---|---|
| Docker Compose | Ejecución local de Flutter |
| Nginx | No documenta Nginx |
| 4 réplicas de backend | No documenta réplicas |
| MySQL | Supabase / PostgreSQL |
| `localhost:80` | `localhost:60144` |
| `/api/health` | Supabase Edge Function `/health` |

---

## 3. Resolución y Evaluación del Runbook

¿Cómo lo solucionaron? ¿El Runbook del otro equipo tenía la respuesta o tuvieron que improvisar?

### Pasos de Solución

1. Se conservaron las evidencias del error antes de modificar el proyecto.
2. Se revisó la línea 25 de `nginx.conf` y se identificó el valor inválido `http_505`.
3. Se corrigió `proxy_next_upstream` para usar únicamente valores válidos:

   ```nginx
   proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
   ```

4. Se corrigieron los headers del proxy para usar las variables de Nginx:

   ```nginx
   proxy_set_header Host $host;
   proxy_set_header X-Real-IP $remote_addr;
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   ```

5. Se validó la sintaxis de Nginx:

   ```bash
   docker compose exec nginx nginx -t
   ```

6. Se reinició / recargó Nginx:

   ```bash
   docker compose restart nginx
   ```

7. Se verificó el endpoint de salud:

   ```bash
   curl.exe http://localhost/api/health
   ```

8. Se ejecutaron 12 peticiones consecutivas para validar estabilidad y distribución entre réplicas:

   ```powershell
   1..12 | ForEach-Object { curl.exe -s http://localhost/api/health; "" }
   ```

**Resultado:**  
El servicio quedó operativo. Todas las peticiones respondieron con HTTP 200, `"status":"ok"` y `"database":{"status":"up"}`. Los distintos valores de `uptime_seconds` confirman que las réplicas estaban siendo utilizadas.

### Utilidad del Runbook

🔴 **Inútil.** El error no estaba contemplado. El Runbook entregado no contenía procedimientos para diagnosticar fallos de Nginx ni de Docker Compose. Fue necesario improvisar el diagnóstico a partir de los logs del contenedor y de la revisión directa de `nginx.conf`.

---

## 4. Acciones de Mejora Preventiva

Si ustedes fueran los dueños de este sistema, ¿qué le agregarían al `RUNBOOK_INCIDENTES.md` basándose en este ataque?

- **Agregar un incidente específico para Nginx:**  
  **INC-NGINX-01 — Nginx no inicia por valor inválido en `proxy_next_upstream`.**  
  Síntoma esperado:  
  ```text
  invalid value "http_505"
  ```  
  Pasos de diagnóstico:
  ```bash
  docker compose logs nginx
  docker compose exec nginx nginx -t
  sed -n '20,30p' nginx.conf
  ```  
  Corrección esperada: usar solo códigos válidos, por ejemplo:
  ```nginx
  proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
  ```

- **Agregar un incidente para contenedores que no levantan:**  
  **INC-DOCKER-01 — Servicio no inicia en Docker Compose.**  
  Incluir comandos:
  ```bash
  docker compose ps
  docker compose logs nginx
  docker compose restart nginx
  ```

- **Agregar validación posterior a cualquier cambio en Nginx:**  
  Incluir como paso obligatorio:
  ```bash
  docker compose exec nginx nginx -t
  curl.exe http://localhost/api/health
  ```

- **Agregar una sección de arquitectura real de SmartCompost:**  
  Documentar servicios, puertos, réplicas, balanceador y endpoint de salud:
  - Nginx en `localhost:80`.
  - 4 réplicas del backend.
  - MySQL.
  - Endpoint `/api/health`.
  - Respuesta esperada: `{"status":"ok","database":{"status":"up"}}`.

- **Agregar verificación de balanceo entre réplicas:**  
  Incluir el comando:
  ```powershell
  1..12 | ForEach-Object { curl.exe -s http://localhost/api/health; "" }
  ```
 


# Capturas

![Captura 1](capturas/Captura%201.jpeg)


![Captura 2](capturas/Captura%202.jpeg)


![Captura 3](capturas/Captura%203.jpeg)


![Captura 4](capturas/Captura%204.jpeg)


Sí, el fallo quedó arreglado. Lo confirman tres cosas:

nginx -t dice que la sintaxis es correcta.
/api/health responde 200 a través del balanceador, con la base de datos arriba.
Las 12 peticiones se repartieron entre varias réplicas, y en docker compose ps los backends estaban healthy
    
