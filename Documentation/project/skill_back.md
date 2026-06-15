---
name: knowyourmarks
description: >
  Skill de contexto para el proyecto KnowYourMarks — aplicación Django + PostgreSQL + Docker
  para benchmarking y comparación de hardware de videojuegos, desarrollada por el equipo
  Centro-Democratico en el curso Ingeniería de Software 1 (UNAL).
  Úsalo SIEMPRE que el usuario mencione KnowYourMarks, project-backend, Centro-Democratico,
  cualquiera de los modelos (BenchmarkSession, HardwareComponent, SessionSample, Hardware,
  Videogame, Result, GameResult, User), cualquier RF (RF_1 a RF_16), cualquier compañero
  (dvargaspa, dangomezma, lcastroam, sanherrerapa, nfontaine), o cualquier tarea de vistas
  Django, tests con mocks, migraciones, o Docker/PostgreSQL en este proyecto.
  También activar cuando el usuario diga "mi proyecto", "el backend", "nuestro proyecto"
  si hay indicios de que se refiere a KnowYourMarks.
---

# KnowYourMarks — Skill de Contexto del Proyecto

## Descripción

KnowYourMarks es una aplicación web de benchmarking de hardware para videojuegos. Permite
monitorear FPS en tiempo real, comparar hardware, generar informes técnicos y mantener un
ranking global de componentes. Desarrollada por 5 estudiantes de la UNAL como proyecto del
curso Ingeniería de Software 1 (2016701).

- **Backend:** `https://github.com/Centro-Democratico/project-backend.git`
- **Docs:** `https://github.com/Centro-Democratico/project-docs.git`
- **App Django principal:** `core/` (modelos, vistas, tests, servicios)
- **Config Django:** `benchmark_backend/` (settings, urls, wsgi)
- **BD local (dev):** PostgreSQL 16 vía Docker Compose

---

## Stack técnico

| Capa | Tecnología |
|---|---|
| Framework | Django 5.2 |
| Base de datos | PostgreSQL 16 (Docker) |
| ORM | Django ORM (sin DRF en vistas principales; `rest_framework` instalado pero no usado aún) |
| Primary keys | `UUIDField(primary_key=True, default=uuid.uuid4, editable=False)` en todos los modelos |
| Auth (provisional) | Header HTTP `X-User-Admin: true` |
| Testing | `django.test.TestCase` + `SimpleTestCase` + `unittest.mock` |
| Dependencias | `Django`, `psycopg2-binary`, `python-dotenv`, `djangorestframework`, `sqlparse` |

---

## Todos los Requisitos Funcionales (RF) y sus responsables

| RF | Título | Responsable | Estado en backend |
|---|---|---|---|
| RF_1 | Overlay de FPS en tiempo real + resumen | dangomezma | `services.py` (lógica pura) |
| RF_2 | Activar/desactivar overlay con hotkey | dangomezma | `services.py` (lógica pura) |
| RF_3 | Lista de hardware recomendado con enlaces | lcastroam | Pendiente |
| RF_4 | Alertas de cuello de botella y fallos | lcastroam | Pendiente |
| RF_5 | Registrar capacidad de fuente de poder (PSU) | lcastroam | Pendiente |
| RF_6 | Comparar resultados con dispositivos estándar | dangomezma | `services.py` (lógica pura) |
| RF_7 | Buscar componente de hardware | sanherrerapa | Pendiente |
| RF_8 | Vista detallada de componente (Component View) | nfontaine | Pendiente |
| RF_9 | Almacenar telemetría anónima para ranking global | dvargaspa | ✅ `POST /telemetry/` |
| RF_10 | Exportar catálogo de componentes a archivo local | sanherrerapa | Pendiente |
| RF_11 | CRUD de componentes del ranking (admin) | dvargaspa | ✅ `GET/POST /components/`, `GET/PUT/DELETE /components/<uuid>/` |
| RF_12 | Guardar resultados de benchmark en archivo local | sanherrerapa | Pendiente |
| RF_13 | Informe técnico detallado de sesión | dvargaspa | ✅ `GET /sessions/`, `GET /sessions/<uuid>/report/` |
| RF_15 | Recomendar hardware por juego | nfontaine | Pendiente |
| RF_16 | Comparar hardware del usuario vs requisitos de juego | nfontaine | Pendiente |

---

## Todos los modelos (`core/models.py`)

Todos los modelos tienen `managed = True` y UUID como PK. Las relaciones usan `ForeignKey`
con `on_delete=models.CASCADE`.

### Modelos existentes (migración 0001 + 0002)

