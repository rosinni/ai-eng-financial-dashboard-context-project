# Verificación del repositorio

## Instrucciones de Ejecución

### Opción recomendada: Docker Compose

Desde la raíz del repositorio:

```bash
docker compose up --build
```

Compose construye los dos servicios definidos en [docker-compose.yml](docker-compose.yml). El frontend queda disponible en `http://localhost:5173`; la API, en `http://localhost:8000`. El backend publica además `5678` para el depurador `debugpy`. Para detener los contenedores, usa `Ctrl+C`; para eliminarlos, ejecuta `docker compose down`.

Comprobaciones rápidas con los servicios activos:

```bash
curl http://localhost:8000/health
curl http://localhost:8000/api/metrics
```

La primera respuesta esperada es `{"status":"ok"}`. La documentación interactiva de FastAPI se sirve en `http://localhost:8000/docs` (OpenAPI JSON: `http://localhost:8000/openapi.json`). Abre `http://localhost:5173` para comprobar la interfaz.

Puertos respaldados por Compose y los comandos de los Dockerfiles:

| Servicio | Puerto del contenedor | Puerto publicado | Uso |
| --- | ---: | ---: | --- |
| Frontend | 5173 | 5173 | Vite; interfaz web |
| Backend | 8000 | 8000 | FastAPI y API HTTP |
| Backend | 5678 | 5678 | `debugpy` |

El proxy de Vite reenvía `/api` a `http://backend:8000`, nombre de servicio que resuelve dentro de la red de Compose. `depends_on` solicita iniciar primero el backend, pero no configura una comprobación de salud ni garantiza que la API ya esté lista para aceptar peticiones.

### Ejecución directa en el host

Se requiere Python con las dependencias de [backend/requirements.txt](backend/requirements.txt), Node.js/npm y las dependencias frontend del lockfile. En una terminal, inicia la API:

```bash
cd backend
python -m pip install -r requirements.txt
python -m debugpy --listen 0.0.0.0:5678 -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

En otra terminal inicia Vite desde el directorio `frontend`. Al ejecutarlo fuera de Docker, define el origen HTTP del backend, porque el proxy configurado usa el nombre DNS de Compose `backend`:

```bash
cd frontend
npm ci
VITE_API_BASE_URL=http://localhost:8000 npm run dev -- --host 0.0.0.0 --port 5173
```

En Linux/macOS, la asignación de variable anterior aplica a ese comando. También puede guardarse `VITE_API_BASE_URL=http://localhost:8000` en `frontend/.env` (el archivo de ejemplo es [frontend/.env.example](frontend/.env.example)).

### Pruebas y validaciones disponibles

Desde la raíz, la suite backend se ejecuta con:

```bash
cd backend && python -m pytest -q
```

Y desde `frontend/` están declarados estos scripts en [frontend/package.json](frontend/package.json):

```bash
npm test
npm run build
npm run lint
```

`npm test` ejecuta Vitest; `npm run build` ejecuta TypeScript (`tsc -b`) y Vite; `npm run lint` ejecuta ESLint. En la inspección de este entorno, `docker compose config` terminó correctamente, pero las suites y comandos frontend no pudieron ejecutarse porque faltaban `pytest`, `vitest`, `tsc` y `eslint` instalados. Por ello, el resultado de esos tests/build/lint queda sin verificar aquí.

## Resumen del Proyecto

El repositorio implementa un panel de métricas financieras con frontend React 19 + TypeScript y backend FastAPI. La descripción general también aparece en [README.es.md](README.es.md) y [README.md](README.md).

En el frontend, [frontend/src/main.tsx](frontend/src/main.tsx) monta [frontend/src/App.tsx](frontend/src/App.tsx). `App` solicita `GET /api/metrics`, administra estados de carga/error, y procesa los movimientos en el cliente mediante `computeKPIs` y `computeMonthlyData`, definidos en [frontend/src/lib/financial-utils.ts](frontend/src/lib/financial-utils.ts). Los tipos compartidos del cliente están en [frontend/src/lib/financial-types.ts](frontend/src/lib/financial-types.ts). La vista compone el encabezado de [frontend/src/components/dashboard/dashboard-header.tsx](frontend/src/components/dashboard/dashboard-header.tsx), los indicadores de [frontend/src/components/dashboard/kpi-row.tsx](frontend/src/components/dashboard/kpi-row.tsx) y [frontend/src/components/dashboard/kpi-card.tsx](frontend/src/components/dashboard/kpi-card.tsx), y dos gráficos de Recharts: [frontend/src/components/dashboard/income-outcome-chart.tsx](frontend/src/components/dashboard/income-outcome-chart.tsx) y [frontend/src/components/dashboard/profit-percent-chart.tsx](frontend/src/components/dashboard/profit-percent-chart.tsx). Los componentes UI compartidos están en `frontend/src/components/ui/`; los estilos y variables de tema, en [frontend/src/index.css](frontend/src/index.css).

La llamada de `App` usa `VITE_API_BASE_URL` si está definida; de lo contrario usa una URL relativa `/api/metrics`. [frontend/vite.config.ts](frontend/vite.config.ts) configura el proxy `/api` hacia `http://backend:8000`, que corresponde al servicio de Compose. [frontend/src/lib/mock-data.ts](frontend/src/lib/mock-data.ts) contiene otra colección de ejemplo, fechada en 2024, pero `App` no la importa: la pantalla carga los datos del backend.

