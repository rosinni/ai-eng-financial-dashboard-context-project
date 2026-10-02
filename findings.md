# Hallazgos de arquitectura y código

Revisión estática del frontend, backend, configuración y pruebas existentes. Los hallazgos describen el estado del repositorio, no presuponen requisitos de producción que no estén documentados.

## Arquitectura

### A1. El módulo de rutas también contiene el dominio y la generación de datos

**Riesgo medio.** [backend/app/routes.py](backend/app/routes.py) define modelos de respuesta, crea los movimientos ficticios, filtra y agrega datos, calcula alertas y comparaciones, y declara todos los handlers HTTP. Esto mantiene pequeño el proyecto actual, pero mezcla límites que cambiarían por motivos distintos al sustituir los datos de muestra o ampliar los endpoints.

**Regla accionable:** al añadir persistencia o una nueva fuente de datos, mantener los handlers como adaptadores HTTP y mover generación/acceso a datos y cálculos de dominio a módulos propios; no duplicar esas reglas dentro de cada handler.

### A2. La semilla modifica el generador aleatorio global

**Riesgo medio.** `generate_mock_movements(seed=42)` llama a `random.seed(seed)` en [backend/app/routes.py](backend/app/routes.py). La función cambia estado compartido del proceso; solicitudes concurrentes pueden intercalar el consumo del generador y dejar de producir resultados independientes y repetibles.

**Regla accionable:** para generación reproducible, instanciar `random.Random(seed)` localmente y pasar ese generador a las funciones que lo usan; evitar modificar el estado aleatorio global desde código de petición.

### A3. El período mostrado no controla el período consultado

**Riesgo alto para la interpretación del panel.** [frontend/src/App.tsx](frontend/src/App.tsx) fija `2024 - Full Year` en el encabezado y solicita `/api/metrics` sin filtros de fecha. En [backend/app/routes.py](backend/app/routes.py), la generación asigna años en función de `date.today()`. Por tanto, el rótulo puede no describir las fechas que alimentan los indicadores y gráficos.

**Regla accionable:** el período visible debe derivarse del filtro aplicado y enviarse en la consulta a la API; no mostrar un intervalo fijo si los datos no se limitan a él.

### A4. CORS está abierto para cualquier origen

**Riesgo de seguridad/configuración.** [backend/app/main.py](backend/app/main.py) configura `allow_origins=["*"]` junto con `allow_credentials=True`, sin distinguir desarrollo de otros entornos.

**Regla accionable:** definir orígenes permitidos por entorno y enumerar los dominios de la aplicación; habilitar credenciales solo cuando el flujo de autenticación las requiera.

## Naming

### N1. `create_date` representa una fecha sin hora

**Riesgo bajo de ambigüedad en el contrato.** [backend/app/routes.py](backend/app/routes.py) tipa `FinancialMovement.create_date` como `date`, no como fecha-hora; el campo equivalente está expuesto en [frontend/src/lib/financial-types.ts](frontend/src/lib/financial-types.ts) como `string` ISO. `create_date` puede interpretarse como el instante en que se creó el registro, aunque el modelo solo tiene fecha de movimiento.

**Regla accionable:** usar un término de dominio explícito y consistente, por ejemplo `transaction_date` si solo importa el día; reservar `created_at` para un timestamp. Si se renombra el campo expuesto, actualizar backend, frontend, tests y contrato de API en el mismo cambio.

### N2. La interfaz mezcla idiomas y convenciones regionales

**Riesgo bajo de coherencia de producto.** Los textos del dashboard y formatos de [frontend/src/lib/financial-utils.ts](frontend/src/lib/financial-utils.ts) usan inglés/en-US y USD; el error de carga de [frontend/src/App.tsx](frontend/src/App.tsx) está en español. El README en español describe la misma aplicación en [README.es.md](README.es.md).

**Regla accionable:** elegir un idioma y una configuración regional predeterminados para todos los textos, fechas y números visibles; centralizarlos para permitir una futura localización sin cadenas sueltas por componente.

## Error Handling

### E1. La UI descarta el error concreto de la petición

