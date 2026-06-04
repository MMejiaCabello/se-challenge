# Flight Delay Prediction API

## Problema

El objetivo del challenge es llevar a una forma productiva el trabajo exploratorio hecho en `challenge/exploration.ipynb`. El modelo predice si un vuelo asociado al aeropuerto SCL tendra atraso operacional.

Para entrenamiento se define `delay = 1` cuando la diferencia entre `Fecha-O` y `Fecha-I` es mayor a 15 minutos. En caso contrario, `delay = 0`.

Del notebook se transcribieron tambien las variables derivadas que se usaron durante el analisis:

- `high_season`
- `min_diff`
- `period_day`
- `delay`

En la version final no todas entran como variables del modelo. Se mantienen en la preparacion porque explican el criterio de negocio y dejan el codigo alineado con el analisis original.

## Modelo

Se mantuvo el enfoque propuesto al final del notebook: usar variables categoricas del vuelo y convertirlas a dummies. Las variables base usadas son:

- `OPERA`
- `TIPOVUELO`
- `MES`

El modelo final usa las 10 variables mas relevantes identificadas en el notebook:

- `OPERA_Latin American Wings`
- `MES_7`
- `MES_10`
- `OPERA_Grupo LATAM`
- `MES_12`
- `TIPOVUELO_I`
- `MES_4`
- `MES_11`
- `OPERA_Sky Airline`
- `OPERA_Copa Air`

Se eligio `LogisticRegression` con `class_weight="balanced"`. En el analisis original no habia una diferencia importante frente a XGBoost para este set reducido de variables, y la regresion logistica es mas simple de operar, mas liviana para servir en una API y suficiente para pasar el criterio de recall de la clase con atraso.

## API

La API esta implementada con FastAPI en `challenge/api.py`.

Endpoints:

- `GET /health`: retorna `{"status": "OK"}`.
- `POST /predict`: recibe una lista de vuelos y retorna una lista de predicciones.

Ejemplo de request:

```json
{
  "flights": [
    {
      "OPERA": "Grupo LATAM",
      "TIPOVUELO": "N",
      "MES": 3
    }
  ]
}
```

Ejemplo de response:

```json
{
  "predict": [0]
}
```

La API valida que:

- `OPERA` exista dentro de las aerolineas observadas en el dataset.
- `TIPOVUELO` sea `I` o `N`.
- `MES` este entre 1 y 12.

Cuando el payload no cumple estas reglas se responde con status `400`.

## Ejecucion Local

Instalar dependencias:

```bash
pip install -r requirements.txt
pip install -r requirements-test.txt
```

Correr tests del modelo:

```bash
make model-test
```

Correr tests de API:

```bash
make api-test
```

Levantar API local:

```bash
uvicorn challenge.api:app --host 0.0.0.0 --port 8000
```

Correr stress test local:

```bash
make stress-test
```

## Docker

Construir imagen:

```bash
docker build -t challenge-mle-api .
```

Ejecutar contenedor:

```bash
docker run --rm -p 8000:8000 challenge-mle-api
```

Probar healthcheck:

```bash
curl http://127.0.0.1:8000/health
```

## Despliegue en Cloud Run

La API esta preparada para Cloud Run. El contenedor escucha el puerto definido por la variable `PORT`, que Cloud Run inyecta al iniciar el servicio.

Comandos usados para desplegar desde la raiz del repositorio:

```bash
gcloud auth login
gcloud config set project <gcp-project-id>
gcloud services enable run.googleapis.com cloudbuild.googleapis.com

gcloud run deploy challenge-mle-api \
  --source . \
  --region us-central1 \
  --allow-unauthenticated
```

Despues del despliegue, la URL publica del servicio debe configurarse en `STRESS_URL` dentro del `Makefile` para ejecutar el stress test contra la API desplegada.

URL desplegada:

```text
https://challenge-mle-api-43741751766.us-central1.run.app
```

El stress test se ejecuto contra esa URL con 100 usuarios y 60 segundos de duracion.

## CI/CD

Se dejaron workflows en `workflows/` y en `.github/workflows/`.

- CI corre en push y pull request hacia `main` o `develop`.
- CI instala dependencias y ejecuta `make model-test` y `make api-test`.
- CD queda como workflow manual. Por ahora construye la imagen Docker y deja listo el punto de extension para publicar la imagen y desplegarla cuando existan credenciales cloud.

Para un despliegue real, el siguiente paso natural seria publicar esta imagen en un registry y desplegarla en Cloud Run. En ese caso se debe actualizar `STRESS_URL` en el `Makefile` con la URL publica del servicio y correr `make stress-test` contra esa URL.
