# prometheus-user-metrics-demo
Despliegue de 2 aplicaciones en OpenShift 4.6 que exponen métricas para prometheus en un proyecto definido por el usuario.

**Instrucciones:**

 1. Con un usuario en rol **cluster-admin**, habilitar el monitoreo de proyectos definidos por el usuario:
  - Agregar  **cluster-monitoring-config** en el proyecto  **openshift-monitoring**
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
 2. Crear el proyecto **metrics-demo**
    ```
    oc new-project metrics-demo
    ```
 3. Para la aplicación prom-app, la cual hace uso de client library GO para prometheus:
  - Crear el despliegue y el servicio para la aplicación prom-app
    ```
    oc create -f prom-app.yml
    ```
  - Crear el CRD Service Monitor
    ```
    oc create -f sm-prom-app.yml
    ```
 4. Para la base de datos mariadb, la cual hace uso de exporter para prometheus:
  - Crear la aplicación mariadb
    ```
    oc new-app --name=mariadb --docker-image=docker.io/library/mariadb -e MYSQL_USER=user -e MYSQL_PASSWORD=user -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=database
    ```
  - Editar el deployment mariadb, y agregar el exporter de mysql
    ```
    oc edit deployment mariadb
    ```
    Quedando de la siguiente forma:
    ```yaml
    - env:
        - name: DATA_SOURCE_NAME
          value: "root:root@(127.0.0.1:3306)/database"
        image: prom/mysqld-exporter
        name: mariadb-exporter
        ports:
        - containerPort: 9104
          protocol: TCP
          name: mariadb-metrics
    ```
  - Crear el CRD Pod Monitor
    ```
    oc create -f pm-mariadb.yml
    ```
 5. Crear el CRD prometheusrule, el cual dispara una alerta de severidad critica cuando se han generado 2 o más tablas en la base de datos **database**
    ```
    oc create -f alert{1,2}-mariadb-create-table.yml
    ```
    Se puede elegir cualquiera de los yaml, la diferencia es que el **alert1-mariadb-create-table.yml** genera una regla que pasa por thanos-rule, mientras que **alert2-mariadb-create-table.yml** se depliega directamente en prometheus.