| Modelo | Tabla | Campos principales | Relaciones |
|---|---|---|---|
| `User` | `user` | `username`, `email`, `password_hash`, `is_admin`, `created_at` | — |
| `Hardware` | `hardware` | `name`, `cpu`, `gpu`, `gb_ram`, `storage_type`, `created_at` | FK → `User` |
| `Videogame` | `videogame` | `name`, `genre`, `developer`, `release_year` | — |
| `VideogameRequirement` | `videogame_requirement` | `cpu`, `gpu`, `gb_ram`, `target_fps`, `resolution`, `settings` | FK → `Videogame` |
| `Result` | `result` | `benchmark_type`, `score`, `fps_avg/min/max`, `resolution`, `settings`, `created_at` | FK → `Hardware` |
| `GameResult` | `game_result` | `fps_min/max/avg`, `resolution`, `settings`, `created_at` | FK → `Hardware`, FK → `Videogame` |
| `HardwareComponent` | `hardware_component` | `brand`, `model`, `type` (`'CPU'`/`'GPU'`), `base_specs`, `launch_year`, `base_score`, `created_at`, `updated_at` | — |
| `BenchmarkSession` | `benchmark_session` | `started_at`, `ended_at`, `cpu_avg/max`, `cpu_temp_avg/max`, `gpu_avg/max`, `gpu_temp_avg/max`, `ram_avg_gb/max_gb`, `disk_read_avg`, `disk_write_avg`, `score`, `is_anonymous`, `created_at` | FK opcional → `Hardware` |
| `SessionSample` | `session_sample` | `timestamp_seconds`, `cpu_pct`, `gpu_pct`, `ram_gb`, `cpu_temp`, `gpu_temp`, `disk_read`, `disk_write` | FK → `BenchmarkSession` (`related_name='samples'`), ordering por `timestamp_seconds` |

---

## Endpoints implementados (`benchmark_backend/urls.py`)

```
GET  POST        /videogames/
GET  PUT DELETE  /videogames/<uuid:pk>/
GET  POST        /components/           (POST requiere admin)
GET  PUT DELETE  /components/<uuid:pk>/ (PUT, DELETE requieren admin)
POST             /telemetry/
GET              /sessions/
GET              /sessions/<uuid:pk>/report/
```

---

## Convenciones de vistas (`core/views.py`)

Para instrucciones detalladas con código, ver `references/add-views.md`.

**Resumen del patrón:**
- Cada endpoint se divide en funciones privadas por método: `_entidad_list_get`, `_entidad_list_create`, etc.
- Un dispatcher `@csrf_exempt @require_http_methods([...])` llama al handler según `request.method`
- Serializer `_entidad_to_dict(obj)` convierte el modelo a dict; los UUID siempre con `str(obj.id)`
- Guard de admin provisional: header `X-User-Admin: true` → función `_require_admin(request)`
- Respuestas: lista → `JsonResponse(lista, safe=False)`, creación → `status=201`, delete → `{'mensaje': '...'}`, error → `status=400`, forbidden → `status=403`
- URLs siempre con `<uuid:pk>`, nunca `<int:pk>`

---

## Convenciones de tests (`core/tests.py`)

Para instrucciones detalladas con código, ver `references/add-tests.md`.

**Resumen del patrón:**
- `TestCase` para tests con BD; `SimpleTestCase` para lógica pura (sin BD)
- Cada clase de test tiene `setUp` con `self.client`, `self.admin_header = {'HTTP_X_USER_ADMIN': 'true'}` y datos de prueba
- Objetos falsos se crean con `MagicMock(spec=ModelClass)` en un método `_MockEntidad()`
- Mocks de ORM: `@patch('core.views.ModelClass.objects.create')`, `@patch('core.views.get_object_or_404')`
- Por endpoint: mínimo 3 tests — happy path, error path, edge case
- Cabecera de admin: `HTTP_X_USER_ADMIN` (Django convierte guiones a guión bajo, agrega `HTTP_`)

---

## Lógica pura (`core/services.py`)

Funciones sin queries a BD, probadas con `SimpleTestCase`:

| Función | RF | Descripción |
|---|---|---|
| `calculate_session_metrics(fps_list)` | RF_1 | Calcula max, min, avg de una lista de FPS |
| `toggle_overlay_state(current_state: bool)` | RF_2 | Invierte el estado del overlay; lanza `ValueError` si no es bool |
| `calculate_performance_gap(user_fps, market_fps)` | RF_6 | Porcentaje de diferencia vs mercado; retorna 0.0 si `market_fps <= 0` |

Nueva lógica de negocio pura → va en `services.py`, no en `views.py`.

---

## Docker y PostgreSQL

Para soluciones detalladas de problemas frecuentes, ver `references/docker-troubleshooting.md`.

**Levantar la BD:**
```bash
cd scripts && docker-compose --env-file ../.env up -d && cd ..
```

**Variables requeridas en `.env`** (solo ASCII en contraseñas):
```
DB_NAME=...   DB_USER=...   DB_PASSWORD=...   DB_HOST=localhost   DB_PORT=5432
```

**Problemas más comunes:** WSL reset, conflictos de migraciones, `managed=False`, caracteres
no-ASCII en `.env`, PostgreSQL no listo al arrancar. Ver `references/docker-troubleshooting.md`.

---

## Referencias disponibles

- **`references/add-model.md`** — checklist y plantilla para agregar un nuevo modelo Django
- **`references/add-views.md`** — patrón CRUD completo con serializer, handlers y dispatcher
- **`references/add-tests.md`** — todos los patrones de mock del proyecto con ejemplos
- **`references/docker-troubleshooting.md`** — soluciones para problemas frecuentes de Docker/PostgreSQL/WSL
