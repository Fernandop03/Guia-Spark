# Guia-Spark
-------------
# 🚀 Despliegue y Optimización de Cluster Spark en Docker Swarm

Este repositorio contiene la configuración y las instrucciones detalladas para desplegar un cluster de **Apache Spark 3.5.5** sobre **Docker Swarm**. El enfoque principal es la **optimización de la asignación de recursos** a los nodos worker de Spark, basándose en las especificaciones de hardware de cada máquina. Esto asegura la estabilidad del cluster, previene fallos por falta de recursos (OOM) y maximiza la utilización eficiente de tu infraestructura.

-----

### 1\. 💡 Filosofía de Arquitectura y Optimización

Un cluster Spark distribuido depende críticamente de la correcta asignación de recursos a sus componentes (Master y Workers). Una configuración inadecuada puede llevar a:

  * **Fallas de Nodos Worker:** Si un worker solicita más CPU o RAM de la que un nodo físico puede proporcionar (después de considerar los recursos para el sistema operativo y el propio Docker), el nodo puede volverse inestable, o el contenedor del worker se detendrá o reiniciará constantemente.
  * **Subutilización de Recursos:** Asignar menos recursos de los disponibles desaprovecha el potencial de tus nodos más potentes.

Para mitigar estos problemas, hemos implementado una estrategia de **Clasificación de Nodos (Node Tiers)**. Esto implica agrupar tus máquinas por capacidades de hardware similares (CPU y RAM) y definir **perfiles de recursos Spark explícitos** para cada grupo.

-----

### 2\. 📊 Clasificación de Nodos y Asignación de Recursos Spark

Hemos analizado detalladamente los # nodos de tu infraestructura. La categorización y la asignación de recursos se basan en un criterio conservador para dejar un margen de seguridad para el sistema operativo del host y el *overhead* de Docker (aproximadamente 1-2 núcleos de CPU y 2-4 GB de RAM).

| Categoría de Rendimiento | Nodos (Hostname)                                                                                                                                                                                                            | Procesador (Cores/Threads) | RAM (GB) | Rol Principal en Cluster | Recursos Spark Worker Asignados (`SPARK_WORKER_CORES`, `SPARK_WORKER_MEMORY`) |
| :----------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------- | :------- | :----------------------- | :----------------------------------------------------------------------------- |
| **Maestro (Master)** | `PC-YAGO`                                                                                                                                                                                                                   | i7-7700 (4C/8T)            | 16       | Spark Master, Jupyter    | N/A (No es worker de datos, se enfoca en orquestación e interfaz)              |
| **Muy Alto (Very High)** | `PC-BRUNO`                                                                                                                                                                                                                  | Ryzen 5 3600 (6C/12T)      | 16       | Spark Worker             | **CPU: 5**, **RAM: 14G** (6C - 1C; 16GB - 2GB)                                  |
| **Alto (High)** | `PC-JESUS`, `PC-JORGE`                                                                                                                                                                                                      | i5-3470 / i7-7700 (4C/8T)  | 16       | Spark Worker             | **CPU: 3**, **RAM: 14G** (4C - 1C; 16GB - 2GB)                                  |
| **Medio (Mid)** | `PC-MIGUEL`, `PC-DANIEL`, `PC-ANGEL`, `PC-ELIAS`, `PC-CESAR`, `PC-CHAVA`, `PC-ROBERTO`, `PC-DAVID`, `PC-ALEX`, `PC-PAOLA`, `PC-ADRIAN`, `PC-ISAI`, `PC-ERIK`, `PC-LUCAS`, `PC-ULISES`, `PC-CARLOS` (16 nodos) | i5-3470 (4C)               | 8        | Spark Worker             | **CPU: 3**, **RAM: 6G** (4C - 1C; 8GB - 2GB)                                   |

-----

### 3\. ⚙️ Archivos de Configuración Esenciales

Estos archivos son la base de tu despliegue y deben residir en el directorio raíz de este repositorio.

#### 3.1. `node_tiers.env`

Este archivo define los **nombres de host** para cada categoría de rendimiento de nodos. Es **absolutamente crucial** que los nombres de host aquí listados coincidan **exactamente** con los que Docker Swarm reporta para tus máquinas. Puedes verificar los nombres de host en tu Swarm con `docker node ls`.

