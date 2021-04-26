# prometheus-user-metrics-demo
Despliegue de 2 aplicaciones en OpenShift 4.6 que exponen métricas para prometheus en un proyecto definido por el usuario. La primera aplicación hace uso de la librería cliente de Prometheus del leguaje GO para exponer sus métricas en el contexto /metrics; en la cual se realiza la configuración de ServiceMonitor para habilitar el monitoreo a nivel de Servicio. La segunda aplicación hace uso del exporter MySQL oficial de Prometheus para realizar su exposición  en el contexto /metrics y se hace uso de PodMonitor para habilitar el monitoreo a nivel de Pod. Existen pasos en los cuales se crean rutas, los cuales se pueden omitir, ya que son meramente de validación.

Una vez deplegadas ambas aplicaciones, es posible visualizarlas en la consola de OCP y Grafana (mediante la instalación del Operador Comunitario de Grafana). Se realiza la creación de alertas y canales de notificación.

# Configuración Previa
Se debe habilitar el monitoreo de proyectos definidos por el usuario, instalar Grafana Operator y realizar las configuraciones necesarias para Grafana tenga acceso a las métricas en los proyecyor definidos por el usuario y de sistema.

Las siguientes actividades deberán ejecutarse con un usuario que tenga el rol **cluster-admin**.

1. **Habiltar el monitoreo de métricas en proyectos definidos por el usuario:**
  - Editar el configmap  **cluster-monitoring-config** en el proyecto  **openshift-monitoring**
      ```yaml
      enableUserWorkload: true
      ```
      Por Ejemplo:
      ```yaml
      apiVersion:  v1
      kind:  ConfigMap
      metadata:
        name:  cluster-monitoring-config
        namespace:  openshift-monitoring
      data:
        config.yaml:  |
          enableUserWorkload: true
      ```
      > Nota: En caso de que el config map **cluster-monitoring-config** no exista, deberá generarse con la estructura mostrada arriba.
2. **Instalar Grafana Operator Comunitario:**

    Debido a que el Grafana instalado en la versión por defecto de OpenShift es de solo lectura, para conseguir visualizar las métricas en los proyectos definidos por el usuario y crear Dashboards personalizados, es necesario instalar la versión comunitaria de Grafana Operator desde Operator Hub.
  - Crear el proyecto en donde se instalará el operador (En este caso se configuro **custom-grafana**)
      ```bash
      oc new-project custom-grafana
      ```
  - En la consola de OpenShift, navegar hacia **OperatorHub** y buscar **Grafana**
      
      ![Búsqueda de Grafana en OperatorHub](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/BusquedaDeGrafanaOperatorHub.png)
  - Seleccionar **Operator Grafana**, presionar **Continue**
  - Aceptar los términos
      
      ![Términos de los operadoradores comunitarions](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/disclaimer.png)
  - Presionar **Install**

      ![Información del operador Grafana](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/GrafanaOperatorInfo.png)
  - Presionar nuevamente **Install** para aceptar la configuración por defecto y desplegar en el proyecto **custom-grafana**

      ![Grafana Operator Subscribe Options](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/GrafanaOperatorSubscribeOptions.png)
  - Verificar la instalación en progreso

      ![Búsqueda de Grafana en OperatorHubInstalación de Grafana Operator en Progreso](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/InstallingGrafanaOperator.png)
  - Verificar Operador Instalado Correctamente

      ![Operador Instalado Correctamente](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/GrafanaOperatorInstalled.png)