**Riesgo medio.** [frontend/src/App.tsx](frontend/src/App.tsx) lanza un error con el código HTTP cuando `response.ok` es falso, pero el `.catch()` ignora el valor y presenta siempre el mismo mensaje. Fallos de red, HTTP y parseo quedan indistinguibles, y no hay acción de reintento.

**Regla accionable:** conservar un tipo/estado de error que diferencie fallo de red y respuesta HTTP, mostrar un mensaje seguro y útil, registrar el detalle técnico sin datos sensibles y ofrecer reintento cuando sea pertinente.

### E2. El cálculo de facetas presupone una colección no vacía

**Riesgo medio al cambiar la fuente de datos.** `build_metrics_facets` accede a `ordered[0]` y `ordered[-1]` en [backend/app/routes.py](backend/app/routes.py); una colección vacía genera una excepción en lugar de una respuesta definida. Actualmente los handlers generan 360 movimientos antes de invocarlo, pero esa garantía depende de la implementación mock.

**Regla accionable:** especificar el contrato para resultados vacíos en cada función/API (respuesta vacía, valores de fecha opcionales o error HTTP deliberado) y probarlo antes de conectar una fuente que pueda no tener registros.

## Testing

### T1. No hay pruebas del flujo visual ni de sus estados de petición

**Cobertura ausente en el repositorio inspeccionado.** [frontend/src/lib/financial-utils.test.ts](frontend/src/lib/financial-utils.test.ts) prueba cálculos y formateadores; no se encontraron pruebas de [frontend/src/App.tsx](frontend/src/App.tsx), del fetch, del estado de error/carga ni de los gráficos.

**Regla accionable:** al cambiar el flujo de datos del dashboard, añadir pruebas de componente para carga exitosa, respuesta HTTP fallida y fallo de red, además de una comprobación de que las métricas presentadas corresponden al payload recibido.

### T2. Las pruebas backend no fijan el comportamiento en entradas vacías o intervalos inválidos

**Cobertura parcial.** [backend/tests/test_routes.py](backend/tests/test_routes.py) cubre endpoints, filtros habituales y algunas formas de respuesta, pero no el caso vacío de `build_metrics_facets` ni el contrato para `start_date > end_date`.

**Regla accionable:** para cada nuevo helper o filtro, probar al menos el caso nominal y los límites relevantes (vacío, parámetros fuera de rango o intervalo invertido) y afirmar explícitamente la respuesta o excepción esperada.

## DX

### D1. Las dependencias Python no están fijadas

**Riesgo medio de builds no reproducibles.** [backend/requirements.txt](backend/requirements.txt) enumera paquetes sin versiones; [backend/Dockerfile](backend/Dockerfile) instala directamente esas dependencias con `pip install`.

**Regla accionable:** fijar versiones compatibles o mantener un lock/archivo de restricciones reproducible para el backend y construir las imágenes desde ese conjunto verificado; actualizar dependencias de forma deliberada.

### D2. El proxy configurado presupone ejecución dentro de Compose

**Riesgo medio de configuración local.** [frontend/vite.config.ts](frontend/vite.config.ts) reenvía `/api` a `http://backend:8000`, hostname definido por Compose en [docker-compose.yml](docker-compose.yml). La ejecución de Vite directamente en el host no necesariamente puede resolver ese nombre; [frontend/.env.example](frontend/.env.example) ofrece `VITE_API_BASE_URL` para cambiar el origen, pero [README.es.md](README.es.md) no distingue claramente Compose de ejecución directa.

**Regla accionable:** documentar cada modo de arranque por separado y dar un valor explícito de `VITE_API_BASE_URL` para ejecución en host (`http://localhost:8000`); mantener el hostname `backend` para la red de Compose.

### D3. Compose no espera a que el backend esté listo

**Riesgo medio durante el arranque.** `depends_on` en [docker-compose.yml](docker-compose.yml) controla el orden de inicio, pero no define healthcheck. Como [frontend/src/App.tsx](frontend/src/App.tsx) hace una sola petición al montar, un backend que aún no acepta conexiones puede dejar el dashboard en estado de error hasta recargar la página.

**Regla accionable:** añadir una comprobación de salud de backend y condicionar el inicio del frontend a que esté saludable, o implementar reintento acotado del fetch; mantener documentado cuál de las dos capas garantiza la disponibilidad inicial.