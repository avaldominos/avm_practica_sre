KeepCoding - Tech School\
BootCamp - DevOps & Cloud Computing - Edicion X.

Módulo - Liberando productos\
Alumno: Alberto Valdominos Martín

# Practica Final

El proyecto inicial es un servidor que utiliza [FastAPI](https://fastapi.tiangolo.com/) para levantar un servidor en el puerto `8081` e implementa inicialmente dos endpoints, test unitarios y utiliza prometheus para levantar un servidor de metricas en el puerto `8000`.


Se modifica el proyecto inical de aplicacion FastAPI del repositorio facilitado, añadiendo las mejoras requeridas.


## Tabla de Contenidos de Mejoras Implementadas

1. [Añadir por lo menos un nuevo endpoint](#1-añadir-por-lo-menos-un-nuevo-endpoint)
2. [Creación de tests unitarios para el nuevo endpoint](#2-creación-de-tests-unitarios-para-el-nuevo-endpoint)
3. [Creación de pipelines de CI/CD](#3-creación-de-pipelines-de-cicd)
4. [Helm chart para desplegar la aplicación en Kubernetes](#4-helm-chart-para-desplegar-la-aplicación-en-kubernetes)
5. [Configuración de monitorización y alertas](#5-configuración-de-monitorización-y-alertas)
6. [Dashboard de Grafana](#6-dashboard-de-grafana)
7. [Recursos](#7-recursos)
8. [Licencia](#8-licencia)


## 1. Añadir por lo menos un nuevo endpoint

Se ha añadido un nuevo endpoint  `/bye` que devuelve `{"msg": "Bye Bye"}`a los existentes `/` y `/health`.
Para ello se ha modificado el fichero [src/application/app.py](./src/application/app.py).
Se ha incluido el siguiente codigo:

```
    @app.get("/bye")
    async def read_bye():
        """Implement bye endpoint"""
        # Increment counter used for register the total number of calls in the webserver
        REQUESTS.inc()
        # Increment counter used for register the total number of calls in the bye endpoint
        BYE_ENDPOINT_REQUESTS.inc()
        return {"msg": "Bye Bye"}
```


## 2. Creación de tests unitarios para el nuevo endpoint

Se ha modificado el fichero [app_test.py](./src/tests/app_test.py), añadiendo el siguiente codigo para probar el nuevo endpoint  `/bye`:

```
@pytest.mark.asyncio
    async def read_bye_test(self):
        """Tests the bye endpoint"""
        response = client.get("bye")

        assert response.status_code == 200
        assert response.json() == {"msg": "Bye Bye"}

```
Para probar la applicación se ejecuta el siguiente comando que genera un report de cobertura:

```
pytest --cov --cov-report=html
```

## 3. Creación de pipelines de CI/CD

Se ha creado pipeline CI/CD en la plataforma Github Actions que cuenta con dos fases:

  - Testing: realiza tests unitarios con cobertura. Ver archivo [./.github/workflows/test.yaml](./.github/workflows/test.yaml). El workflow se activa:\
        - Cada vez que se abre, actualiza o sincroniza una `pull request` en cualquier rama ('*').\
        - El flujo de trabajo también se activa cada vez que se hace `push` a cualquier rama ('*').\
    Esto garantiza que el workflow se ejecute tanto para solicitudes push y pull, manteniendo controles de calidad del código en todas las ramas.
 
  - Build & Push: crea imagen docker de la applicacion fast-api y realiza el push de la misma al registry GHCR. Ver archivo[./.github/workflows/test.yaml](./.github/workflows/release.yaml). El workflow se activa:\
        - Este workflow de GitHub Actions está diseñado para ejecutarse cada vez que se envía una nueva "tag" al repositorio, específicamente una etiqueta que comienza con v, como v1.0, v2.1, etc. Las etiquetas se usan normalmente para marcar versiones específicas de código, lo que hace que este workflow sea adecuado para automatizar el proceso de lanzamiento cada vez que esté lista una nueva versión del código.
        Los comandos git a ejecutar serian por ejemplo:

```
git tag -a v1.1 -m "Release version 1.1"
git push origin v1.1

```
La nueva imagen de docker se encuentra en el apartado de [packages](https://github.com/avaldominos/avm_practica_sre/pkgs/container/avm_practica_sre%2Ffast-api) del repositorio.

El comando docker para descargar la última versión de la imagen es:

```
docker pull ghcr.io/avaldominos/avm_practica_sre/fast-api:latest
```

Ejemplos válidos de la ejecución se pueden ver [aqui](https://github.com/avaldominos/avm_practica_sre/actions). 

## 4. Helm chart para desplegar la aplicación en Kubernetes

Una vez creada la imagen en el repositorio, se crea chart de helm para desplegar la aplicación en Kubernetes, utilizando como ejemplo el disponible en el ejemplo del lab realizado en la clase 3.

Fundamentalmente se ha actualizado los valores de la ruta de la imagen a descargar en el archivo [./fast-api-webapp/values.yaml](fast-api-webapp/values.yaml):

```
image:
  repository: ghcr.io/avaldominos/avm_practica_sre/fast-api
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "latest"
```

## 5. Configuración de monitorización y alertas

### 5.1. Monitorización mediante prometheus en los nuevos endpoints añadidos.

Al añadir el nuevo endpoint `/bye` en el archivo [src/application/app.py](./src/application/app.py) tambien se ha instrumentado la applicación para que la métrica estuviera disponible en Prometheus, mediante el siguiente codigo:

```
REQUESTS = Counter('server_requests_total', 'Total number of requests to this webserver')
HEALTHCHECK_REQUESTS = Counter('healthcheck_requests_total', 'Total number of requests to healthcheck')
MAIN_ENDPOINT_REQUESTS = Counter('main_requests_total', 'Total number of requests to main endpoint')
BYE_ENDPOINT_REQUESTS = Counter('bye_requests_total', 'Total number of requests to bye endpoint')
```

Estas cuatro métricas de tipo count son las que implementa directamente la applicación y estarán disponibles en Prometheus.


### 5.2. Desplegar prometheus a través de Kubernetes mediante minikube y configurar alert-manager

Se siguen los siguientes pasos para desplegar la aplicación FastAPI en minikube con Prometheus y Grafana:

1. Crear un cluster de Kubernetes que utilice la versión `v1.28.3` utilizando minikube para ello a través de un nuevo perfil llamado `avm-practica-sre`:

    ```sh
    minikube start --kubernetes-version='v1.28.3' \
    --cpus=4 \
    --memory=4096 \
    --addons="metrics-server,default-storageclass,storage-provisioner" \
    -p avm-practica-sre

    ```

2. Se configura slack siguiendo las indicaciones del laboratorio de la clase 3, creando el canal `alberto-prometheus-alarms` y se configura webhook entrante para envío de alertas con alert manager.


3. Se modifica el archivo [./kube-prometheus-stack/values.yaml](kube-prometheus-stack/values.yaml) modificando la configuración de `alertmanager` incluyendo el webhook creado y el canal:

    ```sh
    - name: 'slack'
        slack_configs:
        - api_url: 'https://hooks.slack.com/services/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' # <--- AÑADIR EN ESTA LÍNEA EL WEBHOOK CREADO
            send_resolved: true
            channel: '#alberto-prometheus-alarms' # <--- AÑADIR EN ESTA LÍNEA EL CANAL
            title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] Monitoring Event Notification'
            text: |-
            {{ range .Alerts }}
                *Alert:* {{ .Labels.alertname }} - `{{ .Labels.severity }}`
                *Description:* {{ .Annotations.message }}
                *Graph:* <{{ .GeneratorURL }}|:chart_with_upwards_trend:> *Runbook:* <{{ .Annotations.runbook_url }}|:spiral_note_pad:>
                *Details:*
                {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
                {{ end }}
            {{ end }}
    ```

    Se modifica tambien el apartado de `Alertmanager components` para crear alerta cuando el uso de CPU de un contenedor alcance el 90% del limite establecido en los recursos del contendor.

    ````sh
    - alert: fastApiConsumingMoreCPUThanLimit
        expr: avg(rate(container_cpu_usage_seconds_total{pod=~"my-app-fast-api-webapp-.*"}[1m])) by (pod) > 0.9 * avg(kube_pod_container_resource_limits{resource="cpu", container="fast-api-webapp"}) by (pod)
        for: 0m
        labels:
        severity: critical
        alertname: "fast-api-webapp container CPU usage exceeds limit"
        annotations:
        summary: Pod {{ $labels.pod }} CPU usage exceeds limit
        description: "The CPU usage of Pod {{ $labels.pod }} has exceeded the defined limit."
        message: Pod {{ $labels.pod }} is consuming more CPU than its limit
    
    ````



4. Se despliega prometheus en el cluster de minikube creado. Ejecutar los siguientes comandos en el terminal.

    Se añade el repositorio de helm prometheus-community para poder desplegar el chart `kube-prometheus-stack`.

    ```sh
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    ```

    ```sh
    helm -n monitoring upgrade \
    --install prometheus \
    prometheus-community/kube-prometheus-stack \
    -f kube-prometheus-stack/values.yaml \
    --create-namespace \
    --wait --version 55.4.0
    ```

    ```sh
    kubectl -n monitoring get po -w
    ```

5. Se añade el repositorio de mongodb y se despliega el operador de mongodb.

    ```sh
    helm repo add mongodb https://mongodb.github.io/helm-charts
    ```
    ```sh
    helm repo update
    ```
    ```sh
    helm upgrade --install community-operator \
    mongodb/community-operator \
    -n mongodb --create-namespace \
    --wait --version 0.9.0
    ```

6. Se despliega la aplicación mediante el chart de helm creado en el apartado 4. Ejecutar en diferentes terminales:

    ```sh
    helm -n fast-api upgrade my-app --wait --install --create-namespace fast-api-webapp
    ```

    ```sh
    kubectl -n fast-api get po -w
    ```

    ```sh
    kubectl get -n mongodb mongodbcommunity -w
    ```

    ```sh
    kubectl -n mongodb get ep mongodb-svc
    ```
    ```sh
    kubectl -n fast-api logs -f deployment/my-app-fast-api-webapp -c fast-api-webapp
    ```

7. Se realiza el port-forward de los servicios Grafana, Prometheus y fast-api y se accede a los mismos en el navegador (credenciales Grafana `admin` para el usuario y `prom-operator` para la contraseña)

    ```sh
    kubectl -n monitoring port-forward svc/prometheus-grafana 3000:http-web
    ```
    ```sh
    kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090
    ```
    ```sh
    kubectl -n fast-api port-forward svc/my-app-fast-api-webapp 8081:8081
    ```

    En este momento la aplicación está desplegada en el cluster creado y con monitorización de la misma mediante Prometheus y Grafana.

[Screenshot aplicación fast-api](https://github.com/avaldominos/avm_practica_sre/blob/main/img/fast-api.png)

[Screenshot Prometheus](https://github.com/avaldominos/avm_practica_sre/blob/main/img/prometheus.png)

### 5.3. Prueba de estres

Se realiza prueba de de estres, como la realizada en el laboratorio 3 a partir del paso 8. En varios terminales se ejecuta:

    ```sh
    kubectl -n fast-api get hpa -w
    ```

    ```sh
    kubectl -n fast-api get po -w
    ```
Una vez se ejecuta el test de stress se recibe notificación en el canal de slack.

A continuación se muestra una captura con un ejemplo de este tipo de notificación.

![Slack alarm example](./img/slack_alarm.png)


## 6. Dashboard de Grafana

Se configura un dashboard de Grafana visualizando las métricas requeridas. Se puede importar mediante el siguiente [archivo json](/avm_dashboard.json)

[Screenshot dashboard de Grafana](https://github.com/avaldominos/avm_practica_sre/blob/main/img/grafana.png)


## 7. Recursos

- [Documentación oficial Github Actions](https://docs.github.com/en/actions)
- [Documentación oficial FastAPi](https://fastapi.tiangolo.com/)
- [Documentación oficial de Kubernetes](https://kubernetes.io/docs/home/)\
      - [Documentación oficial minikube](https://minikube.sigs.k8s.io/docs/)
- [Documentación oficial Prometheus](https://prometheus.io/docs/introduction/overview/)
- [Documentación oficial Grafana](https://grafana.com/docs/)


## 8. Licencia
Este proyecto está bajo la licencia GNU GPL.