```bash
# Configuration for node performance tiers
# Add or remove hostnames from these lists as needed.
# Ensure hostnames match exactly what 'docker node ls' reports.

# Master node hostname
# IMPORTANT: Verify this matches the hostname reported by 'docker node ls' for your master node.
MASTER_HOSTNAME_CONFIG="PC-YAGO"

# Worker Node Performance Tiers

# VERY_HIGH_PERF_HOSTNAMES: Nodos con Ryzen 5 3600 y 16GB RAM
VERY_HIGH_PERF_HOSTNAMES=("PC-BRUNO")

# HIGH_PERF_HOSTNAMES: Nodos con i5-3470/i7-7700 y 16GB RAM
HIGH_PERF_HOSTNAMES=("PC-JESUS" "PC-JORGE")

# MID_PERF_HOSTNAMES: Nodos con i5-3470 y 8GB RAM (la mayoría)
MID_PERF_HOSTNAMES=("PC-MIGUEL" "PC-DANIEL" "PC-ANGEL" "PC-ELIAS" "PC-CESAR" "PC-CHAVA" "PC-ROBERTO" "PC-DAVID" "PC-ALEX" "PC-PAOLA" "PC-ADRIAN" "PC-ISAI" "PC-ERIK" "PC-LUCAS" "PC-ULISES" "PC-CARLOS")

# LOW_PERF_HOSTNAMES: Not used in this configuration.
LOW_PERF_HOSTNAMES=()
```

#### 3.2. `setup_swarm.sh`

Este script es responsable de preparar tu Docker Swarm. Crea la red *overlay* necesaria para la comunicación entre contenedores y, fundamentalmente, **aplica las etiquetas `spark-role` y `performance`** a cada uno de tus nodos. Estas etiquetas son la base para que Docker Swarm pueda desplegar los servicios Spark Worker en los nodos con las capacidades correctas.

