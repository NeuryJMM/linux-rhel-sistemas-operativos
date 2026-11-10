[← Volver al laboratorio](../README.md)

# 23 · Administración de contenedores con Portainer

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Contenedores-f472b6?style=flat-square&logo=docker&logoColor=white)
![Portainer](https://img.shields.io/badge/Portainer-CE-60a5fa?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Instalación de **Portainer CE**, una interfaz web para administrar Docker, ejecutada como un contenedor más. Se conecta un entorno de Docker mediante el **agente de Portainer** y se detiene desde la interfaz el contenedor de Nginx creado en la [práctica 22](../22-docker-nginx).

## Objetivo

Administrar contenedores de forma gráfica en lugar de por línea de comandos: instalar Portainer, conectarlo a un entorno Docker mediante su agente y controlar un contenedor existente desde el navegador.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Requisito previo | Docker Engine instalado ([práctica 22](../22-docker-nginx)) |
| Imagen | `portainer/portainer-ce:lts` |
| Interfaz web | `https://127.0.0.1:9443` |
| Puerto del agente | `9001` |
| Volumen de datos | `portainer_data` |

> [!NOTE]
> **Cómo funciona Portainer.** Portainer es una aplicación web que se comunica con el motor de Docker a través de su API. Para ello, el contenedor de Portainer recibe acceso al **socket de Docker** (`/var/run/docker.sock`) del servidor. El **agente** (`portainer/agent`) es un componente adicional que se instala en cada servidor Docker que se quiera administrar, y expone la API de ese servidor hacia Portainer en el puerto `9001`.

> [!WARNING]
> **El socket de Docker equivale a acceso de `root`.** Montar `/var/run/docker.sock` dentro de un contenedor permite que ese contenedor cree otros, monte cualquier directorio del servidor y, en la práctica, controle por completo el sistema anfitrión. Es lo que hace útil a Portainer, pero también significa que **quien controle Portainer controla el servidor**. Por eso conviene protegerlo con una contraseña robusta, no exponer el puerto `9443` a Internet y limitar el acceso por firewall a las redes de administración.

## Índice

- [Parte A: Instalar Portainer](#parte-a-instalar-portainer)
- [Parte B: Acceder y crear el usuario administrador](#parte-b-acceder-y-crear-el-usuario-administrador)
- [Parte C: Conectar un entorno con el agente](#parte-c-conectar-un-entorno-con-el-agente)
- [Parte D: Detener un contenedor desde Portainer](#parte-d-detener-un-contenedor-desde-portainer)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Instalar Portainer

**1.** Creamos un volumen de Docker, donde Portainer guardará su base de datos y su configuración, de modo que sobrevivan si el contenedor se elimina o se actualiza:

```bash
sudo docker volume create portainer_data
```

**2.** Ejecutamos el contenedor de Portainer:

```bash
sudo docker run -d -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:lts
```

| Opción | Significado |
|---|---|
| `-d` | Ejecuta el contenedor en segundo plano |
| `-p 9443:9443` | Publica el puerto HTTPS de la interfaz web de Portainer |
| `--name portainer` | Asigna un nombre fácil de recordar al contenedor |
| `--restart=always` | Lo reinicia automáticamente si se detiene o si el servidor se reinicia |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Da a Portainer acceso al motor de Docker del servidor |
| `-v portainer_data:/data` | Guarda los datos de Portainer en el volumen creado |
| `portainer/portainer-ce:lts` | Edición Community, en su versión de **soporte extendido** (*Long-Term Support*) |

> [!TIP]
> La etiqueta `lts` apunta a la versión de soporte extendido, que recibe correcciones durante más tiempo y cambia menos a menudo que `latest`. Es la opción más estable para un servidor que se quiere mantener.

**3.** Comprobamos que el contenedor está en ejecución:

```bash
sudo docker ps -a
```

Debe aparecer `portainer` con estado `Up` y el puerto `9443` publicado.

---

## Parte B: Acceder y crear el usuario administrador

**1.** Desde el navegador del servidor, entramos a:

```text
https://127.0.0.1:9443
```

> [!NOTE]
> El navegador muestra una advertencia de certificado: Portainer genera un certificado **autofirmado** por defecto. Es esperado en un laboratorio. Se acepta el riesgo para continuar; en un entorno real, se configura un certificado emitido por una entidad de confianza.

**2.** En el primer acceso, Portainer pide crear el **usuario administrador** y su contraseña. Una vez creado, se inicia sesión en la interfaz.

> [!IMPORTANT]
> Portainer solo permite crear ese primer usuario durante los **primeros minutos** tras iniciar el contenedor. Pasado ese plazo, el asistente se bloquea por seguridad y hay que reiniciar el contenedor (`sudo docker restart portainer`) para poder crearlo.

---

## Parte C: Conectar un entorno con el agente

**1.** En Portainer, vamos a **Add Environments** → **Docker Standalone** → **Start Wizard**.

**2.** El asistente muestra un **comando** para instalar el agente. Lo copiamos y lo pegamos en la terminal del servidor.

> [!NOTE]
> El comando exacto lo genera el asistente de Portainer y depende de la versión instalada, por lo que no se reproduce aquí. Se obtiene siempre de ese asistente para garantizar que coincide con la versión del servidor. Es, en esencia, un `docker run` que despliega el contenedor `portainer_agent` publicando el puerto `9001`.

**3.** De vuelta en Portainer, completamos los campos del entorno:

| Campo | Valor |
|---|---|
| **Name** | Un nombre descriptivo para el entorno |
| **Environment address** | `<IP-del-servidor>:9001` (la dirección del equipo donde corre el agente y el puerto del agente) |

**4.** Hacemos clic en **Connect** y después en **Close**. El entorno queda registrado y se puede administrar desde la interfaz.

> [!TIP]
> En esta práctica, el agente se instala en el **mismo servidor** donde corre Portainer, con fines de demostración. En un uso real, el agente se instala en cada servidor **remoto** que se quiera administrar, y una sola instancia de Portainer los gestiona a todos desde un panel único. Si se usa el firewall, el puerto `9001/tcp` debe estar permitido entre Portainer y cada agente.

---

## Parte D: Detener un contenedor desde Portainer

**1.** En la interfaz, entramos al entorno que se acaba de crear y vamos a **Containers**.

**2.** Seleccionamos el contenedor **Nginx** (el creado en la [práctica 22](../22-docker-nginx)) y hacemos clic en **Stop**.

**3.** El contenedor pasa a estado detenido. Para comprobarlo, se puede intentar abrir la página de la práctica anterior (`127.0.0.1:8888`): deja de responder. Con **Start**, se vuelve a iniciar.

> [!NOTE]
> Esta parte depende de que el contenedor de Nginx de la práctica 22 siga existiendo. Si se eliminó, la lista de contenedores de Portainer no lo mostrará y habrá que volver a crearlo con el comando `docker run` de esa práctica.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=k6j5S9PXwLk)

---

## Referencia de comandos

| Comando | Función |
|---|---|
| `docker volume create <nombre>` | Crea un volumen persistente de Docker |
| `docker run -d -p <puerto>:<puerto> --name <nombre> --restart=always ...` | Ejecuta un contenedor en segundo plano, con nombre y reinicio automático |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Da a un contenedor acceso al motor de Docker del anfitrión |
| `docker ps -a` | Lista todos los contenedores |
| `docker restart <contenedor>` | Reinicia un contenedor |
| `https://<servidor>:9443` | Interfaz web de Portainer |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| La página de Portainer no carga | El contenedor no está activo, o se usó `http` en vez de `https` | Verificar `docker ps` y entrar por `https://<IP>:9443` |
| Portainer indica que el tiempo de configuración inicial expiró | Pasó el plazo para crear el primer usuario | Ejecutar `sudo docker restart portainer` y crearlo de inmediato |
| No se puede acceder desde otro equipo | El firewall bloquea el puerto 9443 | `sudo firewall-cmd --permanent --add-port=9443/tcp` y `sudo firewall-cmd --reload` |
| El entorno no conecta ("Connect" falla) | La dirección del agente es incorrecta, o el puerto 9001 está bloqueado | Revisar `IP:9001`, comprobar que `portainer_agent` esté activo con `docker ps` y permitir `9001/tcp` en el firewall |
| El contenedor de Portainer arranca y se detiene | Falta acceso al socket de Docker, por ejemplo por SELinux | Revisar `docker logs portainer` y `ausearch -m avc -ts recent` |
| No aparece el contenedor Nginx | Se eliminó o nunca se creó | Volver a crearlo con el comando de la [práctica 22](../22-docker-nginx) |

---

[← Volver al laboratorio](../README.md)