El backend se crea en [backend/app/main.py](backend/app/main.py), donde se configura FastAPI, CORS abierto y el router de [backend/app/routes.py](backend/app/routes.py). Este último genera 360 movimientos financieros sintéticos con semilla `42` por petición; no se encontró una base de datos ni una fuente externa configurada. Incluye `GET /health` y los endpoints `GET /api/metrics`, `/api/metrics/facets`, `/api/metrics/summary`, `/api/metrics/categories/top`, `/api/metrics/comparison`, `/api/metrics/alerts`, `/api/metrics/b2b` y `/api/metrics/b2c`. Hay filtros por fechas, tipo de operación, categoría, grupo temporal y tipo de negocio según endpoint. El dashboard actual solo llama a `/api/metrics`; los demás endpoints existen en el backend, pero no se conectan desde la vista principal inspeccionada.

Las pruebas API están en [backend/tests/test_routes.py](backend/tests/test_routes.py); las pruebas de cálculos del frontend, en [frontend/src/lib/financial-utils.test.ts](frontend/src/lib/financial-utils.test.ts). Los archivos [backend/Dockerfile](backend/Dockerfile) y [frontend/Dockerfile](frontend/Dockerfile) definen imágenes de desarrollo con recarga automática, no describen un despliegue de producción.

## Tabla/Rastro de Verificación

| Estado | Afirmación | Evidencia y alcance |
| --- | --- | --- |
| ✅ **Verificado** | El proyecto tiene un servicio frontend y uno backend ejecutables con Docker Compose. | [docker-compose.yml](docker-compose.yml) define ambos servicios; `docker compose config` se validó correctamente. |
| ✅ **Verificado** | Los puertos publicados son `5173`, `8000` y `5678`. | `docker-compose.yml`, [frontend/Dockerfile](frontend/Dockerfile) y `backend/Dockerfile`; confirmados además con `docker compose config`. |
| ✅ **Verificado** | La interfaz usa React/TypeScript/Vite; la API usa FastAPI/Uvicorn. | [frontend/package.json](frontend/package.json), [frontend/vite.config.ts](frontend/vite.config.ts), [backend/requirements.txt](backend/requirements.txt) y `backend/app/main.py`. |
| ✅ **Verificado** | `GET /health` existe y devuelve `{"status":"ok"}`. | `backend/app/routes.py` y `backend/tests/test_routes.py`. La URL local con Compose es `http://localhost:8000/health`. |
| ✅ **Verificado** | El dashboard carga datos de `/api/metrics` y calcula KPI y series mensuales en el navegador. | `frontend/src/App.tsx` y `frontend/src/lib/financial-utils.ts`. |
| ✅ **Verificado** | La API genera datos sintéticos, no consulta una base de datos declarada en este repositorio. | `backend/app/routes.py` define `generate_mock_movements(seed=42)` y los modelos; no hay servicio o dependencia de base de datos en Compose ni en `backend/requirements.txt`. |
| ✅ **Verificado** | El backend ofrece endpoints de resumen, facetas, categorías, comparación, alertas y subconjuntos B2B/B2C. | Rutas explícitas en `backend/app/routes.py`; las pruebas correspondientes están en `backend/tests/test_routes.py`. Que existan no significa que la interfaz los utilice. |
| ✅ **Verificado** | `frontend/.env.example` existe y deja `VITE_API_BASE_URL` vacío por defecto. | [frontend/.env.example](frontend/.env.example). La presencia se confirmó en el listado del directorio y el contenido se leyó desde el terminal. |
| ❌ **Incorrecto / Corregido** | “El proxy `/api` funciona igual al ejecutar Vite directamente en el host.” | No es una conclusión respaldada por la configuración: [frontend/vite.config.ts](frontend/vite.config.ts) apunta al hostname `backend`, definido como servicio de Compose. Para ejecución directa en el host, se debe establecer `VITE_API_BASE_URL=http://localhost:8000` o configurar un DNS equivalente. |
| ❌ **Incorrecto / Corregido** | “La pantalla actualmente presenta el año 2024 de datos reales.” | El periodo del encabezado de `frontend/src/App.tsx` está fijado en `2024 - Full Year`, pero la API genera datos sintéticos y asigna sus años con la fecha del sistema en `backend/app/routes.py`. El frontend mostrado no aplica filtro de fechas para hacer coincidir ese rótulo. |
| ❌ **Incorrecto / Corregido** | “`frontend/.env.example` no existe.” | Esa sospecha preliminar fue incorrecta: el archivo oculto sí está en el repositorio. La búsqueda inicial por glob no lo mostró, pero el listado y la lectura directa lo confirmaron. |
| ❓ **Sin verificar** | El despliegue en producción, disponibilidad, persistencia y seguridad operativa están resueltos. | Solo se encontraron imágenes de desarrollo, datos en memoria y CORS con `allow_origins=["*"]`; no hay configuración de despliegue productivo, base persistente, autenticación ni verificación de salud de Compose que permita afirmar esos aspectos. |
| ❓ **Sin verificar** | Las pruebas, compilación y lint pasan. | Se identificaron los comandos y suites, pero no se pudieron ejecutar en este entorno: faltan `pytest` y las dependencias frontend (`vitest`, `tsc`, `eslint`). |