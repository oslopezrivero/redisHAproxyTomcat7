# Despliegue de Tomcat 7 con Redis y HAProxy utilizando Docker Compose

Este repositorio proporciona una guía y recursos para desplegar una aplicación web basada en Tomcat 7, utilizando Redis para el almacenamiento en caché y sesiones, y HAProxy como balanceador de carga para alta disponibilidad, todo orquestado con Docker Compose.

## Objetivos

- Implementar un entorno de alta disponibilidad para aplicaciones web usando Docker.
- Utilizar Redis para mejorar el rendimiento y escalabilidad mediante la gestión de sesiones y el almacenamiento en caché.
- Configurar HAProxy para la distribución de carga y failover utilizando el balanceo `leastconn`.
- Proporcionar una guía paso a paso para desplegar esta arquitectura con Docker Compose.
- Configurar un frontend de HAProxy para monitorear su funcionamiento.

## Arquitectura

La arquitectura del despliegue consta de los siguientes contenedores Docker:

1.  **Tomcat 7:** Uno o más contenedores con Tomcat 7 que ejecutan la aplicación web.
2.  **Redis:** Un contenedor con Redis para la gestión de sesiones y caché.
3.  **HAProxy:** Un contenedor con HAProxy para el balanceo de carga, incluyendo un frontend para estadísticas.

## Prerrequisitos

Antes de comenzar, asegúrate de tener lo siguiente:

-   **Docker instalado:** Docker y Docker Compose instalados en tu sistema.
-   **Conocimientos básicos:** Familiaridad con Docker, Docker Compose y conceptos de redes.
-   **Clon del repositorio:** Has clonado este repositorio en tu máquina.
-  **Imagen WAR de tu aplicación web:** Prepara el archivo WAR de tu aplicación web, listo para ser montado en el contenedor de Tomcat.
-   **Configuración:** Archivos de configuración necesarios como `haproxy.cfg`, `redis-data-cache.properties`, y `context.xml`.

## Guía de Despliegue Paso a Paso

### 1. Preparación de la Configuración

1.  **Configuración de HAProxy:** Crea o utiliza el archivo `haproxy.cfg` con la siguiente configuración:
    ```
    log stdout local0
    log stdout local1 notice
    stats socket /run/haproxy/admin.sock mode 666 level admin expose-fd listeners
    stats timeout 30s
    daemon

    defaults
      log global
      mode http
      option httplog
      option dontlognull
      timeout connect 5000
      timeout client 50000
      timeout server 50000
      timeout queue 10000
      timeout http-request 10000
      timeout http-keep-alive 10000
      retries 3

    frontend http-in
      bind *:80
      default_backend tomcat_servers

    frontend stats
      bind *:9999
      stats enable
      stats uri /haproxy_stats
      stats auth admin:password  # CAMBIAR ESTO PARA PRODUCCIÓN

    backend tomcat_servers
      balance leastconn
      server tomcat1 host1:8080 check inter 5000 fall 3 rise 2
      server tomcat2 hosts2:8080 check inter 5000 fall 3 rise 2
    ```
    *   **Importante:**  Reemplaza `host1` y `hosts2` con los nombres de host o IPs de tus servidores Tomcat.  **¡CAMBIA LA CONTRASEÑA DE `stats auth` EN PRODUCCIÓN!**
    *   **`balance leastconn`**: El balanceo de carga ahora se realiza utilizando el algoritmo `leastconn`.
    *   **Frontend `stats`**: Se habilita un frontend para visualizar las estadísticas de HAProxy, accesible en el puerto 9999.
    *   **Checks de Salud:**  Cada servidor Tomcat ahora tiene configurados checks de salud con parámetros para intervalos, fallos y recuperaciones.