```bash
#!/bin/bash

# Exit immediately if a command exits with a non-zero status.
set -e

# --- General Configuration ---
# Name of the overlay network that Spark services will use.
# Must match the network name defined in 'docker-compose.yml'.
NETWORK_NAME="hadoop_spark_net"

# --- Node Tier Configuration ---
# This file contains the hostname definitions for each performance tier.
CONFIG_FILE="node_tiers.env"

# Verify if the configuration file exists.
if [[ -f "$CONFIG_FILE" ]]; then
    echo "Loading node tier configuration from $CONFIG_FILE..."
    # 'source' loads variables defined in the file into the current script's environment.
    source "$CONFIG_FILE"
else
    echo "ERROR: Configuration file '$CONFIG_FILE' not found."
    echo "Please ensure '$CONFIG_FILE' exists and contains the necessary definitions."
    exit 1
fi

# Validate that the master node's primary variable is defined.
: "${MASTER_HOSTNAME_CONFIG:?ERROR: MASTER_HOSTNAME_CONFIG is not defined in $CONFIG_FILE. Please define it.}"

echo "----------------------------------------------------"
echo "Preparing Docker Swarm infrastructure..."
echo "----------------------------------------------------"

# 1. Create or Verify the Overlay Network
echo "Checking/Creating overlay network '$NETWORK_NAME'..."
if ! docker network ls | grep -q "$NETWORK_NAME"; then
    docker network create -d overlay --attachable "$NETWORK_NAME"
    echo "  -> Network '$NETWORK_NAME' created successfully."
else
    echo "  -> Network '$NETWORK_NAME' already exists. Continuing."
fi
echo "----------------------------------------------------"

# 2. Labeling Swarm Nodes
echo "Starting node labeling process for the Swarm..."
# Iterate over each node in the Swarm to apply the correct labels.
docker node ls --format "{{.ID}}\t{{.Hostname}}" | while read NODE_ID NODE_HOSTNAME; do
    echo "  Processing node: '$NODE_HOSTNAME' (ID: '$NODE_ID')"

    # Clean up previous Spark role and performance labels to ensure a clean configuration.
    echo "    -> Clearing previous Spark and performance labels (if they exist)..."
    docker node update --label-rm spark-role --label-rm performance "$NODE_ID" 2>/dev/null || true # '2>/dev/null || true' to ignore errors if the label doesn't exist.

    SPARK_ROLE_LABEL=""       # Variable to store the Spark role (master/worker)
    PERFORMANCE_LABEL=""      # Variable to store the performance level (very-high/high/mid/low)

    # Assign master role to the node configured as MASTER_HOSTNAME_CONFIG.
    if [[ "$NODE_HOSTNAME" == "$MASTER_HOSTNAME_CONFIG" ]]; then
        SPARK_ROLE_LABEL="spark-role=master"
        echo "    -> Assigning role: '$SPARK_ROLE_LABEL'"
        docker node update --label-add "$SPARK_ROLE_LABEL" "$NODE_ID"
    fi

    # Assign worker role and performance level to the other nodes.
    # The master node will NOT be labeled as a worker in this configuration to dedicate its resources to control.
    if [[ "$NODE_HOSTNAME" != "$MASTER_HOSTNAME_CONFIG" ]]; then
        SPARK_ROLE_LABEL="spark-role=worker"
        echo "    -> Assigning role: '$SPARK_ROLE_LABEL'"
        docker node update --label-add "$SPARK_ROLE_LABEL" "$NODE_ID"

        # Search and assign the performance level (searched in order from most to least powerful).
        # Check Very High Performance list
        for hostname_in_list in "${VERY_HIGH_PERF_HOSTNAMES[@]}"; do
            if [[ "$hostname_in_list" == "$NODE_HOSTNAME" ]]; then
                PERFORMANCE_LABEL="performance=very-high"
                break
            fi
        done

        # Check High Performance list
        if [[ -z "$PERFORMANCE_LABEL" ]]; then # Only if a performance label hasn't been found yet
            for hostname_in_list in "${HIGH_PERF_HOSTNAMES[@]}"; do
                if [[ "$hostname_in_list" == "$NODE_HOSTNAME" ]]; then
                    PERFORMANCE_LABEL="performance=high"
                    break
                fi
            done
        fi

        # Check Mid Performance list
        if [[ -z "$PERFORMANCE_LABEL" ]]; then
            for hostname_in_list in "${MID_PERF_HOSTNAMES[@]}"; do
                if [[ "$hostname_in_list" == "$NODE_HOSTNAME" ]]; then
                    PERFORMANCE_LABEL="performance=mid"
                    break
                fi
            done
        fi

        # Check Low Performance list
        if [[ -z "$PERFORMANCE_LABEL" ]]; then
            for hostname_in_list in "${LOW_PERF_HOSTNAMES[@]}"; do
                if [[ "$hostname_in_list" == "$NODE_HOSTNAME" ]]; then
                    PERFORMANCE_LABEL="performance=low"
                    break
                fi
            done
        fi

        # Apply the performance label if one was found.
        if [[ -n "$PERFORMANCE_LABEL" ]]; then
            docker node update --label-add "$PERFORMANCE_LABEL" "$NODE_ID"
            echo "    -> Assigned performance level: '$PERFORMANCE_LABEL'"
        else
            echo "    -> WARNING: No specific performance level defined for hostname '$NODE_HOSTNAME'."
            echo "       This node will not be used by Spark Worker services that require a performance label."
        fi
    fi
    echo "  ----------------------------------------------------"
done

echo ""
echo "Node labeling process completed."
echo "----------------------------------------------------"
echo "VERIFICATION:"
echo "To confirm that labels were applied correctly, run:"
echo "  docker node ls --format 'table {{.Hostname}}\t{{.Status}}\t{{.Labels}}'"
echo "Or for a specific node:"
echo "  docker node inspect <node_id_or_hostname> | grep -A 5 Labels"
echo "----------------------------------------------------"
echo "Script 'setup_swarm.sh' finished!"
```

#### 3.3. `docker-compose.yml`

Este archivo define todos los servicios Docker que formarán tu cluster Spark. Se incluye un nuevo servicio para los workers de "Muy Alto Rendimiento" y se ajustan las asignaciones de CPU (`SPARK_WORKER_CORES`), RAM (`SPARK_WORKER_MEMORY`) y el número de `replicas` para cada tipo de worker según la clasificación anterior.

