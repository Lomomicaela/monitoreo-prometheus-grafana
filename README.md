# Monitoreo con Prometheus y Grafana: explicación del `docker-compose.yml`

## Introducción

Este informe documenta el redespliegue del stack de monitoreo de la Etapa 6 y explica, servicio por servicio, el `docker-compose.yml` con el que se levanta.

En la práctica anterior (`crud-logs-analisis-cli`) un problema se investigaba después de ocurrido, filtrando archivos de log con `grep` y `awk`. En esta etapa el enfoque es el contrario: se mide el sistema mientras funciona, mediante **métricas**. Un log describe con detalle un evento puntual, pero hay que buscarlo y leerlo; una métrica es un número que cambia en el tiempo (peticiones por segundo, conexiones a la base, uso de CPU) y permite ver de un vistazo un pico o una caída.

El stack está formado por cinco servicios definidos en el compose:

- **`basedatos`**: el servidor MySQL.
- **`exportador`**: traduce las estadísticas internas de MySQL a un formato que Prometheus puede leer.
- **`medidor`**: cAdvisor, que mide el consumo de cada contenedor.
- **`recolector`**: Prometheus, que junta todas las métricas.
- **`tablero`**: Grafana, que las muestra en dashboards.

Además hay una pieza que **no** está en el compose: la aplicación Flask que se monitorea. Corre directamente en el host (puerto 8888) y publica sus métricas en `/metrics`. Esto condiciona la configuración de dos servicios, como se explica en las Partes 2 y 3.

El informe se organiza en dos bloques, más un anexo con las capturas de pantalla de todo el procedimiento (Figuras 1 a 22). Primero está el código completo para redesplegar el stack, en el orden en que se ejecuta. Después viene la explicación, dividida en tres partes: persistencia de datos (Parte 1), recolección de métricas (Parte 2) y visualización y topología (Parte 3), más una conclusión que las une.

### Nombres utilizados

A los servicios y archivos se les asignaron nombres propios. Esta tabla permite relacionarlos con los de la consigna:

| En la consigna | En este stack | Qué es |
|---|---|---|
| `mysql` | `basedatos` | Base de datos MySQL |
| `dbexporter` | `exportador` | Exporter de métricas de MySQL |
| `cadvisor` | `medidor` | cAdvisor, mide los contenedores |
| `prometheus` | `recolector` | Prometheus, junta las métricas |
| `grafana` | `tablero` | Grafana, los dashboards |
| `mynetwork` | `red_metricas` | La red de Docker |
| `my.cnf` | `credenciales.cnf` | Credenciales del exporter |
| `prometheus.yml` | `config-prometheus.yml` | Configuración de scraping |

La base de datos se llama `escuela` (su tabla sigue siendo `alumnos`, porque la app la consulta con ese nombre). Las contraseñas son `ClaveDb_2026` para MySQL y `GrafanaClave_26` para Grafana.

---

# Código completo

### 1. Instalar las herramientas

En este entorno el comando es `docker-compose` (con guion); el `docker compose` con espacio no está instalado.

```bash
apt-get update -qq
apt-get install -y -qq docker-compose python3-pip git
pip3 install flask mysql-connector-python prometheus-flask-exporter --break-system-packages --ignore-installed
```

### 2. Crear la carpeta y los tres archivos

```bash
mkdir -p /root/metricas
cd /root/metricas
```

`credenciales.cnf`:

```bash
cat > credenciales.cnf << 'EOF'
[client]
user=root
password=ClaveDb_2026
host=basedatos
port=3306
EOF
```

`config-prometheus.yml`:

```bash
cat > config-prometheus.yml << 'EOF'
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'recolector'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'contenedores'
    static_configs:
      - targets: ['medidor:8080']

  - job_name: 'base-de-datos'
    static_configs:
      - targets: ['exportador:9104']

  - job_name: 'app-alumnos'
    static_configs:
      - targets: ['host.docker.internal:8888']
EOF
```

`docker-compose.yml`:

```bash
cat > docker-compose.yml << 'EOF'
version: '3'
services:

  basedatos:
    image: mysql:latest
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ClaveDb_2026
    networks:
      - red_metricas

  exportador:
    image: prom/mysqld-exporter:latest
    restart: always
    volumes:
      - ./credenciales.cnf:/.my.cnf:ro
    command:
      - --config.my-cnf=/.my.cnf
    depends_on:
      - basedatos
    networks:
      - red_metricas

  medidor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    restart: always
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    networks:
      - red_metricas

  recolector:
    image: prom/prometheus:latest
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./config-prometheus.yml:/etc/prometheus/prometheus.yml
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      - red_metricas

  tablero:
    image: grafana/grafana:13.2.2
    restart: always
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=GrafanaClave_26
    depends_on:
      - recolector
    networks:
      - red_metricas

networks:
  red_metricas:
EOF
```

