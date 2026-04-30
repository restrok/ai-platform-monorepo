# k3s-gpu-orchestration

Fase 1: Fundaciones y Red (El terreno conocido)
Primero, armamos la base utilizando tus fortalezas actuales para aislar el entorno.

OS & Red: Instala Ubuntu Server (sin entorno gráfico) en la PC. Crea una VLAN específica en tu MikroTik para este nodo, aislándolo de tu red doméstica principal.

El Clúster: Despliega K3s (de Rancher). Es un binario único, consume muy pocos recursos y es el estándar de la industria para Edge Computing y laboratorios.

Infra as Code: Gestiona todo el despliegue del clúster utilizando Terraform.

Fase 2: La Capa de Abstracción de Hardware (El corazón del Lab)
Aquí es donde adquieres el conocimiento "enterprise" engañando a Kubernetes para que divida tu GTX 970.

NVIDIA Container Toolkit: Instala los drivers privativos de NVIDIA y el toolkit de contenedores a nivel de sistema operativo.

Device Plugin: Aplica el manifiesto oficial del k8s-device-plugin de NVIDIA en tu clúster K3s.

Time-Slicing ConfigMap: Esta es la tarea crítica. Configura el plugin pasándole un ConfigMap donde le indicas que divida la GPU física en, por ejemplo, 4 réplicas virtuales ([nvidia.com/gpu](https://nvidia.com/gpu): 4).

Fase 3: Observabilidad AI / MLOps (El diferencial)
Un modelo en producción sin monitoreo es inútil. Aquí brillas como SRE.

DCGM-Exporter: Despliega el NVIDIA Data Center GPU Manager Exporter como un DaemonSet.

El Stack Clásico: Levanta tu stack de Prometheus y Grafana (que ya dominas).

El Dashboard MLOps: Configura paneles específicos para monitorear el uso de VRAM por Pod, temperatura de la GPU, y ciclos de reloj. Esto demuestra que no solo despliegas modelos, sino que garantizas su disponibilidad.

Fase 4: Despliegue de Workloads Concurrentes (La Prueba de Fuego)
Para demostrar que el Time-Slicing funciona, necesitas correr cosas en paralelo sin que colapsen por falta de memoria.

Workload A (ETL): Contenedoriza tu script de extracción de datos de Garmin. Haz que corra como un CronJob en K8s para procesar los archivos FIT periódicamente.

Workload B (Inferencia Ligera): Levanta una pequeña API en Python (FastAPI) cargando un modelo de regresión clásico (ej. Scikit-learn o un XGBoost pequeño) entrenado previamente con tus datos de pulsaciones. Exige explícitamente en el YAML de este Pod el recurso [nvidia.com/gpu](https://nvidia.com/gpu): 1.

Resultado: Verás en Grafana cómo ambos pods comparten la GTX 970 de 4GB sin arrojar errores de OOM (Out of Memory).