```yaml
version: '3.8'

services:
  spark-master:
    image: bitnami/spark:3.5.5
    hostname: spark-master
    user: root # Run as root to avoid permission issues in some environments.
    environment:
      - SPARK_MODE=master
      - SPARK_MASTER_HOST=spark-master
      - SPARK_LOCAL_IP=0.0.0.0
      - SPARK_DRIVER_HOST=spark-master
      - SPARK_DRIVER_BIND_ADDRESS=0.0.0.0
      - SPARK_USER=root
      - SPARK_RPC_AUTHENTICATION_ENABLED=no # Simplified for test/dev environment.
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
    ports:
      - "8080:8080" # Spark Master UI
      - "7077:7077" # Spark Master RPC Port
    deploy:
      mode: replicated
      replicas: 1 # A single instance of the Master.
      placement:
        constraints:
          - node.labels.spark-role == master # Deploys on the node labeled as master.
    networks:
      - hadoop_spark_net # Overlay network for inter-service communication.

  # Service for VERY HIGH PERFORMANCE Nodes (e.g., PC-BRUNO)
  spark-worker-very-high:
    image: bitnami/spark:3.5.5
    hostname: "spark-worker-vhigh-{{.Node.Hostname}}" # Dynamic hostname for identification.
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077 # Points to the Spark master.
      - SPARK_WORKER_WEBUI_PORT=8081 # Worker UI port.
      - SPARK_WORKER_CORES=5 # CPU allocation: 5 cores for Spark.
      - SPARK_WORKER_MEMORY=14G # RAM allocation: 14 GB for Spark.
    ports:
      - "8081:8081" # Exposes the worker UI port.
    deploy:
      mode: replicated
      replicas: 1 # There is 1 Very High Performance node.
      placement:
        constraints:
          - node.labels.spark-role == worker
          - node.labels.performance == very-high # Deploys on nodes with this label.
    networks:
      - hadoop_spark_net

  # Service for HIGH PERFORMANCE Nodes (e.g., PC-JESUS, PC-JORGE)
  spark-worker-high:
    image: bitnami/spark:3.5.5
    hostname: "spark-worker-high-{{.Node.Hostname}}"
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_WEBUI_PORT=8083 # Different port to avoid conflicts with other worker types if they were on the same host (though this setup avoids it).
      - SPARK_WORKER_CORES=3 # CPU allocation: 3 cores for Spark.
      - SPARK_WORKER_MEMORY=14G # RAM allocation: 14 GB for Spark.
    ports:
      - "8083:8083"
    deploy:
      mode: replicated
      replicas: 2 # There are 2 High Performance nodes.
      placement:
        constraints:
          - node.labels.spark-role == worker
          - node.labels.performance == high
    networks:
      - hadoop_spark_net

  # Service for MID PERFORMANCE Nodes (the majority of your nodes)
  spark-worker-mid:
    image: bitnami/spark:3.5.5
    hostname: "spark-worker-mid-{{.Node.Hostname}}"
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_WEBUI_PORT=8084 # Another different port.
      - SPARK_WORKER_CORES=3 # CPU allocation: 3 cores for Spark.
      - SPARK_WORKER_MEMORY=6G # RAM allocation: 6 GB for Spark.
    ports:
      - "8084:8084"
    deploy:
      mode: replicated
      replicas: 16 # There are 16 Mid Performance nodes.
      placement:
        constraints:
          - node.labels.spark-role == worker
          - node.labels.performance == mid
    networks:
      - hadoop_spark_net

  jupyter-notebook:
    image: jupyter/pyspark-notebook:latest
    hostname: jupyter-notebook
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - SPARK_HOME=/usr/local/spark
      - JUPYTER_TOKEN=sparkcluster # Jupyter Lab access token (!!!CHANGE IN PRODUCTION!!!)
    ports:
      - "8888:8888" # Jupyter Lab access port.
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.labels.spark-role == master # Deploys on the master node.
    networks:
      - hadoop_spark_net

networks:
  hadoop_spark_net:
    external: true # The network must be created externally by the setup_swarm.sh script.
```

#### 3.4. `dhcp-compose.yml`

Este archivo contiene el servicio DHCP. Es crucial levantarlo **antes** que cualquier otro servicio del cluster para asegurar que los nodos reciban sus IPs correctamente en tu red aislada (si aplica).

