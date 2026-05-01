Plan de Estudio Técnico: Laboratorio LLMOps E2E de Bajo Costo en GKE

Este plan de estudio ha sido diseñado para transformar el conocimiento teórico en capacidad arquitectónica real. A diferencia de los codelabs empaquetados que ocultan las complejidades operativas, este laboratorio exige la construcción del stack desde cero mediante Infraestructura como Código (IaC). Como arquitectos, nuestra meta no es solo "hacerlo funcionar", sino optimizar la relación costo-rendimiento utilizando hardware accesible como las GPUs NVIDIA L4, evitando la burocracia y los costos prohibitivos de las TPUs.

1. Fundamentos de Arquitectura de Inferencia Desagregada

La inferencia de modelos Transformer es inherentemente ineficiente cuando se ejecuta de forma monolítica. Debemos entender la separación física de las fases para eliminar cuellos de botella:

* Prefill (Fase de Prefijo): Procesa el prompt de entrada para calcular los estados de atención iniciales. Es una operación compute-bound (limitada por cómputo) debido a las intensas multiplicaciones de matrices. En arquitecturas compartidas, un prefill pesado bloquea los pasos de decodificación de otros usuarios.
* Decode (Fase de Decodificación): Genera tokens de forma autoregresiva. Es un proceso memory-bound (limitado por ancho de banda de memoria) que depende de la velocidad de lectura de los pesos desde la HBM (High Bandwidth Memory). Aquí, los ciclos de cómputo suelen desperdiciarse.

La Desagregación Prefill-Decode (PD Disaggregation) rompe este conflicto asignando cada fase a nodos especializados, permitiendo que el prefill no degrade la interactividad del decode.

Métricas Crave de Rendimiento (North Star Metrics)

Métrica	Fase Crítica	Impacto Arquitectónico
TTFT (Time to First Token)	Prefill	Sensibilidad del sistema. Reducir mediante cómputo masivo o KV Cache Hits.
TPOT (Time Per Output Token)	Decode	Velocidad de lectura percibida. Depende del ancho de banda de memoria.
Throughput (Tokens/Sec)	Ambas	Capacidad total del sistema. Optimizado mediante batching y desacoplamiento.
Queue Latency	Global	Tiempo de espera en cola antes del procesamiento.

2. Configuración de la Capa Física (Módulo 1: Infraestructura)

Para este laboratorio, utilizaremos Terraform en la carpeta 1-infra/. El despliegue se basa en un diseño de "AI Hypercomputer" simplificado para bajo costo.

Redes y VPC: El Mandato del MTU

La configuración de red no es negociable. Se debe implementar una VPC personalizada con un MTU de 8896 (Jumbo Frames).

* Justificación: El tráfico de tensores entre nodos de prefill y decode mediante NCCL (NVIDIA Collective Communications Library) es masivo. Un MTU estándar de 1500 provocaría fragmentación de paquetes, degradando la estabilidad y el rendimiento de la inferencia distribuida.

GKE Standard: Gestión de Node Pools

Evitaremos GKE Autopilot para mantener control total sobre la programación de hardware. Configuraremos tres pools específicos:

1. System Pool: Nodos de CPU estándar para servicios de gestión.
2. GPU Pools (L4): Dos pools especializados utilizando instancias g2-standard-12 (1x L4) o g2-standard-24 (2x L4).
  * Taints: Es obligatorio aplicar Taints estrictos (nvidia.com/gpu:NoSchedule) para asegurar que la VRAM sea territorio exclusivo del modelo, evitando "noisy neighbors".
  * Instancias Spot: Para profesionales conscientes del presupuesto, el uso de nodos Spot es el estándar de facto, permitiendo operar por debajo de los $5 USD/hora.

3. Middleware y Orquestación (Módulo 2: Platform)

La capa de plataforma (2-platform/) actúa como el sistema operativo de nuestra infraestructura de IA.

* KubeRay Operator: Fundamental para la coordinación. A diferencia de un pod estático, vLLM requiere un nodo "Head" que gestione el estado global de la memoria y nodos "Workers" que ejecuten el cómputo.
* Inference Extension API & kgateway: Implementaremos el ruteo mediante kgateway (basado en Envoy). Esta capa no es un balanceador de carga HTTP simple; es un motor de ruteo inteligente que implementa el Endpoint Picker Protocol (EPP), permitiendo seleccionar el backend basándose en la utilización del KV Cache y la profundidad de la cola.

