# Manual de instalación

## 1. Requisitos
- Docker y Docker Compose (versiones probadas)
- Git, puertos libres: 80 (Nginx), 8081 (Dozzle), 8089 (Locust)

## 2. Clonar el repositorio
git clone <URL> && cd <carpeta>

## 3. Variables de entorno
(.env si hay; qué valor lleva cada una)

## 4. Levantar
docker compose up --build -d

## 5. Verificar
- docker compose ps  -> 4 réplicas "healthy"
- curl http://localhost/health  -> {"status":"ok"}
- Dozzle: http://localhost:8081
- Locust: http://localhost:8089

## 6. Prueba de carga
Pasos para correr Locust y qué resultado esperar.

## 7. Apagar y limpiar
docker compose down -v

## 8. Problemas comunes
| Síntoma | Causa | Solución |