```yaml
# Before bringing up this service, ensure you assign a static IP to your network interface connected to the switch.
#
# 1. Verify the connection name with:
#    nmcli connection show
#
#    Example output:
#    NAME                UUID                                  TYPE      DEVICE
#    Wired connection 1  1c2b8...                              ethernet  enx68e43b307d75
#
# 2. Assign a manual IP by running (adjust the connection name if necessary):
#    sudo nmcli connection modify "Wired connection 1" \
#      ipv4.method manual \
#      ipv4.addresses 192.168.0.2/24 \
#      ipv4.gateway 192.168.0.1 \
#      ipv4.dns 8.8.8.8
#
# 3. Restart the connection to apply changes:
#    sudo nmcli connection down "Wired connection 1"
#    sudo nmcli connection up "Wired connection 1"
#
# Once done, you can bring up this DHCP service with Docker Compose.


version: "3.8"

services:
  dhcp:
    image: jpillora/dnsmasq
    container_name: dhcp
    restart: unless-stopped
    network_mode: "host"
    cap_add:
      - NET_ADMIN
      - NET_RAW
    command:
      - "-k"
      - "--port=0"
      - "--log-facility=-"
      - "--interface=enx68e43b..." # !!!IMPORTANT!!! Change this to your actual network interface name (e.g., eth0, enp0s3, enx68e43b307d75)
      - "--dhcp-range=192.168.0.100,192.168.0.200,1h" # IP range to assign and lease time.
      - "--dhcp-option=option:router,192.168.0.1" # Gateway (your router).
      - "--dhcp-option=option:dns-server,8.8.8.8,8.8.4.4" # DNS servers.
```

-----

### 4\. 🚀 Flujo de Despliegue Detallado

Sigue estos pasos en el orden indicado para un despliegue exitoso de tu cluster Spark.

#### 4.1. **Pre-despliegue (En cada nodo: Maestro y Workers)**

1.  **Instalar Docker y Docker Compose:**
      * Asegúrate de que Docker Engine y Docker Compose estén instalados y configurados en **todos tus 20 nodos**. Consulta la documentación oficial de Docker para tu distribución de Linux.
2.  **Configurar Carpetas de Datos (En cada nodo Worker):**
      * Para asegurar que los workers Spark tengan espacio para datos temporales o *shuffle*, crea una carpeta con permisos adecuados en **cada nodo worker** (todos excepto `PC-YAGO`):
        ```bash
        sudo mkdir -p /data/spark
        sudo chmod 777 /data/spark # Permisos de escritura para el contenedor
        ```

#### 4.2. **Configuración y Arranque del Servicio DHCP (Solo en el Nodo Maestro: `PC-YAGO`)**

Este paso es **fundamental** para la asignación de IPs en tu red. El servicio DHCP debe estar operativo antes de que los nodos worker se unan o intenten obtener IPs.

1.  **Asignar IP Estática al Maestro:**
      * Conéctate a tu nodo maestro (`PC-YAGO`).
      * **Identifica tu Interfaz de Red:** Ejecuta `nmcli connection show` y anota el nombre de tu conexión de red cableada (ej., `Wired connection 1`, `enp0s3`, `eth0`).
      * **Configura la IP Estática:** Sustituye `YOUR_NETWORK_INTERFACE_NAME` por el nombre real de tu interfaz y ajusta la IP, gateway y DNS según tu configuración de red.
        ```bash
        sudo nmcli connection modify "YOUR_NETWORK_INTERFACE_NAME" \
          ipv4.method manual \
          ipv4.addresses 192.168.0.2/24 \
          ipv4.gateway 192.168.0.1 \
          ipv4.dns "8.8.8.8,1.1.1.1"
        ```
      * **Reinicia la Conexión:**
        ```bash
        sudo nmcli connection down "YOUR_NETWORK_INTERFACE_NAME"
        sudo nmcli connection up "YOUR_NETWORK_INTERFACE_NAME"
        ```
      * **Verifica la IP:** Confirma que la interfaz tiene la IP estática asignada:
        ```bash
        ip a show YOUR_NETWORK_INTERFACE_NAME
        ```
2.  **Preparar y Ejecutar `dhcp-compose.yml`:**
      * **¡Edita `dhcp-compose.yml`\!** Cambia la línea `--interface=enx68e43b...` para usar el nombre real de tu interfaz de red en el nodo maestro.
      * Guarda el archivo `dhcp-compose.yml` en tu nodo maestro.
      * Levanta el servicio DHCP:
        ```bash
        docker compose -f dhcp-compose.yml up -d
        ```
      * **Verificación:** Asegúrate de que el contenedor `dhcp` esté en estado `Up`:
        ```bash
        docker ps
        ```

#### 4.3. **Inicialización y Configuración de Docker Swarm (Desde el Nodo Maestro)**

Todos los siguientes comandos se ejecutan desde el nodo maestro (`PC-YAGO`).