3. **Configurar Grafana Personalizado para visualizar las métricas de sistema y las de proyectos definidos por el usuario:**
  - En una terminal, crear el CRD grafana definido en el archivo **grafana-intance.yaml**; el cual generará la instancia de Grafana **grafana-instance** que tendra las credenciales **root**/**secret**
      ```bash
      oc create -f grafana-instance.yaml
      ```
  - Asignar el rol de cluster **cluster-monitoring-view** al service account **grafana-serviceaccount**
      ```bash
      oc adm policy add-cluster-role-to-user cluster-monitoring-view -z grafana-serviceaccount
      ```
  - Crear el secreto **token-grafana-sa** que incluye la variable TOKEN_SA, la cual contiene el TOKEN del Service Account **grafana-serviceaccount**
      ```bash
      oc create secret generic token-grafana-sa --from-literal=TOKEN_SA=$(oc serviceaccounts get-token grafana-serviceaccount)
      ```
  - Crear el origen de datos de Grafana **prometheus-grafanadatasource** mediante la definición del CRD GrafanaDataSource encontrada en el archivo **grafana-datasource.yaml**
      ```bash
      oc create -f grafana-datasource.yaml
      ```
      Este paso generará una configuración de Origen de Datos de Prometheus para grafana que apunte al servicio de **thanos-querier** en el proyecto **openshift-monitoring**; el cual es un punto central para la recolección de métricas de sistema y de proyectos definidos por el usuario, de acuerdo al diagrama siguiente:

      ![Configuración de Origen de Datos](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/DataSourceThanosPrometheusGrafana.png)
  - Para validar que la configuración sea correcta, ingresar a Grafana, primero, obtener la ruta:
      ```bash
      oc get routes -n custom-grafana
      ```
  - Acceder en un explorador web a la URL de Grafana, dar click en **Sign In**:

      ![Sig In Grafana](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/SignInGrafana.png)
  - Ingresar como usuario administrativo con las credenciales **root**/**secret**:

      ![Log In Grafana](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/LogInGrafana.png)
  - Ir a **Configuration**->**Data Sources**:

      ![Seleccionar Data Sources Grafana](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/MenuDataSourcesGrafana.png)
  - Seleccionar el Origen de Datos **Prometheus**:

      ![Seleccionar Prometheus DS](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/PrometheusDS.png)
  - Validar configuración y sin modificar, hacer click en **Save & Test**:

      ![Test Data Source](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/TestPrometheusDS.png)

# Despliegue de Aplicación prom-app: Uso de Service Monitor
Los pasos para desplegar una aplicación que hace uso de una librería cliente de Prometheus para exposición de métricas y habilitar su visualización en la consola de OpenShift a traves del CRD ServiceMonitor son los siguientes:
1. **Creación del proyecto metrics-demo como espacio de trabajo para el despliegue de las aplicaciones:**
  - En una terminal, crear el proyecto **metrics-demo**
      ```bash
      oc new-project metrics-demo
      ``` 
2. **Despliegue de la aplicación:**
  - Crear los objetos Deployment y Service de acuerdo **prom-app** a la definición del archivo prom-app.yml
      ```bash
      oc create -f prom-app.yml -n metrics-demo
      ```
3. **[Opcional] Validación de exposición de métricas en el contexto /metrics:**
  - Crear la ruta prom-app exponiendo el servicio prom-app
      ```bash
      oc expose service prom-app -n metrics-demo
      ```
  - Probar la aplicación
      ```bash
      curl -w '\n' $(oc get routes prom-app --template={{.spec.host}} -n metrics-demo)
      ```
  - Obtener las métricas expuestas
      ```bash
      curl $(oc get routes prom-app --template={{.spec.host}} -n metrics-demo)/metrics
      ```
4. **Habilitar y Visualizar las métricas de la aplicación en OpenShift:**
  - Crear el CRD ServiceMonitor **prom-app-monitor** de acuerdo a la definición del archivo **sm-prom-app.yml**
      ```bash
      oc create -f sm-prom-app.yml -n metrics-demo
      ```
  - En la consola de administración de OpenShift, sleccionar la vista **Developer** e ir a **Monitoring** pestaña **Metrics**.

      ![MetricsApp1](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/metricsapp1.png)

      > Asegurarse que el proyecto metrics-demo este seleccionado en la parte superior.
  - Seleccionar **Custom Query**, escribir un PromQL de Prometheus (En ele ejemplo, se ejecuta la visualización de la métrica **http_request_duration_seconds_count{code="200",handler="found",method="get"}** y presionar **Enter**, se mostrará una gráfica si el PromQL esta correctamente formado. 

      ![QueryApp1](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/queryapp1.png)
