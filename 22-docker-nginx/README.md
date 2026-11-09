[← Volver al laboratorio](../README.md)

# 22 · Instalación de Docker y despliegue de un servidor web con Nginx

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Engine-f472b6?style=flat-square&logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-Contenedor-60a5fa?style=flat-square&logo=nginx&logoColor=white)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Instalación de **Docker Engine** en RHEL desde el repositorio oficial de Docker, y despliegue de un servidor web **Nginx** dentro de un contenedor, sirviendo una página propia desde un directorio del servidor.

## Objetivo

Instalar Docker, comprobar que funciona, descargar una imagen desde Docker Hub y ejecutar un contenedor que publique una página web personalizada, montando un directorio del sistema como volumen.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Software | Docker Engine (`docker-ce`), repositorio oficial de Docker |
| Imagen | `nginx` (Docker Hub) |
| Directorio del sitio en el servidor | `/home/website` |
| Puerto publicado | `8888` (host) → `80` (contenedor) |

> [!NOTE]
> **Contenedor frente a máquina virtual.** Un contenedor comparte el núcleo del sistema operativo del anfitrión y empaqueta solo la aplicación y sus dependencias, por lo que arranca en segundos y consume mucho menos que una VM. Una **imagen** es la plantilla de solo lectura; un **contenedor** es una instancia en ejecución de esa imagen.

## Índice

