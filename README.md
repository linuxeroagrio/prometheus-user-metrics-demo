# prometheus-user-metrics-demo
Despliegue de 2 aplicaciones en OpenShift 4.6 que exponen métricas para prometheus en un proyecto definido por el usuario. La primera aplicación hace uso de la librería cliente de Prometheus del leguaje GO para exponer sus métricas en el contexto /metrics; en la cual se realiza la configuración de ServiceMonitor para habilitar el monitoreo a nivel de Servicio. La segunda aplicación hace uso del exporter MySQL oficial de Prometheus para realizar su exposición  en el contexto /metrics y se hace uso de PodMonitor para habilitar el monitoreo a nivel de Pod. Existen pasos en los cuales se crean rutas, los cuales se pueden omitir, ya que son meramente de validación.

Una vez deplegadas ambas aplicaciones, es posible visualizar las métricas expuestas en la consola de OCP y Grafana (mediante la instalación del Operador Comunitario de Grafana). 

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

# Despliegue de Aplicación prom-app: Uso Librería Cliente Prometheus y Service Monitor
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
  - Crear la ruta prom-app exponiendo el servicio **prom-app**
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
  - Seleccionar **Custom Query**, escribir un PromQL de Prometheus (En el ejemplo, se ejecuta la visualización de la métrica **http_request_duration_seconds_count{code="200",handler="found",method="get"}** y presionar **Enter**, se mostrará una gráfica si el PromQL esta correctamente formado. 

      ![QueryApp1](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/queryapp1.png)
# Despliegue de Base de Datos MariDB: Uso Exporter Prometheus y Pod Monitor
Los pasos para desplegar una aplicación que hace uso de exporter de Prometheus de Prometheus para exposición de métricas y habilitar su visualización en la consola de OpenShift a traves del CRD PodMonitor son los siguientes:
1. **Asegurarse de trabajar en el proyecto metrics-demo:**
  - En una terminal, seleccionar el proyecto **metrics-demo**
      ```bash
      oc project metrics-demo
      ``` 
2. **Despliegue de la base de datos:**
  - Crear la aplicación **mariadb** teniendo como fuenta la imagen de contenedor MariaDB de DockerHub; la creara la base de datos **database** con las credenciales **user**/**user** para usuario común y **root/root** para usuario privilegiado.
      ```bash
      oc new-app --name=mariadb --docker-image=docker.io/library/mariadb -e MYSQL_USER=user -e MYSQL_PASSWORD=user -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=database -n metrics-demo
      ```
3. **Configurar Exporter de Prometheus:**
  - Editar el despliegue **mariadb**, agrgando el contenedor con la imagen oficial del exporter de Prometheus para MySQL. En la definición se grega la variable de entorno DATA_SOYRCE que apunta la base de datos y se define el puerto por el cual será expuesto el servicio web que dará lugar al contexto /metrics. 
      ```bash
      oc edit deployment/mariadb -n metrics-demo
      ```
      Se debera agregar a la sección containers:
      ```yaml
      - env:
        - name: DATA_SOURCE_NAME
          value: root:root@(127.0.0.1:3306)/database
        image: prom/mysqld-exporter
        name: mariadb-exporter
        ports:
        - containerPort: 9104
          name: mariadb-metrics
          protocol: TCP
      ```
      Por Ejemplo:

      ![ExporterContainerDefinition](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/exporterContainerDefinition.png)
4. **[Opcional] Validación de exposición de métricas en el contexto /metrics:**
  - Crear el servicio **mariadb-metrics** que apunta al puerto 9104 del pod **mariadb**
      ```bash
      oc expose deployment/mariadb --name=mariadb-metrics --port=9104 --target-port=9104 -n metrics-demo
      ```
  - Crear la ruta **mariadb-metrics** exponiendo el servicio **mariadb-metrics**
      ```bash
      oc expose service mariadb-metrics -n metrics-demo
      ```
  - Obtener las métricas expuestas
      ```bash
      curl $(oc get routes mariadb-metrics --template={{.spec.host}} -n metrics-demo)/metrics
      ```
5. **Habilitar y Visualizar las métricas de la base de datos en OpenShift:**
  - Crear el CRD PodMonitor **pod-mariadb-monitor** de acuerdo a la definición del archivo **pm-mariadb.yml**
      ```bash
      oc create -f pm-mariadb.yml -n metrics-demo
      ```
  - En la consola de administración de OpenShift, sleccionar la vista **Developer** e ir a **Monitoring** pestaña **Metrics**.

      ![MetricsApp1](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/metricsapp1.png)

      > Asegurarse que el proyecto metrics-demo este seleccionado en la parte superior.
  - Seleccionar **Custom Query**, escribir un PromQL de Prometheus (En el ejemplo, se ejecuta la visualización de la métrica **mysql_up** y presionar **Enter**, se mostrará una gráfica si el PromQL esta correctamente formado. 

      ![QueryApp2](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/queryapp2.png)
# Configuración de Alertas haciendo uso de CRD PrometheusRule
En esta sección, se mostraá como configurar una alerta en la base de datos mariadb desplegada y configurada en la sección anterior. Los pasos para configurar alertas en las métricas de los proyectos definidos por el usuario, son los siguientes:
1. **Asegurarse de trabajar en el proyecto metrics-demo:**
  - En una terminal, seleccionar el proyecto **metrics-demo**
      ```bash
      oc project metrics-demo
      ``` 
2. **Configuración de la alerta:**
  - Crear el CRD PrometheusRule **mariadb-alert-rules** de acuerdo a la definición encontrada en los archivos **alert{1,2}-mariadb-create-table.yml**. La estructura especifica el disparo de una alarma de severidad crítica si se han ejecutado 2 ó más creaciones de tabla (Mediante el comando CREATE TABLE), para que sea disparada, la condición debera sostenerse por 1 segundo o más.
      ```bash
      oc create -f alert1-mariadb-create-table.yml -n metrics-demo
      ```
      ó
      
      ```bash
      oc create -f alert2-mariadb-create-table.yml -n metrics-demo
      ```
      > Nota: Crear solo uno de las 2 definiciones mostradas. La diferencia entre **alert1-mariadb-create-table.yml** y **alert2-mariadb-create-table.yml** es que la primera regla se despliega para ser tratada por el componente **thanos-ruler**, mientras que la segunda, es desplegada directamente en **Prometheus**. La aplicación de la segunda es recomendada, cuando en ninguna de las regas es utilizada una métrica del sistema, lo que ayuda a mejorar el rendimiento de ejecución de la misma.
3. **Forzar Disparo de la Alarma:**
  - En la consola de administración de OpenShift, sleccionar la vista **Administrator** e ir a **Monitoring**->**Alerting** . Eliminar el filtro **Platform** y agregar el filtro **User**. El resultado es que no exista ninguna alarma relacionada a la regla **create-table-alert**

      ![User Alerting Rules](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/UserAlerts1.png)
  - En la terminal, hacer un port forward al puerto 3306 al pod de mariadb.
      ```bash
      oc port-forward $(oc get pods --no-headers -o custom-columns=":metadata.name" -l deployment=mariadb -n metrics-demo) 3306:3306
      ```
  - En otra terminal, conectarse a la base de datos.
      ```bash
      mysql -u root -proot -h 127.0.0.1 -P 3306 --protocol=TCP database
      ```
  - Crear dos tablas.
      ```sql
      CREATE TABLE uno(uno INT);
      CREATE TABLE dos(dos INT);
      exit
      ```
  - En la consola de OpenShift, validar el disparo de la alerta.

      ![Disparo Alerta](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/disparoalerta1.png)
      ![Inforamación Alerta](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/informacionalerta1.png)
      ![Visualizacion Alerta](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/visualizacionalerta1.png)
# Configuración de DashBoard en Grafana para visualización de métricas en proyectos definidos por el usuario
En esta sección, se mostraá como desplegar un dashboard en Grafana haciendo uso del CRD GrafanaDashboard.
1. **Asegurarse de trabajar en el proyecto custom-grafana:**
  - En una terminal, seleccionar el proyecto **custom-grafana**
      ```bash
      oc project custom-grafana
      ``` 
2. **Creación y validación del Dashboard:**
  - Crear el CRD GrafanaDashBoard **create-tables-mysql** de acuerdo a la definición encontrada en el archivo **grafana-dashboard.yaml**. La estructura especifica la definición de un dashboard simple que muestra el conteo de la ejecución del comando **CREATE TABLE** en una base de datos MariaDB, además, configura una alerta que se dispara cuando se ha ejecutado el comando más de 5 veces en un periodo sostenido de 1 minuto por una evaluación cada 30 segundos.
      ```bash
      oc create -f grafana-dashboard.yml -n custom-grafana
      ```
  - En la interfaz de Grafana ir a **Dashboargs**->**Manage**:

      ![DashBoards Grafana](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/DashboardsGrafana.png)
  - Seleccionar el Dashboard **create-tables-mysql**, visualizar la recolección de la métrica de creación de tablas.

      ![DashBoards Grafana](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/dashboardGrafanaCreateTable.png)
  - Seleccionar el nombre del DashBoard y Dar Click en Editar. Visualizar el query y la configración de la alarma.

      ![EditDashBoard](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/editDashboard.png)
      ![Query DashBoard Grafana](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/QueryDashBoard.png)
      ![Grafana Alert Config](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/GrafanaAlertConfig.png)
3. **Forzar Disparo de la Alarma:**
  - En la terminal, hacer un port forward al puerto 3306 al pod de mariadb.
      ```bash
      oc port-forward $(oc get pods --no-headers -o custom-columns=":metadata.name" -l deployment=mariadb -n metrics-demo) 3306:3306
      ```
  - En otra terminal, conectarse a la base de datos.
      ```bash
      mysql -u root -proot -h 127.0.0.1 -P 3306 --protocol=TCP database
      ```
  - Crear cuatro tablas.
      ```sql
      CREATE TABLE tres(tres INT);
      CREATE TABLE cuatro(cuatro INT);
      CREATE TABLE cinco(cinco INT);
      CREATE TABLE seis(seis INT);
      exit
      ```
  - En la interfaz de Grafana, esperar al menos 1 minuto y validar el disparo de la alerta.

      ![Navegacion a Alerta](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/navegacionGrafanaAlerta.png)
      ![Estado de la alerta](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/EstadoAlertaGrafana.png)
# Conclusion
Se ha realizado la configuración de OpenShift para activar el monitoreo de métricas en los proyectos definidos por el usuario, y se ha realizado la visualización de métricas y configuración alertas tanto en la consola de OpenShift como en una instalación de Grafana haciendo uso del operator comunitario para dos aplicaciones desplegadas. 
# Referencias
[Stack de Monitoreo en OCP 4.6](https://docs.openshift.com/container-platform/4.6/monitoring/)

[Documentación Oficial de Prometheus](https://prometheus.io/docs/introduction/overview/)

[Client Libraries de Prometheus](https://prometheus.io/docs/instrumenting/clientlibs/)

[Exporters de Prometheus](https://prometheus.io/docs/instrumenting/exporters/)

[Exporter de MySQL](https://github.com/prometheus/mysqld_exporter)

[Documentación del Operador de Grafana](https://github.com/integr8ly/grafana-operator/tree/v3.10.0/documentation)

[Configuración de Grafana](https://grafana.com/docs/grafana/latest/administration/configuration/)
