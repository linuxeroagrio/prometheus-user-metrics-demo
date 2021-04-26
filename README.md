# prometheus-user-metrics-demo
Despliegue de 2 aplicaciones en OpenShift 4.6 que exponen métricas para prometheus en un proyecto definido por el usuario; visualizarlas en la consola de OCP y Grafana (mediante la instalación del Operador Comunitario de Grafana). Creación de alertas y canales de notificación.

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
      ```
      oc new-proyect custom-grafana
      ```
  - En la consola de OpenShift, navegar hacia **OperatorHub** y buscar **Grafana**.
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
      ![Operador Instalado Correctamente](https://gitlab.consulting.redhat.com/consulting-mw-redhat-mx/prometheus-user-metrics-demo/-/raw/main/images/GrafanaOperatorSubscribeOptions.png)
