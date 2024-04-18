# Whitestack Challenge #1

La misión de este desafío fue lograr desplegar una aplicación dentro de un cluster de Kubernetes, manteniendo intactas sus funcionalidades.

La aplicación seleccionada para este desafío es [Tengen-Tetris](https://github.com/aitorperezzz/tengen-tetris), una web application creada usando Python Flask en el backend y p5.js en el frontend, la cual permite jugar Tetris en modo multijugador dentro de una misma red.

Para ello se realizaron los siguientes pasos:

## Crear una imagen de Docker de la webapp

Debido a que la aplicación utiliza una base de datos SQLite para la persistencia de los datos, se optó por crear un usuario y otorgarle permisos sobre el directorio de la aplicación. También se añaden variables de entorno que permitirán modificar la dirección y puerto desde el cual se sirve la aplicación.

```go
# syntax=docker/dockerfile:1

# Comments are provided throughout this file to help you get started.
# If you need more help, visit the Dockerfile reference guide at
# https://docs.docker.com/go/dockerfile-reference/

# Want to help us make this template better? Share your feedback here: https://forms.gle/ybq9Krt8jtBL3iCk7

ARG PYTHON_VERSION=3.11.9
FROM python:${PYTHON_VERSION}-alpine as base

# Prevents Python from writing pyc files.
ENV PYTHONDONTWRITEBYTECODE=1

# Keeps Python from buffering stdout and stderr to avoid situations where
# the application crashes without emitting any logs due to buffering.
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Create a non-privileged user that the app will run under.
# See https://docs.docker.com/go/dockerfile-user-best-practices/
# ARG UID=10001
# RUN adduser \
#     --disabled-password \
#     --gecos "" \
#     --home "/nonexistent" \
#     --shell "/sbin/nologin" \
#     --no-create-home \
#     --uid "${UID}" \
#     appuser

# Download dependencies as a separate step to take advantage of Docker's caching.
# Leverage a cache mount to /root/.cache/pip to speed up subsequent builds.
# Leverage a bind mount to requirements.txt to avoid having to copy them into
# into this layer.
RUN --mount=type=cache,target=/root/.cache/pip \
    --mount=type=bind,source=requirements.txt,target=requirements.txt \
    python -m pip install -r requirements.txt

# Copy the source code into the container.
COPY . .

# Create a non-privileged user that the app will run under.
RUN adduser -D appuser && \
    chown -R appuser:appuser /app

# Switch to the non-privileged user to run the application.
USER appuser

# Environment variables for Flask.
ENV FLASK_HOST=0.0.0.0
ENV FLASK_PORT=8080

# Expose the port that the application listens on.
EXPOSE 8080

# Run the application.
CMD python application.py
```

## Crear manifiestos k8s

Se crean los manifiestos para desplegar, configurar y exponer la aplicación. Se optó por utilizar un ingress-nginx y utilizar session affinity mediante cookies para el manejo de las conexiones. Se escapa del scope del desafío la implementación de una base de datos común en caso de trabajar con más réplicas.

- Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tengen-tetris
spec:
  selector:
    matchLabels:
      app: tengen-tetris
  replicas: 1
  template:
    metadata:
      labels:
        app: tengen-tetris
    spec:
      containers:
      - name: tengen-tetris
        image: felipemunozri/tengen-tetris:2.0
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: tengent-tetris
```

- Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: tengen-tetris
spec:
  selector:
    app: tengen-tetris
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

- Configmap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tengen-tetris
data:
  flask_host: 0.0.0.0
  flask_port: "8080"
```

- Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tengen-tetris
  annotations:
    # nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-path: "/"
spec:
  ingressClassName: nginx
  rules:
  - host: tengen-tetris.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tengen-tetris
            port:
              number: 8080
```

## Crear un Helm chart

Se crean los templates que permitirán hacer el despliegue de los elementos k8s mediante un helm install. Se crea también el archivo values que permite pasar opciones de configuración al chart.

- Values
```yaml
# app name info
appName: tengen-tetris
# deployment info
deployment:
  replicas: 1
  image:
    name: felipemunozri/tengen-tetris
    tag: "2.0"
  pullPolicy: Always
  resources:
    limits:
      memory: 128Mi
      cpu: 500m
# configmap info
configmap:
  name: tengen-tetris
  data:
    serverHost: 0.0.0.0
    serverPort: 8080
# ingress info
ingress:
  className: nginx
  host: tengen-tetris.test
```

## Despliegue de la webapp en un cluster k8s

Comprobación del despliegue de la aplicación en un cluster de Kubernetes (Docker Desktop). Se corrobora la funcionalidad de los modos un jugador, multijugador, y del score board.

- Docker Desktop

![Captura desde 2024-04-18 13-21-31](https://github.com/felipemunozri/tengen-tetris/assets/52138009/98b2f976-e1b9-41f3-a35d-017937d27964)


- Un jugador

![Captura desde 2024-04-18 13-11-15](https://github.com/felipemunozri/tengen-tetris/assets/52138009/60ded7f9-f7c4-4052-9d29-9f797215f00e)

- Multijugador

![Captura desde 2024-04-18 13-06-07](https://github.com/felipemunozri/tengen-tetris/assets/52138009/72446617-d35a-4028-8fea-1c3f6ca52f1d)

- Score Board
  
![Captura desde 2024-04-18 13-17-23](https://github.com/felipemunozri/tengen-tetris/assets/52138009/4d83e368-36b9-4954-a3af-98bd4143191f)