- [Parte A: Preparar el sistema](#parte-a-preparar-el-sistema)
- [Parte B: Instalar Docker Engine](#parte-b-instalar-docker-engine)
- [Parte C: Verificar la instalación](#parte-c-verificar-la-instalación)
- [Parte D: Descargar la imagen de Nginx](#parte-d-descargar-la-imagen-de-nginx)
- [Parte E: Crear el sitio web](#parte-e-crear-el-sitio-web)
- [Parte F: Ejecutar el contenedor](#parte-f-ejecutar-el-contenedor)
- [Limpieza (opcional)](#limpieza-opcional)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Preparar el sistema

**1.** Desinstalamos versiones anteriores de Docker y los motores de contenedores que vienen por defecto en RHEL (`podman` y `runc`), para evitar conflictos con el nuevo paquete:

```bash
sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine \
                  podman \
                  runc
```

> [!NOTE]
> Si alguno de esos paquetes no está instalado, `dnf` lo indica y sigue con los demás; no es un error. **Podman** es el motor de contenedores que Red Hat incluye y respalda oficialmente en RHEL, y no necesita un servicio permanente (*daemon*) como Docker. Eliminarlo es opcional: ambos pueden convivir, pero aquí se quitó para evitar conflictos con `runc`, que instala Docker.

---

## Parte B: Instalar Docker Engine

**1.** Instalamos el complemento que permite administrar repositorios y agregamos el repositorio oficial de Docker:

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

**2.** Instalamos Docker Engine y sus componentes:

```bash
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

| Paquete | Función |
|---|---|
| `docker-ce` | El motor de Docker (el servicio que ejecuta los contenedores) |
| `docker-ce-cli` | La herramienta de línea de comandos `docker` |
| `containerd.io` | El entorno de ejecución de contenedores de bajo nivel |
| `docker-buildx-plugin` | Compilación avanzada de imágenes |
| `docker-compose-plugin` | Define y ejecuta aplicaciones de varios contenedores (`docker compose`) |

> [!TIP]
> Al instalar desde un repositorio nuevo, `dnf` solicita **confirmar la clave GPG** de Docker. Es el mecanismo que garantiza que los paquetes provienen del repositorio oficial; se acepta tras comprobar que el origen es `download.docker.com`.

**3.** Habilitamos e iniciamos el servicio, y comprobamos su estado:

```bash
sudo systemctl enable --now docker
sudo systemctl start docker
sudo systemctl status docker
```

> [!NOTE]
> `enable --now` ya habilita **e inicia** el servicio, así que el segundo comando (`start`) es redundante: no causa ningún problema, simplemente no hace nada nuevo.

---

## Parte C: Verificar la instalación

**1.** Ejecutamos el contenedor de prueba oficial:

```bash
sudo docker run hello-world
```

Docker no encuentra la imagen `hello-world` en el equipo, la descarga de Docker Hub, crea un contenedor y lo ejecuta. Al terminar, imprime un mensaje que confirma que la instalación funciona.

**2.** Consultamos la versión instalada:

```bash
docker --version
```

> [!TIP]
> Todos los comandos de `docker` se ejecutan con `sudo` porque el socket del servicio solo es accesible por `root`. Agregar un usuario al grupo `docker` evita el `sudo`, pero conviene saber que **pertenecer a ese grupo equivale a tener acceso de `root`** sobre el sistema (un contenedor puede montar cualquier directorio del anfitrión). Por eso en esta práctica se mantiene el uso de `sudo`.

---

## Parte D: Descargar la imagen de Nginx

**1.** Buscamos la imagen en Docker Hub:

```bash
sudo docker search nginx
```

Aparece una lista de imágenes relacionadas; la primera, `nginx`, es la **imagen oficial**.

**2.** Descargamos la imagen:

```bash
sudo docker pull nginx
```

**3.** Verificamos que quedó almacenada localmente:

```bash
sudo docker images
```

> [!NOTE]
> Sin una etiqueta (*tag*) explícita, `docker pull nginx` descarga la etiqueta `latest`, que cambia con el tiempo. Para un despliegue reproducible conviene fijar una versión concreta, por ejemplo `nginx:1.27`.

---

## Parte E: Crear el sitio web

**1.** Creamos el directorio que contendrá el sitio y abrimos el archivo de la página:

```bash
sudo mkdir /home/website
cd /home/website/
sudo nano index.html
```

**2.** Contenido del archivo:

```html
<html>
 <head>
  <title>DOCKER x NGINX</title>
 </head>
 <body>
  <H3>Neury Montero<br>
  Matrícula: 2025-0829</H3>
 </body>
</html>
```

> [!TIP]
> La página no declara su codificación de caracteres. Algunos navegadores pueden mostrar mal las letras con tilde (como la **í** de *Matrícula*) si la adivinan incorrectamente. Se evita agregando `<meta charset="UTF-8">` dentro de `<head>`.

---

## Parte F: Ejecutar el contenedor

**1.** Ejecutamos un contenedor de Nginx que publica el puerto 80 en el puerto 8888 del servidor y toma su contenido del directorio que acabamos de crear:

```bash
sudo docker run -d -p 8888:80 -v /home/website:/usr/share/nginx/html/ nginx
```

| Opción | Significado |
|---|---|
| `-d` | Ejecuta el contenedor en segundo plano (*detached*) |
| `-p 8888:80` | Publica el puerto `80` del contenedor en el puerto `8888` del anfitrión |
| `-v /home/website:/usr/share/nginx/html/` | Monta el directorio `/home/website` del servidor sobre la carpeta donde Nginx busca su sitio, de modo que sirve **tu** `index.html` |
| `nginx` | Imagen a utilizar |

**2.** Comprobamos que el contenedor está en ejecución:

```bash
sudo docker ps -a
```

`-a` muestra **todos** los contenedores, incluidos los detenidos. La columna `PORTS` debe indicar `0.0.0.0:8888->80/tcp`.

**3.** Abrimos la página en el navegador de la máquina:

```text
127.0.0.1:8888
```

Se muestra la página con el nombre y la matrícula.

> [!NOTE]
> Al estar el contenido **montado como volumen**, cualquier cambio en `/home/website/index.html` se refleja de inmediato al recargar la página, sin reiniciar el contenedor.

> [!NOTE]
> `127.0.0.1` solo funciona si el navegador se ejecuta **en el mismo equipo** que Docker. Desde otra máquina, se usa la IP del servidor (`http://<IP-del-servidor>:8888`) y hay que permitir el puerto en el firewall: `sudo firewall-cmd --permanent --add-port=8888/tcp` seguido de `sudo firewall-cmd --reload`.

---

## Limpieza (opcional)

Para detener y eliminar el contenedor cuando ya no se necesite (el nombre o ID se obtiene con `docker ps`):

```bash
sudo docker stop <nombre-o-id>
sudo docker rm <nombre-o-id>
```

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=u8yiPQjEThI)

---

## Referencia de comandos

| Comando | Función |
|---|---|
| `dnf config-manager --add-repo <url>` | Agrega un repositorio de paquetes |
| `systemctl enable --now docker` | Habilita e inicia el servicio de Docker |
| `docker run hello-world` | Verifica que Docker funciona |
| `docker search <término>` | Busca imágenes en Docker Hub |
| `docker pull <imagen>` | Descarga una imagen |
| `docker images` | Lista las imágenes almacenadas localmente |
| `docker run -d -p <host>:<contenedor> -v <ruta>:<ruta> <imagen>` | Ejecuta un contenedor en segundo plano, publicando un puerto y montando un volumen |
| `docker ps` / `docker ps -a` | Lista los contenedores en ejecución / todos los contenedores |
| `docker stop` / `docker rm` | Detiene / elimina un contenedor |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| `Cannot connect to the Docker daemon` | El servicio no está activo, o se ejecutó el comando sin `sudo` | Verificar `systemctl status docker` y usar `sudo` |
| `dnf` no encuentra `docker-ce` | El repositorio de Docker no se agregó correctamente | Revisar `dnf repolist` y volver a ejecutar `config-manager --add-repo` |
| Conflicto de paquetes con `runc` o `podman` al instalar | Quedan paquetes previos de otro motor de contenedores | Ejecutar el paso de desinstalación de la Parte A |
| `port is already allocated` | Otro proceso o contenedor ya usa el puerto 8888 | Elegir otro puerto (`-p 9999:80`) o detener el contenedor que lo ocupa |
| La página responde `403 Forbidden` | SELinux impide al contenedor leer el directorio montado | Agregar `:Z` al volumen (`-v /home/website:/usr/share/nginx/html/:Z`) para que Docker reetiquete el directorio; verificar con `ausearch -m avc -ts recent` |
| La página responde `404` o muestra la página por defecto de Nginx | El volumen no apunta a la ruta correcta, o `index.html` no existe en `/home/website` | Revisar la ruta del `-v` y el contenido con `ls /home/website` |
| No se accede desde otra máquina | El firewall bloquea el puerto publicado | Permitir `8888/tcp` en `firewalld` |
| Las letras con tilde se ven mal | El HTML no declara su codificación | Agregar `<meta charset="UTF-8">` dentro de `<head>` |

---

[← Volver al laboratorio](../README.md)