### 3. Levantar los contenedores

```bash
docker-compose up -d
docker-compose ps
```

### 4. Dejar corriendo la aplicación que se monitorea

La app Flask de la práctica no forma parte del compose: se clona, se adapta a los valores de este stack y se ejecuta en el host. Antes hay que esperar a MySQL y crear su base.

```bash
DB_ID=$(docker ps -qf "name=basedatos")

until docker exec $DB_ID mysqladmin ping -h 127.0.0.1 -uroot -pClaveDb_2026 --silent 2>/dev/null; do
  echo "Esperando MySQL..."; sleep 2
done

docker exec -i $DB_ID mysql -h 127.0.0.1 -uroot -pClaveDb_2026 << 'EOSQL'
CREATE DATABASE IF NOT EXISTS escuela;
USE escuela;
CREATE TABLE IF NOT EXISTS alumnos (
  id INT PRIMARY KEY AUTO_INCREMENT,
  nombre VARCHAR(50) NOT NULL,
  apellido VARCHAR(50) NOT NULL,
  fecha_nacimiento DATE NOT NULL
);
INSERT INTO alumnos (nombre, apellido, fecha_nacimiento) VALUES
  ('Lucia', 'Fernandez', '2000-04-12'),
  ('Tomas', 'Ruiz', '1999-09-30'),
  ('Camila', 'Sosa', '2001-02-18'),
  ('Bruno', 'Acosta', '1998-11-05'),
  ('Valentina', 'Ibarra', '2002-06-22');
EOSQL

cd /root
git clone --branch monitoring https://github.com/pablopedernera0/crud-python.git
cd crud-python

DB_IP=$(docker inspect $DB_ID --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}')
sed -i "s/172.18.0.2/$DB_IP/" app.py
sed -i 's/mysecretpassword/ClaveDb_2026/' app.py
sed -i 's/"database": "alumnos"/"database": "escuela"/' app.py

grep -A7 DB_CONFIG app.py

nohup python3 app.py > app.log 2>&1 &
```

### 5. Verificar que se miden las tres fuentes

```bash
curl -s http://127.0.0.1:8888/metrics | head -20
```

Luego, en el puerto 9090 (**Status → Target health**), los cuatro targets tienen que figurar en estado `UP`.

### 6. Configurar Grafana

Este paso se hace desde el navegador y tiene ocho partes. Los nombres de los menús corresponden a **Grafana 13.2.2**, la versión fijada en el compose; en otras versiones pueden estar en otro lugar. Las capturas de cada parte están en el Anexo.

#### 6.1 Entrar a Grafana

1. Abrí Grafana en el puerto 3000. En una máquina propia la dirección es `http://localhost:3000`. En Killercoda, hacé clic en el ícono de menú (≡) arriba a la derecha, elegí **Traffic / Ports**, escribí `3000` en **Custom Ports** y hacé clic en **Access** (Figuras 13 y 14).
2. Ingresá con usuario `admin` y contraseña `GrafanaClave_26` (Figura 15). Si Grafana propone cambiar la contraseña, se puede omitir.

#### 6.2 Conectar Grafana con Prometheus

1. En el menú lateral izquierdo, andá a **Connections → Data sources**.
2. Hacé clic en **Add data source** y elegí **Prometheus**.
3. En **Connection → Prometheus server URL** escribí:
   ```
   http://recolector:9090
   ```
   Se usa el nombre del servicio y no `localhost`, porque Grafana y Prometheus están en contenedores distintos.
4. Bajá hasta el final de la página y hacé clic en **Save & test**. Tiene que aparecer un mensaje de conexión exitosa (Figura 16).

#### 6.3 Crear el dashboard y abrir el editor de paneles

1. En el menú lateral, andá a **Dashboards**.
2. Arriba a la derecha, hacé clic en **New → New dashboard**.
3. Si Grafana pregunta por el diseño del tablero, elegí **Auto grid**.
4. Hacé clic en **Add visualization** (en algunas versiones es la tarjeta **Panel** del menú **Add** de la derecha). Si pide una fuente de datos, elegí **prometheus**.

