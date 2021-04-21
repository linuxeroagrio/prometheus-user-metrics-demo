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