4. Observabilidad y Monitoreo de LLMOps (Módulo 3: LLMops)

Sin métricas granulares, el arquitecto está "volando a ciegas". La configuración de dcgm-exporter debe ser el pilar de nuestra telemetría.

Especificación de Recolección (PromQL Targets)

Debemos capturar métricas que revelen la saturación del hardware:

* dcgm_gpu_temp: Para monitorear el estrangulamiento térmico.
* vllm:kv_cache_usage_ratio: El indicador clave de salud. Un ratio del 100% indica que el sistema comenzará a desalojar contexto o a re-computar.
* dcgm_tensor_copy_util: Para identificar si estamos limitados por cómputo (Compute-bound).

Dashboard de Grafana

El diseño debe incluir paneles que correlacionen el Uso de KV Cache con la Latencia de la Cola. Si la latencia aumenta mientras la VRAM está llena, hemos identificado un cuello de botella de memoria que requiere escalado horizontal o políticas de Tiered Storage.

5. Despliegue de Workloads e Inferencia Distribuida (Módulo 4: Workloads)

Utilizaremos llm-d, el framework de Red Hat, Google e IBM, para gestionar la inferencia distribuida.

* Configuración de Modelo: Para este laboratorio, el "caballo de batalla" será Llama-3-8B-Instruct. Aunque la infraestructura es capaz de servir Qwen3-32B o Llama-3.1-405B, el modelo 8B garantiza que las pruebas de carga no agoten el presupuesto.
* Workload Identity: No se permite el uso de llaves estáticas. Debemos configurar Workload Identity para que los pods de vLLM accedan de forma segura a los buckets de GCS para cargar pesos y secretos de Hugging Face.
* Ruteo de Inferencia: En inference-routing.yaml, definiremos InferencePool (el conjunto de recursos) e InferenceModel (la abstracción del servicio), permitiendo al Gateway realizar ruteo basado en la criticidad de la solicitud (InferenceObjective).

6. Gestión Avanzada de KV Cache y GKE Inference Gateway

El KV Cache es el recurso más caro y crítico en inferencia. Implementaremos una estrategia de Tiered KV Cache (LMCache) para expandir la capacidad más allá de la HBM de la GPU.

Jerarquía de Almacenamiento y Rendimiento

1. Tier 1: GPU HBM: Velocidad máxima, capacidad mínima.
2. Tier 2: CPU RAM: Capacidad media, latencia moderada.
3. Tier 3: Local SSD (vía emptyDir): Capacidad masiva.

Evidencia de Impacto: Según benchmarks en GKE, la implementación del Tier 3 (SSD Local) para contextos largos de 100k tokens permite:

* Una reducción del 79% en el TTFT, ya que evita la re-computación de prefijos.
* Un incremento del 264% en el Throughput de entrada.

Ruteo Prefix-Aware

El GKE Inference Gateway utiliza Prefix-aware routing para dirigir solicitudes con prompts similares a los mismos nodos. Esto maximiza el "hit ratio" del caché y es la diferencia entre un sistema que escala y uno que colapsa bajo carga.

7. Protocolo de Ejecución y Optimización de Presupuesto

Como arquitectos seniors, el manejo del presupuesto es una métrica de éxito. Este laboratorio ha sido diseñado para costar entre **$2 y 4 USD por sesión**, comparado con los >60 USD que costaría una implementación basada en TPUs v6e.

Checklist del Ciclo de Vida

1. Aprovisionamiento (15 min): terraform apply. Levantamos la VPC con MTU 8896 y los node pools de L4.
2. Orquestación: kubectl apply de los manifiestos de plataforma (KubeRay, kgateway).
3. Deploy & Test (30 min): Despliegue de Llama-3-8B y ejecución de pruebas de carga. Monitoreo obligatorio de la saturación de VRAM en Grafana.
4. Destrucción Inmediata: terraform destroy.

Mandato final: No persista recursos. La infraestructura debe ser tratada como efímera. La maestría técnica viene de la capacidad de levantar y destruir este stack completo en minutos, no de mantenerlo encendido.