Se abre el editor del panel (*Edit panel*), que tiene tres zonas: el gráfico, arriba a la izquierda; la sección **Queries**, abajo, donde se escribe la consulta; y las opciones a la derecha (**Suggestions** y **All visualizations**).

#### 6.4 Panel 1: "Tráfico de la app"

1. En la consulta **A**, en la fila donde dice **Builder | Code**, hacé clic en **Code**.
2. En la caja de texto que aparece, pegá:
   ```
   rate(flask_http_request_duration_seconds_count[1m])
   ```
3. Hacé clic en **Run queries** o apretá `Shift + Enter`.
4. Apagá el interruptor **Table view** (arriba a la izquierda) para ver el gráfico y no una tabla.
5. A la derecha, en **All visualizations**, elegí **Time series**.
6. En las opciones del panel, en el campo **Title**, escribí `Tráfico de la app`.
7. Arriba, configurá el rango de tiempo en **Last 5 minutes** y la actualización automática en **5s**.
8. Hacé clic en **Save** (arriba a la derecha), ponele un nombre al dashboard, por ejemplo `Monitoreo`, y confirmá (Figura 17).

> **Atención:** el buscador **Search for...** de la derecha sirve para buscar tipos de gráfico, no para escribir consultas. Si la consulta se pega ahí, Grafana responde "Could not find anything matching your query".

#### 6.5 Panel 2: "Conexiones a la base"

1. Con el dashboard abierto, hacé clic en **Edit** (arriba a la derecha).
2. Hacé clic en **Add → Visualization** (o en la tarjeta **Panel**).
3. Cambiá a **Code**, pegá la consulta y ejecutala con **Run queries**:
   ```
   mysql_global_status_threads_connected
   ```
4. Elegí **Time series**, poné el título `Conexiones a la base` y guardá con **Save** (Figura 18).

#### 6.6 Panel 3: "CPU de la base"

1. En la **terminal** (no en Grafana), obtené el ID del contenedor de MySQL:
   ```bash
   docker ps -qf "name=basedatos"
   ```
   En esta práctica devolvió `966d329d8cda` (Figura 19).
2. Agregá un panel nuevo como en 6.5, cambiá a **Code** y pegá la consulta reemplazando `ID_DEL_CONTENEDOR` por el ID obtenido:
   ```
   rate(container_cpu_usage_seconds_total{id=~".*966d329d8cda.*"}[1m])
   ```
   cAdvisor identifica a cada contenedor por el ID de su grupo de control (*cgroup*) y no por su nombre, por eso se filtra por ID con la expresión regular `=~`.
3. Ejecutá con **Run queries**, poné el título `CPU de la base` y guardá con **Save** (Figuras 20 y 21).

> **Atención:** solo el primer comando (`docker ps ...`) va en la terminal. La línea `rate(...)` se escribe únicamente en Grafana; si se pega en la terminal, bash responde con un error de sintaxis.

#### 6.7 Generar carga y observar el dashboard

Con el dashboard abierto, en **Last 5 minutes** y con actualización de **5s**, ejecutá en la terminal el comando del paso 7 (Figura 22). Los paneles de tráfico y de conexiones tienen que subir mientras corre la carga y volver a bajar cuando termina.

#### 6.8 Problemas frecuentes

| Qué se ve | Causa | Cómo se soluciona |
|---|---|---|
| "Could not find anything matching your query" | La consulta se pegó en el buscador de visualizaciones | Hacer clic en **Clear** y pegarla en **Queries → Code** |
| No aparece la caja para escribir la consulta | El editor está en modo **Builder** | Cambiar a **Code** |
| Se ve una tabla en lugar de un gráfico | **Table view** está activado | Apagar el interruptor |
| El panel de tráfico dice "No data" | La app no recibió peticiones en el último minuto | Generar carga (paso 7) |
| Todos los paneles dicen "No data" | Un target caído o la URL del data source incorrecta | Revisar **Status → Target health** en Prometheus y probar la URL `http://recolector:9090` con **Save & test** |
| `bash: syntax error near unexpected token` | Se pegó la consulta `rate(...)` en la terminal | Esa línea va en Grafana, no en la terminal |

### 7. Generar carga mirando el dashboard

```bash
seq 1 1500 | xargs -P 30 -I{} curl -s -o /dev/null http://127.0.0.1:8888/
```

---