1.  **Inicializar Swarm (si no está inicializado):**

      * Si tu Swarm no está activo, inicialízalo:
        ```bash
        docker swarm init --advertise-addr <IP_ESTATICA_DEL_MAESTRO>
        ```
      * **¡Anote el token\!** Este comando le proporcionará un token de unión. Lo necesitará para el siguiente paso.

2.  **Unir Nodos Workers al Swarm (En cada nodo Worker):**

      * Conéctate a **cada uno de tus 19 nodos worker** (PC-MIGUEL, PC-DANIEL, etc.).
      * Ejecuta el comando `docker swarm join` que obtuviste del maestro:
        ```bash
        docker swarm join --token <TOKEN_DEL_MAESTRO> <IP_ESTATICA_DEL_MAESTRO>:2377
        ```

3.  **Verificar Nodos en el Swarm (Desde el Maestro):**

      * Asegúrate de que todos los 20 nodos estén listados y en estado `Ready`:
        ```bash
        docker node ls
        ```

4.  **Copiar y Ejecutar `setup_swarm.sh`:**

      * Copia los archivos **actualizados** `node_tiers.env` y `setup_swarm.sh` a la misma carpeta en el nodo maestro.
      * Haz el script ejecutable y lánzalo:
        ```bash
        chmod +x setup_swarm.sh
        ./setup_swarm.sh
        ```
      * **Verificar Etiquetas (¡MUY IMPORTANTE\!):** Después de que el script termine, valida que las etiquetas `spark-role` y `performance` se aplicaron correctamente:
        ```bash
        docker node ls --format 'table {{.Hostname}}\t{{.Status}}\t{{.Labels}}'
        ```
        `PC-YAGO` debe tener `spark-role=master`. Los otros nodos deben tener `spark-role=worker` y su respectiva etiqueta de rendimiento (`performance=very-high`, `performance=high`, o `performance=mid`).

#### 4.4. **Despliegue del Cluster Spark (Desde el Nodo Maestro)**

1.  **Copia el `docker-compose.yml`:**

      * Asegúrate de tener la **versión más reciente** del `docker-compose.yml` en la misma carpeta desde donde ejecutarás el comando.

2.  **Desplegar el Stack de Spark:**

    ```bash
    docker stack deploy -c docker-compose.yml spark-cluster
    ```

    Puedes elegir un nombre diferente para tu stack si lo deseas (ej., `my-spark-app`).

3.  **Verificar Servicios Desplegados:**

      * Confirma que todos los servicios se han desplegado con el número de réplicas esperado:
        ```bash
        docker service ls
        ```
        Deberías ver 1 réplica para `spark-master` y `jupyter-notebook`, 1 para `spark-worker-very-high`, 2 para `spark-worker-high` y 16 para `spark-worker-mid`.

4.  **Monitorear Logs de los Workers:**

      * Si algún worker no se inicia o falla, revisa sus logs para diagnosticar el problema:
        ```bash
        docker service logs spark-cluster_spark-worker-mid --follow # Ejemplo para workers mid
        # O para otros tipos:
        # docker service logs spark-cluster_spark-worker-high --follow
        # docker service logs spark-cluster_spark-worker-very-high --follow
        ```

#### 4.5. **Acceso a las Interfaces Web**

Una vez que todos los servicios estén `Running`, puedes acceder a las UIs desde tu navegador:

  * **Spark Master UI:** `http://<IP_ESTATICA_DEL_MAESTRO>:8080`
  * **Jupyter Lab:** `http://<IP_ESTATICA_DEL_MAESTRO>:8888/lab` (usa el token `sparkcluster`. **¡CAMBIAR EN PRODUCCIÓN POR SEGURIDAD\!**)

-----

### 5\. 🛠️ Previsiones, Resolución de Problemas y Mantenimiento

Esta sección te equipa con el conocimiento para diagnosticar y resolver los problemas más comunes, y mantener tu cluster saludable.

