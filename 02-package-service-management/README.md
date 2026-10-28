[← Volver al laboratorio](../README.md)

# 02 · Antivirus ClamAV: instalación, actualización y escaneos programados

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![ClamAV](https://img.shields.io/badge/ClamAV-Antivirus-f472b6?style=flat-square)
![Cron](https://img.shields.io/badge/Cron-Tareas%20Programadas-60a5fa?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Instalación y configuración de **ClamAV**, un motor antivirus de código abierto, en RHEL: habilitación de los repositorios necesarios, actualización automática de firmas con FreshClam, escaneos programados con `cron` y revisión de los registros generados.

## Objetivos

| Punto | Descripción |
|:--:|---|
| **A** | Implementar ClamAV en una distribución de preferencia |
| **B** | Configurar las actualizaciones automáticas de la base de firmas |
| **C** | Configurar los escaneos periódicos |
| **D** | Mostrar los *logs* generados por ClamAV |

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Gestor de paquetes | `dnf` |
| Repositorios requeridos | CodeReady Builder (CRB) y EPEL |
| Componentes instalados | `clamav`, `clamd`, `clamav-update` |
| Servicio de actualización de firmas | `clamav-freshclam` |
| Script de escaneo | `/usr/local/bin/escaneo_clamav.sh` |
| Log de escaneos | `/var/log/clamav_scan.log` |

## Índice

- [Parte A: Implementar ClamAV](#parte-a-implementar-clamav)
- [Parte B: Actualizaciones automáticas](#parte-b-actualizaciones-automáticas)
- [Parte C: Escaneos periódicos con cron](#parte-c-escaneos-periódicos-con-cron)
- [Parte D: Logs generados por ClamAV](#parte-d-logs-generados-por-clamav)
- [Referencia de comandos y archivos](#referencia-de-comandos-y-archivos)
- [Demostración complementaria: actualización de repositorios e instalación manual de herramientas](#demostración-complementaria-actualización-de-repositorios-e-instalación-manual-de-herramientas)
- [Segunda demostración complementaria: servidor web Apache (httpd)](#segunda-demostración-complementaria-servidor-web-apache-httpd)
- [Demostración en video](#demostración-en-video)
- [Lo aprendido](#lo-aprendido)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Implementar ClamAV

**1.** Actualizamos los repositorios a su versión más reciente (con privilegios de `sudo`):

```bash
dnf update -y
```

![dnf update -y antes de empezar](images/a01-dnf-update.jpg)


**2.** Activamos el repositorio **CodeReady Builder (CRB)**, donde se encuentran varias dependencias de ClamAV. En algunas distribuciones ya viene activado por defecto.

![Activación del repositorio CodeReady Builder](images/a02-codeready-builder.jpg)


**3.** Instalamos el repositorio **EPEL** (*Extra Packages for Enterprise Linux*), donde se encuentra el paquete de ClamAV:

![Instalación del repositorio EPEL](images/a03-instalar-epel.jpg)


> [!IMPORTANT]
> Es importante activar el repositorio CodeReady Builder **antes** de instalar EPEL, porque muchos de sus paquetes dependen de él.

**4.** Actualizamos nuevamente los repositorios, ya con CRB y EPEL habilitados:

![dnf update -y tras habilitar los repositorios](images/a04-dnf-update-2.jpg)


**5.** Instalamos ClamAV y sus componentes:

```bash
sudo dnf -y install clamav clamd clamav-update
```

![Instalación de clamav, clamd y clamav-update](images/a05-dnf-install-clamav.jpg)


**6.** Validamos que la instalación se completó correctamente:

```bash
clamscan --version
```

![Verificación con clamscan --version](images/a06-clamscan-version.jpg)


> [!NOTE]
> ClamAV incluye dos herramientas principales: **ClamScan**, que es la que se acaba de instalar y realiza los escaneos en busca de virus, y **FreshClam**, que actualiza la base de firmas que ClamScan utiliza para detectarlos.

**7.** Ejecutamos FreshClam para descargar la base de firmas por primera vez:

```bash
sudo freshclam
```

![Ejecución de sudo freshclam](images/a07-sudo-freshclam.jpg)


**8.** Creamos el archivo de log de FreshClam y le asignamos los permisos necesarios para que pueda administrarlo:

```bash
touch /var/log/freshclam.log
```

![Creación del archivo de log de FreshClam](images/a08-touch-freshclam-log.jpg)


```bash
chmod 600 /var/log/freshclam.log
chown root /var/log/freshclam.log
```

![chmod 600 y chown root sobre freshclam.log](images/a09-permisos-freshclam-log.jpg)


**9.** Activamos el servicio de FreshClam:

```bash
systemctl start clamav-freshclam.service
```

![systemctl start clamav-freshclam.service](images/a10-systemctl-start-freshclam.jpg)


**10.** Para verificar que se activó correctamente, repetimos el comando anterior cambiando `start` por `status`:

```bash
systemctl status clamav-freshclam.service
```

![Estado activo del servicio clamav-freshclam](images/a11-systemctl-status-freshclam.jpg)


El servicio aparece como `active (running)`, confirmando que se inicializó correctamente.

---

## Parte B: Actualizaciones automáticas

**1.** Entramos a la configuración global de FreshClam:

```bash
nano /etc/freshclam.conf
```

![Apertura de /etc/freshclam.conf](images/b01-nano-freshclam-conf.jpg)


**2.** Buscamos la línea `DatabaseMirror database.clamav.net` y la descomentamos. Esto activa un dominio primario desde el cual se descargan las actualizaciones de la base de firmas.

**3.** Buscamos la línea `UpdateLogFile /var/log/freshclam.log` y la descomentamos. Esto centraliza el registro de las actualizaciones en ese archivo.

![DatabaseMirror database.clamav.net descomentado](images/b02-databasemirror-descomentado.jpg)


**4.** Guardamos (`Ctrl + O`, `Enter`) y salimos (`Ctrl + X`).

**5.** Habilitamos el servicio para que se inicie automáticamente con el sistema:

```bash
systemctl enable clamav-freshclam
```

![systemctl enable clamav-freshclam](images/b03-systemctl-enable-freshclam.jpg)


> [!NOTE]
> `systemctl start` inicia el servicio en la sesión actual; `systemctl enable` hace que se inicie automáticamente en cada arranque del sistema. Se necesitan ambos para que las actualizaciones automáticas queden activas de forma permanente.

---

## Parte C: Escaneos periódicos con cron

**1.** Creamos un script para automatizar el proceso de escaneo, en la ruta `/usr/local/bin/`:

```bash
nano /usr/local/bin/escaneo_clamav.sh
```

![Creación del script /usr/local/bin/escaneo_clamav.sh](images/c01-nano-escaneo-script.jpg)


**2.** Establecemos que el archivo se interprete en Bash:

```bash
#!/bin/bash
```

![Línea #!/bin/bash del script](images/c02-shebang-bash.png)


**3.** Escribimos el resto del script:

```bash
#!/bin/bash
LOGFILE="/var/log/clamav_scan.log"
clamscan -r /home /var --move=/quarantine --log=$LOGFILE
```

![Contenido del script: LOGFILE y clamscan -r](images/c03-script-clamscan.jpg)


`LOGFILE` define la variable con la ruta del archivo de registro. `clamscan -r` escanea `/home` y `/var` de forma recursiva, `--move=/quarantine` mueve a cuarentena los archivos infectados que se encuentren, y `--log=$LOGFILE` guarda el resultado del escaneo en el archivo de registro, haciéndolo visible más adelante.

**4.** Guardamos (`Ctrl + O`, `Enter`) y salimos (`Ctrl + X`).

**5.** Le damos permisos de ejecución al script:

```bash
chmod +x /usr/local/bin/escaneo_clamav.sh
```

![chmod +x sobre el script](images/c04-chmod-ejecucion.jpg)


**6.** Configuramos el escaneo periódico con `crontab`:

```bash
sudo crontab -e
```

![sudo crontab -e](images/c05-crontab-e.jpg)


**7.** Presionamos `I` para entrar en modo de inserción (el editor por defecto de `crontab -e` en este sistema es `vi`) y escribimos:

```cron
0 3 * * * /usr/local/bin/escaneo_clamav.sh
```

![Entrada de crontab: 0 3 * * * escaneo_clamav.sh](images/c06-crontab-entrada.jpg)


Esta expresión de `cron` ejecuta el script todos los días, sin excepción, a las **3:00 a.m.**

**8.** Presionamos `Esc` y escribimos `:wq` para guardar y salir de `vi`.

> [!TIP]
> La sintaxis de `cron` sigue el orden `minuto hora día-del-mes mes día-de-la-semana`. El asterisco (`*`) significa "cualquier valor". Por eso `0 3 * * *` se lee como *"al minuto 0, de la hora 3, de cualquier día, de cualquier mes, de cualquier día de la semana"*: todos los días a las 3:00 a.m.

---

## Parte D: Logs generados por ClamAV

**1.** Realizamos un escaneo manual al directorio `home`:

```bash
clamscan -r /home
```

**2.** Esperamos a que finalice.

**3.** Al terminar, se muestra el **SCAN SUMMARY** con los detalles del escaneo: virus conocidos, directorios y archivos escaneados, archivos infectados, datos escaneados y el tiempo que tomó.

![SCAN SUMMARY del escaneo al directorio home](images/d01-clamscan-home-resumen.jpg)


**4.** También se puede consultar el registro guardado por el script programado, usando `cat` en vez de ejecutar `clamscan` de nuevo:

```bash
cat /var/log/clamav_scan.log
```

![cat /var/log/clamav_scan.log](images/d02-cat-clamav-scan-log.jpg)

![Contenido del log con el SCAN SUMMARY](images/d03-log-scan-summary.jpg)


El archivo de log muestra el mismo tipo de resumen (**SCAN SUMMARY**), generado automáticamente la próxima vez que `cron` ejecute el script a las 3:00 a.m.

---

## Referencia de comandos y archivos

| Comando / archivo | Función |
|---|---|
| `dnf update -y` | Actualiza los repositorios y paquetes instalados |
| `subscription-manager repos --enable codeready-builder-for-rhel-*` | Habilita el repositorio CodeReady Builder |
| `dnf install epel-release` | Instala el repositorio EPEL |
| `dnf install clamav clamd clamav-update` | Instala ClamAV y sus componentes |
| `clamscan --version` | Verifica la instalación y la versión de ClamAV |
| `freshclam` | Descarga o actualiza manualmente la base de firmas |
| `systemctl start/enable/status clamav-freshclam` | Inicia, habilita al arranque o consulta el estado del servicio de actualización |
| `/etc/freshclam.conf` | Configuración de FreshClam (mirror de la base de datos, archivo de log) |
| `clamscan -r <ruta>` | Escanea un directorio de forma recursiva |
| `crontab -e` | Edita las tareas programadas del usuario actual |
| `/var/log/clamav_scan.log` | Registro de los escaneos programados |

---

## Lo aprendido

Esta práctica permitió instalar y configurar un antivirus de línea de comandos en RHEL de principio a fin: habilitar los repositorios necesarios (CodeReady Builder y EPEL), instalar ClamAV, mantener su base de firmas actualizada con FreshClam, y automatizar los escaneos con un script de Bash programado mediante `cron`.

Se comprendió la diferencia entre `clamscan` (el motor de escaneo) y `freshclam` (el actualizador de firmas), por qué CodeReady Builder debe habilitarse antes que EPEL, la diferencia entre iniciar un servicio (`start`) y dejarlo habilitado de forma permanente (`enable`), y la sintaxis de `cron` para programar tareas recurrentes. También se trabajó con redirección de logs y con las banderas de `clamscan` para mover automáticamente los archivos infectados a cuarentena.

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo en el laboratorio. Las capturas del procedimiento se encuentran en la carpeta `images/`.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=WTApO2vpP38)

## Demostración complementaria: actualización de repositorios e instalación manual de herramientas

Esta demostración trabaja dos cosas: mantener los repositorios del sistema al día, y qué hacer cuando una herramienta **no** está disponible en ellos.

### Actualizar los repositorios

**1.** Actualizamos los paquetes con `dnf update`:

```bash
dnf update
```

Se muestra el resumen de instalación/actualización y se confirma con `y`.

![dnf update y resumen de paquetes](images/comp01-dnf-update.jpg)
![Resumen de la transacción](images/comp02-paquetes-resumen.jpg)

Durante el proceso, `dnf` importa las llaves GPG de Red Hat (necesarias para verificar la autenticidad de los paquetes) y ejecuta la transacción.

![Importación de llaves GPG y transacción completada](images/comp03-gpg-transaccion.jpg)

**2.** De forma similar, `dnf upgrade` actualiza los paquetes instalados a su versión más reciente disponible en los repositorios:

```bash
dnf upgrade
```

![dnf upgrade](images/comp04-dnf-upgrade.jpg)

> [!NOTE]
> `dnf update` y `dnf upgrade` son, en la práctica, equivalentes en la mayoría de los casos; `upgrade` es el nombre más explícito, mientras que `update` se mantiene por compatibilidad con versiones anteriores de la herramienta (`yum`).

### Explorar los repositorios configurados

**3.** Listamos los repositorios habilitados:

```bash
dnf repolist
```

Se muestran los repositorios activos por defecto: **AppStream** y **BaseOS**.

![dnf repolist: AppStream y BaseOS](images/comp05-repolist.jpg)

**4.** Para ver **todos** los repositorios disponibles, incluidos los deshabilitados (como los de depuración y código fuente):

```bash
dnf repolist all
```

![Comando dnf repolist all](images/comp06-repolist-all-comando.png)
![Listado completo de repositorios, habilitados y deshabilitados](images/comp07-repolist-all-completo.jpg)

### Instalar una herramienta que no está en los repositorios

**5.** Buscamos una herramienta (en este caso, **Bashtop**, un monitor de sistema para la terminal) en los repositorios configurados:

```bash
dnf search bashtop
```

```text
No matches found.
```

![dnf search bashtop sin resultados](images/comp08-dnf-search-bashtop.jpg)

**6.** Como no está disponible por `dnf`, la obtenemos directamente desde su repositorio en GitHub con `git clone` (si `git` no está instalado, `dnf` ofrece instalarlo automáticamente en el momento):

```bash
git clone https://github.com/aristocratos/bashtop.git
```

![Instalación de git-core y clonado del repositorio](images/comp09-git-clone-bashtop.jpg)

**7.** Entramos a la carpeta clonada y revisamos su contenido:

```bash
cd bashtop
ls
```

![Contenido del repositorio clonado](images/comp10-cd-ls-bashtop.jpg)

**8.** Ejecutamos Bashtop directamente, sin necesidad de instalarlo como paquete:

```bash
./bashtop
```

Al iniciar, realiza una serie de verificaciones (herramientas disponibles, CPU, dispositivos de red, actualizaciones).

![Pantalla de inicio de Bashtop](images/comp11-bashtop-splash.jpg)

**9.** Bashtop se ejecuta como un monitor de sistema interactivo en la terminal (CPU, memoria, disco, red y procesos):

![Bashtop en ejecución](images/comp12-bashtop-ejecucion.jpg)

**10.** Presionando `Esc`, se accede al menú de opciones (tema de color, orden de procesos, intervalo de actualización, entre otros):

![Menú de opciones de Bashtop](images/comp13-bashtop-opciones.jpg)

### Limpieza

**11.** Como Bashtop no se instaló mediante `dnf`, `dnf autoremove` no lo reconoce como un paquete que pueda desinstalar:

```bash
cd ..
dnf autoremove bashtop
```

```text
No match for argument: bashtop
```

Por eso, se elimina manualmente la carpeta clonada con `rm -r`, que confirma la eliminación de cada archivo:

```bash
rm -r bashtop
```

![Limpieza manual de la carpeta clonada](images/comp14-limpieza-rm.jpg)

**12.** Por último, limpiamos la caché local de paquetes de `dnf`:

```bash
dnf clean all
```

![dnf clean all](images/comp15-dnf-clean-all.jpg)


---
## Segunda demostración complementaria: servidor web Apache (httpd)

Esta demostración instala y configura el servidor web **Apache (`httpd`)** en RHEL, con dos sitios distintos sirviéndose desde el mismo servidor: uno en el puerto estándar (80) y otro en un puerto adicional (8080), cada uno con su propio dominio local y su propio contenido.

### Instalar y activar Apache

**1.** Instalamos el paquete `httpd`:

```bash
dnf install httpd
```

![Comando dnf install httpd](images/a01-dnf-install-httpd.jpg)

Se instalan, además del propio `httpd`, sus dependencias (`apr`, `apr-util`, `httpd-filesystem`, `mod_http2`, `redhat-logos-httpd`, entre otras).

![Paquetes a instalar: httpd y sus dependencias](images/a02-paquetes-httpd.jpg)

**2.** Habilitamos el servicio para que inicie automáticamente con el sistema, lo iniciamos, y verificamos su estado:

```bash
systemctl enable httpd
systemctl start httpd
systemctl status httpd
```

El servicio queda **activo (running)**, escuchando en el puerto 80.

![systemctl enable, start y status de httpd](images/a03-enable-start-status.jpg)

> [!NOTE]
> El mensaje `AH00558: httpd: Could not reliably determine the server's fully qualified domain name` es una advertencia común y no impide que el servicio funcione: indica que Apache no pudo determinar automáticamente el nombre de dominio completo del servidor, y usa una dirección por defecto mientras tanto. Se resuelve configurando la directiva `ServerName` en `httpd.conf`.

### Primer sitio: hola.localhost (puerto 80)

**3.** Creamos el directorio para el contenido del primer sitio y un archivo `index.html`:

```bash
cd /var/www/html
mkdir hola
cd ./hola
nano index.html
```

![mkdir hola y nano index.html](images/b01-crear-hola-index.jpg)

**4.** Contenido del archivo:

```html
<!DOCTYPE html>
<html>
<head><title>Hola Mundo</title></head>
<body>
<h1>Hola Mundo</h1>
</body>
</html>
```

![Contenido de index.html para hola.localhost](images/b02-hola-index-contenido.png)

**5.** Editamos la configuración principal de Apache:

```bash
nano /etc/httpd/conf/httpd.conf
```

![Comando nano /etc/httpd/conf/httpd.conf](images/b03-nano-httpd-conf.jpg)

El archivo trae, por defecto, varias secciones comentadas con su explicación (por ejemplo, sobre `EnableMMAP` y `EnableSendfile`).

![Configuración por defecto de httpd.conf](images/b04-httpd-conf-inicio.jpg)

Al final del archivo, agregamos un bloque `VirtualHost` que asocia el dominio `hola.localhost` con la carpeta `hola` creada antes:

```apache
<VirtualHost *:80>
  ServerName hola.localhost
  DocumentRoot "/var/www/html/hola"
  ServerAdmin root@localhost
</VirtualHost>
```

**6.** Para que el nombre `hola.localhost` se resuelva hacia el propio equipo, lo agregamos a `/etc/hosts`:

```bash
nano /etc/hosts
```

![Comando nano /etc/hosts](images/b05-nano-etc-hosts.png)

```text
127.0.0.1   hola.localhost
```

![/etc/hosts con hola.localhost agregado](images/b06-etc-hosts-hola.jpg)

**7.** Reiniciamos el servicio para aplicar los cambios:

```bash
systemctl restart httpd
```

![systemctl restart httpd](images/b07-systemctl-restart-httpd.png)

**8.** Ingresamos a `http://hola.localhost` desde el navegador: se muestra la página creada.

![hola.localhost mostrando Hola Mundo en el navegador](images/b08-navegador-hola-mundo.jpg)

### Segundo sitio: estudiante.localhost (puerto 8080)

**9.** Repetimos el procedimiento para un segundo sitio, esta vez sirviéndose en el puerto **8080**. Creamos su directorio y su `index.html`:

```bash
cd /var/www/html
mkdir estudiante
cd ./estudiante
nano index.html
```

![mkdir estudiante y nano index.html](images/c01-crear-estudiante-index.jpg)

**10.** Contenido del archivo, con información del estudiante:

```html
<!DOCTYPE html>
<html>
<head><title>Estudiante</title></head>
<body>
<h2>Neury Jorge Montero Mejia</h2>
<p>Matricula: 2025-0829</p>
<p>Materia: Sistemas Operativos 3<br>
Docente: Adrian Alcantara<p/>
</body>
</html>
```

![Contenido de index.html para estudiante.localhost](images/c02-estudiante-index-contenido.jpg)

**11.** Volvemos a editar `httpd.conf`:

```bash
nano /etc/httpd/conf/httpd.conf
```

![Segunda edición de httpd.conf](images/c03-nano-httpd-conf-2.jpg)

Agregamos una segunda directiva `Listen`, para que Apache también escuche en el puerto 8080 (además del 80):

```apache
Listen 80
Listen 8080
```

![Directiva Listen 8080 agregada](images/c04-listen-8080.jpg)

Y un segundo bloque `VirtualHost`, para ese puerto:

```apache
<VirtualHost *:80>
  ServerName hola.localhost
  DocumentRoot "/var/www/html/hola"
  ServerAdmin root@localhost
</VirtualHost>

<VirtualHost *:8080>
  ServerName estudiante.localhost
  DocumentRoot "/var/www/html/estudiante"
  ServerAdmin root@localhost
</VirtualHost>
```

![Bloques VirtualHost para los puertos 80 y 8080](images/c05-virtualhost-ambos.jpg)

**12.** Agregamos también el segundo dominio a `/etc/hosts`:

```text
127.0.0.1   hola.localhost
127.0.0.1   estudiante.localhost
```

![/etc/hosts con hola.localhost y estudiante.localhost](images/c06-etc-hosts-ambos.jpg)

**13.** Reiniciamos el servicio con `systemctl restart httpd` e ingresamos al navegador: ambos sitios funcionan de forma independiente, cada uno en su propio puerto.

![Ambos sitios funcionando en el navegador](images/c07-navegador-ambos-sitios.jpg)

> [!TIP]
> Esta técnica, donde un mismo servidor atiende varios sitios distintos (cada uno con su propio `ServerName` y `DocumentRoot`), se llama **hosting virtual** (*virtual hosting*). Aquí se diferencian por **puerto** (80 y 8080); en un servidor real orientado a Internet, lo más común es diferenciarlos por **nombre de dominio**, todos escuchando en el mismo puerto 80 o 443, y que Apache elija el sitio correcto según el encabezado `Host` de cada solicitud.

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| Falla la instalación de paquetes de EPEL | CodeReady Builder no estaba habilitado antes | Habilitar CRB y volver a ejecutar `dnf update -y` antes de instalar EPEL |
| `freshclam` no descarga la base de firmas | El servicio no está activo, o no hay conexión al mirror | Verificar `systemctl status clamav-freshclam` y la conectividad de red |
| El script de escaneo no se ejecuta a la hora programada | Falta el permiso de ejecución, o la sintaxis de `cron` es incorrecta | Confirmar `chmod +x` sobre el script y revisar la expresión con `crontab -l` |
| No se encuentra el archivo de log | La ruta del script no coincide con la consultada | Verificar que `LOGFILE` en el script sea la misma ruta usada en `cat` |
| `freshclam.log` no se puede escribir | Permisos insuficientes sobre el archivo | Revisar `chmod 600` y `chown root` sobre `/var/log/freshclam.log` |

---

[← Volver al laboratorio](../README.md)