2.  **Configuración de Sesión con Redis para Tomcat:**
    *  Crea los archivos necesarios para la gestión de sesiones de Redis en Tomcat. Por ejemplo:
        -   `redis-data-cache.properties`:
            ```
# - ${ENV_VARIABLE_NAME}

#- redis hosts. ex: 127.0.0.1:6379, 127.0.0.2:6379, 127.0.0.2:6380, ....
redis.hosts=redis-container:6379

#- redis password.
redis.password=Password

#- set true to enable redis cluster mode. (default value: false)
redis.cluster.enabled=false

#- set true to enable redis sentinel mode. (default value: false)
redis.sentinel.enabled=false
#- redis sentinel master name. (default value: mymaster)
redis.sentinel.master=mymaster

#- redis database. (default value: 0)
#- redis.database=0

#- redis connection timeout. (default value: 2000 ms)
#- redis.timeout=2000

#- enable redis and standard session mode. (default value: false)
#- If enabled,
#- Must be enabled sticky session in your load balancer configuration. Else this manager may not return the updated session values.
#- Session values are stored in local jvm and redis.
#- If redis is down/not responding, requests uses jvm stored session values to process user requests. Redis comes back the values will be synced.
lb.sticky-session.enabled=false

#- session persistent policies. (default value: DEFAULT) ex: DEFAULT, SAVE_ON_CHANGE
# policies - DEFAULT, SAVE_ON_CHANGE, ALWAYS_SAVE_AFTER_REQUEST
# SAVE_ON_CHANGE: every time session.setAttribute() or session.removeAttribute() is called the session will be saved.
# ALWAYS_SAVE_AFTER_REQUEST: force saving after every request, regardless of whether or not the manager has detected changes to the session.
session.persistent.policies=DEFAULT

#- single-sign-on session timeout. (default value: 0 ms (-infinite))
redis.sso.timeout=0
            ```
        -   `context.xml`:
            ```xml
            <Context>
    <!-- Default set of monitored resources -->
    <WatchedResource>WEB-INF/web.xml</WatchedResource>

    <!-- Configuración para utilizar Redis como administrador de sesiones -->
                <Valve className="tomcat.request.session.redis.SessionHandlerValve" />
                <Manager className="tomcat.request.session.redis.SessionManager" />
            </Context>
            ```
    *  Asegúrate que estos archivos serán montados en el contenedor Tomcat utilizando el `docker-compose.yml`.
3.  **Librerias de gestión de sesión:** Asegúrate de tener las librerias necesarias para que tomcat use redis para la gestión de sesión, entre ellas : `jedis-*.jar`, `tomcat-redis-session-manager-*.jar`, `commons-pool2-*.jar`, `slf4j-api-*.jar`.
### 2. Despliegue con Docker Compose

1.  **Copia el archivo `docker-compose.yml`:** Asegúrate de que el archivo `docker-compose.yml` esté en la raíz de tu repositorio.

2.  **Ejecuta Docker Compose:**
    ```bash
    docker-compose up -d
    ```
    Esto creará y ejecutará los contenedores en segundo plano.

3.  **Verifica los Contenedores:**
    ```bash
     docker ps
    ```
    Deberías ver los contenedores `redis`, `haproxy` y `tomcat-1` (o como los hayas nombrado) en ejecución.

### 3. Despliegue de la Aplicación Web

1.  **Montar el archivo WAR:** Debes asegúrate que el archivo WAR de la aplicación se encuentre en una ruta accesible para el Dockerfile del contenedor Tomcat. De tal manera que la aplicación sea copiada dentro de la carpeta webapps al momento de contruir la imagen.

### 4. Acceso a la Aplicación Web

*   Abre tu navegador web y accede a la dirección IP o dominio del servidor donde se está ejecutando Docker Compose, en el puerto 80 (por ejemplo, `http://localhost`).
*   HAProxy distribuirá el tráfico a los contenedores Tomcat utilizando el algoritmo `leastconn`.

### 5. Acceso a las Estadísticas de HAProxy

*   Abre tu navegador web y accede a la dirección IP o dominio del servidor donde se está ejecutando Docker Compose, en el puerto 9999, seguido de la ruta `/haproxy_stats` (por ejemplo, `http://localhost:9999/haproxy_stats`).
*   Ingresa el usuario `admin` y la contraseña `password` cuando se te solicite. **Recuerda cambiar estas credenciales en un entorno de producción.**

## Archivo `docker-compose.yml`

A continuación, se muestra el ejemplo de `docker-compose.yml` que proporcionaste (con algunas notas adicionales):

```yaml
version: "3.2"
services:
  redis:
    container_name: redis-tomcat
    hostname: redis
    image: redis:alpine
    ports:
      - "6379:6379"
    restart: always
    networks:
      default:
        aliases:
          - redis
  haproxy:
    container_name: haproxy
    hostname: haproxy
    image: haproxy:latest
    ports:
      - "80:80"
      - "9999:9999"
    restart: always
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
      - ./run/haproxy:/run/haproxy
      - ./dev/log:/dev/log
    networks:
      default:
        aliases:
          - haproxy
  app:
    container_name: tomcat-1
    hostname: app
    image: openjdk:7u91-jdk-alpine
    ports:
      - 8080:8080
    volumes:
      - ./redis-data-cache.properties:/opt/tomcat/conf/redis-data-cache.properties
      - ./context.xml:/opt/tomcat/conf/context.xml
      - ./commons-pool2-2.6.2.jar:/opt/tomcat/lib/commons-pool2-2.6.2.jar
      - ./jedis-3.0.1.jar:/opt/tomcat/lib/jedis-3.0.1.jar
      - ./slf4j-api-1.7.26.jar:/opt/tomcat/lib/slf4j-api-1.7.26.jar
      - ./tomcat-cluster-redis-session-manager-4.0.jar:/opt/tomcat/lib/tomcat-cluster-redis-session-manager-4.0.jar
networks:
    default:
        driver: bridge