#### 5.1. **Problemas Comunes y Diagnóstico**

  * **Problema: Un nodo no se etiqueta correctamente o tiene etiquetas duplicadas.**

      * **Síntoma:** Los servicios Spark Worker no se despliegan en los nodos correctos, o `docker service ls` muestra menos réplicas de las esperadas.
      * **Diagnóstico:**
          * Verifica las etiquetas de tus nodos: `docker node ls --format 'table {{.Hostname}}\t{{.Status}}\t{{.Labels}}'`
          * Confirma que el `hostname` del nodo en `node_tiers.env` coincide **exactamente** con el que `docker node ls` reporta.
      * **Solución:** Si hay etiquetas incorrectas, elimínalas manualmente (`docker node update --label-rm <nombre_etiqueta> <node_id_o_hostname>`) y luego vuelve a ejecutar `./setup_swarm.sh`.

  * **Problema: Un worker Spark no se inicia o "muere" (el contenedor se reinicia constantemente).**

      * **Síntoma:** `docker ps` muestra el contenedor en `Exited` o `restarting`. `docker service ls` muestra `0/N` réplicas o un número menor al esperado.
      * **Diagnóstico (¡CRÍTICO\!):** **Revisa los logs del servicio afectado.** Usa `docker service logs spark-cluster_<nombre_del_servicio_worker> --follow`.
          * **Errores de Memoria (OOM):** Si ves `OutOfMemoryError`, "JVM exited" o "heap space", `SPARK_WORKER_MEMORY` es demasiado alto para la RAM disponible del nodo.
          * **Problemas de Conectividad:** Si hay mensajes como "failed to connect to master" o "connection refused", revisa la red `hadoop_spark_net` y la accesibilidad del puerto `7077` en el maestro.
      * **Solución:**
        1.  **Ajuste de Recursos:** Si es un problema de memoria/CPU, **reduce** los valores de `SPARK_WORKER_CORES` o `SPARK_WORKER_MEMORY` en el `docker-compose.yml` para el tipo de worker afectado. Considera dejar un margen más grande para el SO y Docker (ej., 3-4GB de RAM).
        2.  **Re-desplegar:** Después de cualquier cambio en `docker-compose.yml`, siempre actualiza el stack: `docker stack deploy -c docker-compose.yml spark-cluster`.

  * **Problema: Un nodo físico se cae o se desconecta del Swarm.**

      * **Previsión:** Docker Swarm es inherentemente resiliente. Intentará automáticamente reprogramar las réplicas de los servicios de ese nodo en otros nodos disponibles que cumplan con los mismos `placement constraints` (etiquetas).
      * **Monitoreo:** `docker service ls` mostrará el número actual de réplicas (ej., `1/2` si una réplica de dos falló). Una vez que el nodo se recupere y se una al Swarm, las réplicas se balancearán de nuevo.

  * **Problema: Discrepancias en el conteo de nodos (más o menos nodos de los esperados con workers).**

      * **Diagnóstico:** Compara el número de hostnames en cada lista de `node_tiers.env` con el número de `replicas` configurado en `docker-compose.yml` y con la salida real de `docker node ls`.
      * **Solución:**
          * **Nodo Extra:** Si tienes un nodo que no debería ser worker Spark, pero lo está siendo, asegúrate de que **no esté** en ninguna de las listas `_PERF_HOSTNAMES` en `node_tiers.env`. Si ya tiene etiquetas, elimínalas manualmente (`docker node update --label-rm spark-role --label-rm performance <node_id>`) y ejecuta `docker stack deploy` de nuevo.
          * **Nodo Faltante:** Si un nodo esperado no tiene workers o no está etiquetado, verifica su estado (`docker node ls`, `docker node inspect <hostname>`). Asegúrate de que esté encendido, unido al Swarm y que su hostname esté correctamente listado en `node_tiers.env`.

#### 5.2. **Mejores Prácticas y Consideraciones Futuras**

  * **Actualizaciones:** Para actualizar las imágenes de Spark o Jupyter, solo necesitas modificar el tag de la imagen en `docker-compose.yml` y ejecutar `docker stack deploy`. Docker Swarm gestionará la actualización gradual.
  * **Persistencia:** Para cargas de trabajo que requieren persistencia de datos (aunque Spark procesa datos transitorios en RAM), considera el uso de volúmenes persistentes.
  * **Seguridad:** Este setup está simplificado para un ambiente de desarrollo/prueba. Para producción, es **IMPRESCINDIBLE** configurar la autenticación, encriptación y SSL para Spark y Jupyter, así como implementar reglas de firewall robustas.
  * **Escalabilidad:** Si necesitas escalar, simplemente agrega más nodos con recursos similares a un tier existente, actualiza `node_tiers.env`, ejecuta `setup_swarm.sh` y luego incrementa las `replicas` correspondientes en `docker-compose.yml` antes de desplegar de nuevo.

-----

¡Con esta guía, tienes una base sólida y optimizada para operar tu cluster Spark \!
