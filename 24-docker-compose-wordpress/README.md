[← Volver al laboratorio](../README.md)

# 24 · Despliegue de WordPress con Docker Compose

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker%20Compose-v2-f472b6?style=flat-square&logo=docker&logoColor=white)
![WordPress](https://img.shields.io/badge/WordPress-MySQL-60a5fa?style=flat-square&logo=wordpress&logoColor=white)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Despliegue de un sitio **WordPress** con su base de datos **MySQL** usando **Docker Compose**, que define ambos contenedores, su red y sus volúmenes en un único archivo y los levanta con un solo comando.

## Objetivo

Desplegar una aplicación de varios contenedores (aplicación web y base de datos) de forma declarativa y reproducible, con almacenamiento persistente para que los datos sobrevivan a la recreación de los contenedores.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Requisito previo | Docker Engine instalado ([práctica 22](../22-docker-nginx)) |
| Herramienta | Docker Compose v2.27.0 |
| Directorio del proyecto | `~/wordpress-compose` |
| Contenedores | `wordpress_db` (MySQL 5.7) y `wordpress_app` (WordPress) |
| Puerto publicado | `8888` (host) → `80` (contenedor de WordPress) |
| Volúmenes | `db_data` (base de datos) y `wp_data` (archivos de WordPress) |

> [!NOTE]
> **Qué aporta Compose.** En lugar de escribir un `docker run` largo por cada contenedor, un archivo `docker-compose.yml` describe todos los servicios de la aplicación. Compose crea además una **red interna** donde los contenedores se resuelven por el nombre de su servicio: por eso WordPress puede llegar a la base de datos con la dirección `db:3306`, sin conocer ninguna IP.

## Índice

- [Parte A: Instalar Docker Compose](#parte-a-instalar-docker-compose)
- [Parte B: Crear el proyecto](#parte-b-crear-el-proyecto)
- [Parte C: El archivo docker-compose.yml](#parte-c-el-archivo-docker-composeyml)
- [Parte D: Desplegar los contenedores](#parte-d-desplegar-los-contenedores)
- [Parte E: Acceder a WordPress](#parte-e-acceder-a-wordpress)
- [Administración del proyecto (opcional)](#administración-del-proyecto-opcional)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Instalar Docker Compose

**1.** Descargamos el binario de Docker Compose v2.27.0 desde su repositorio de GitHub y le damos permisos de ejecución:

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

**2.** Verificamos la instalación:

```bash
docker-compose version
```

> [!NOTE]
> **Hay dos formas de usar Compose, y esta práctica usa ambas.** El binario que se acaba de instalar se invoca con guion (`docker-compose ...`). El **complemento** `docker-compose-plugin`, que se instaló junto con Docker en la práctica 22, se invoca como subcomando (`docker compose ...`, sin guion). Ambos son Compose v2 y leen el mismo archivo. Si el complemento ya está instalado, este paso de descarga manual **no es necesario**: bastaría con `docker compose version`. Más adelante, el despliegue se ejecuta con la forma sin guion.

> [!TIP]
> Al descargar un binario desde Internet y ejecutarlo con privilegios, conviene comprobar su **suma de verificación** (SHA-256) contra la que publica el proyecto junto a cada versión, para asegurarse de que el archivo no fue alterado ni quedó incompleto.

---

## Parte B: Crear el proyecto

**1.** Creamos el directorio del proyecto y el archivo de definición:

```bash
mkdir ~/wordpress-compose
cd ~/wordpress-compose
sudo nano docker-compose.yml
```

---

## Parte C: El archivo docker-compose.yml

**1.** Contenido del archivo:

```yaml
services:
  db:
    image: mysql:5.7
    container_name: wordpress_db
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp_user
      MYSQL_PASSWORD: CAMBIAR_CLAVE_WP
      MYSQL_ROOT_PASSWORD: CAMBIAR_CLAVE_ROOT
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    container_name: wordpress_app
    restart: always
    ports:
      - "8888:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: CAMBIAR_CLAVE_WP
    depends_on:
      - db
    volumes:
      - wp_data:/var/www/html

volumes:
  db_data:
  wp_data:
```

> [!IMPORTANT]
> **Las contraseñas se sustituyeron por marcadores.** En el laboratorio se usaron claves de ejemplo muy simples, y se reemplazaron aquí por `CAMBIAR_CLAVE_WP` y `CAMBIAR_CLAVE_ROOT` para no publicar credenciales, aunque sean de prueba. Hay que escribir valores propios. **`MYSQL_PASSWORD` (servicio `db`) y `WORDPRESS_DB_PASSWORD` (servicio `wordpress`) deben ser exactamente el mismo valor**, porque son la misma credencial vista desde los dos lados: la que crea el usuario en MySQL y la que WordPress usa para conectarse.

**2.** Qué define cada parte:

| Elemento | Significado |
|---|---|
| `services` | Los contenedores que forman la aplicación |
| `image` | Imagen de Docker Hub a utilizar |
| `container_name` | Nombre fijo del contenedor (si no, Compose genera uno a partir del proyecto) |
| `restart: always` | Reinicia el contenedor si se detiene o si se reinicia el servidor |
| `environment` | Variables de entorno con las que se configura cada imagen |
| `ports: "8888:80"` | Publica el puerto 80 del contenedor en el 8888 del servidor |
| `depends_on: db` | Inicia primero la base de datos y después WordPress |
| `volumes` (en cada servicio) | Monta un volumen con nombre en una ruta del contenedor |
| `volumes` (al final) | Declara los volúmenes con nombre `db_data` y `wp_data` |

> [!NOTE]
> **El servicio `db` no publica ningún puerto**, y eso es lo correcto: la base de datos solo es accesible desde la red interna de Compose, es decir, desde el contenedor de WordPress. Nada fuera del servidor puede conectarse directamente a MySQL.

> [!WARNING]
> **`mysql:5.7` ya no recibe actualizaciones de seguridad.** MySQL 5.7 llegó al final de su vida útil en octubre de 2023. Es adecuado para un laboratorio, pero en cualquier entorno real conviene una versión con soporte vigente, como `mysql:8.0` o `mariadb`. Además, `wordpress:latest` cambia con el tiempo: para un despliegue reproducible, conviene fijar una versión concreta de la imagen.

> [!TIP]
> Las contraseñas escritas directamente en el archivo quedan visibles para cualquiera que pueda leerlo, y se subirían a un repositorio si el archivo se versiona. La práctica habitual es sacarlas a un archivo `.env` (que se excluye del repositorio) y referenciarlas como `${MYSQL_PASSWORD}`, o usar *secrets* de Docker.

---

## Parte D: Desplegar los contenedores

**1.** Con el archivo guardado, levantamos todo el proyecto:

```bash
sudo docker compose up -d
```

`up` crea (si hace falta) la red, los volúmenes y los contenedores, y los inicia. `-d` los deja en segundo plano. En la primera ejecución, Docker descarga las imágenes de MySQL y WordPress, por lo que puede tardar.

**2.** Comprobamos que los dos contenedores están en ejecución:

```bash
sudo docker ps
```

Deben aparecer `wordpress_db` y `wordpress_app`, este último con el puerto `8888` publicado hacia el `80`.

> [!NOTE]
> El comando se ejecuta con `sudo` y desde el directorio del proyecto, porque Compose busca ahí el archivo `docker-compose.yml`. También puede ejecutarse desde otro directorio indicando el archivo con `-f <ruta>`.

---

## Parte E: Acceder a WordPress

**1.** Abrimos el sitio en el navegador:

```text
http://localhost:8888
```

La primera vez, WordPress muestra normalmente su **asistente de instalación**, donde se elige el idioma y se define el título del sitio y el usuario administrador. Con eso queda listo el sitio y su panel de administración (`/wp-admin`).

> [!NOTE]
> `localhost` solo funciona desde el mismo equipo que ejecuta Docker. Desde otra máquina, se usa la IP del servidor (`http://<IP-del-servidor>:8888`) y hay que permitir el puerto en el firewall: `sudo firewall-cmd --permanent --add-port=8888/tcp` seguido de `sudo firewall-cmd --reload`.

> [!WARNING]
> **El puerto 8888 ya se usó en la [práctica 22](../22-docker-nginx)** para el contenedor de Nginx. Si ese contenedor sigue en ejecución, el despliegue falla con un error de puerto ya asignado. En la [práctica 23](../23-docker-portainer) ese contenedor se detiene desde Portainer, lo que libera el puerto. Si no se detuvo, hay que hacerlo (`sudo docker stop <nombre>`) o cambiar el puerto de WordPress en `docker-compose.yml`, por ejemplo a `"8080:80"`.

> [!NOTE]
> El sitio funciona sobre **HTTP**, sin cifrado: el usuario y la contraseña del administrador viajan en claro por la red. Es aceptable en un laboratorio local; en un sitio real se coloca delante un proxy inverso con HTTPS.

---

## Administración del proyecto (opcional)

Comandos útiles, ejecutados desde el directorio del proyecto:

```bash
sudo docker compose ps          # estado de los servicios del proyecto
sudo docker compose logs -f     # registros de todos los servicios, en directo
sudo docker compose stop        # detiene los contenedores, sin eliminarlos
sudo docker compose start       # vuelve a iniciar los contenedores detenidos
sudo docker compose down        # detiene y elimina contenedores y red (los volúmenes se conservan)
```

> [!CAUTION]
> `docker compose down` conserva los datos, porque los volúmenes sobreviven. Pero **`docker compose down -v` elimina también los volúmenes**, y con ellos la base de datos y los archivos de WordPress: el sitio vuelve a mostrar el asistente de instalación desde cero. Es útil para empezar de nuevo, y destructivo si se ejecuta por error.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=Pe-R_S0-n98)

---

## Referencia de comandos

| Comando | Función |
|---|---|
| `docker-compose version` / `docker compose version` | Muestra la versión de Compose (binario independiente / complemento) |
| `docker compose up -d` | Crea e inicia todos los servicios del proyecto en segundo plano |
| `docker compose ps` | Lista los servicios del proyecto y su estado |
| `docker compose logs -f` | Muestra los registros de los servicios en directo |
| `docker compose stop` / `start` | Detiene / inicia los contenedores del proyecto |
| `docker compose down` | Detiene y elimina contenedores y red, conservando los volúmenes |
| `docker compose down -v` | Lo mismo, y elimina también los volúmenes (destruye los datos) |
| `docker ps` | Lista los contenedores en ejecución |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| `port is already allocated` al hacer `up` | Otro contenedor (por ejemplo, el Nginx de la práctica 22) ya usa el puerto 8888 | Detenerlo con `docker stop <nombre>` o cambiar el puerto publicado en `docker-compose.yml` |
| WordPress muestra *Error establishing a database connection* | La base de datos aún se está iniciando, o las contraseñas de ambos servicios no coinciden | Esperar unos segundos y revisar `docker compose logs db`; confirmar que `MYSQL_PASSWORD` y `WORDPRESS_DB_PASSWORD` son idénticas |
| Se cambió una contraseña en el archivo, pero sigue sin conectar | MySQL solo lee esas variables la **primera vez**, al inicializar el volumen; después las ignora | Si no hay datos que conservar, `docker compose down -v` y volver a desplegar; si los hay, cambiar la clave dentro de MySQL |
| `docker: 'compose' is not a docker command` | Falta el complemento `docker-compose-plugin` | Instalarlo con `sudo dnf install docker-compose-plugin`, o usar el binario con guion (`docker-compose`) |
| `docker-compose: command not found` | El binario no está en una ruta del `PATH`, o no tiene permisos de ejecución | Verificar que existe `/usr/local/bin/docker-compose` y repetir el `chmod +x` |
| No se accede al sitio desde otro equipo | El firewall bloquea el puerto 8888 | Permitir `8888/tcp` en `firewalld` |
| El sitio vuelve a pedir la instalación inicial | Se eliminaron los volúmenes (`down -v`) | Es lo esperado; para conservar los datos, usar solo `down` |
| El contenedor `wordpress_db` se reinicia continuamente | Error de configuración o de permisos en el volumen | Revisar `docker compose logs db` |

---

[← Volver al laboratorio](../README.md)
