[← Volver al laboratorio](../README.md)

# Examen Final · Cumplimiento PCI-DSS con OpenSCAP y Nessus

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![OpenSCAP](https://img.shields.io/badge/OpenSCAP-SCAP%20Workbench-f472b6?style=flat-square)
![Nessus](https://img.shields.io/badge/Tenable-Nessus-60a5fa?style=flat-square)
![PCI-DSS](https://img.shields.io/badge/Compliance-PCI--DSS-8b5cf6?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Examen final de la materia de Seguridad de Sistemas Operativos: implementación de **OpenSCAP** para auditar el cumplimiento de un servidor RHEL con el estándar **PCI-DSS**, uso de **Nessus** como segunda capa de verificación de vulnerabilidades, y remediación automatizada y manual de los hallazgos encontrados.

## Introducción

**OpenSCAP** es una herramienta de código abierto para automatizar la auditoría de seguridad y la gestión de vulnerabilidades en sistemas Linux. Se apoya en el estándar **SCAP** (*Security Content Automation Protocol*), que permite evaluar de forma estructurada si un sistema cumple con las políticas y configuraciones de seguridad recomendadas.

Revisar manualmente cada parámetro de un servidor para detectar fallos o malas configuraciones es ineficiente. OpenSCAP resuelve esto escaneando el equipo, comparándolo contra normativas de seguridad reconocidas (como PCI-DSS) y generando reportes visuales que indican qué está correctamente configurado y qué necesita ajustarse.

## Objetivos

| Punto | Descripción |
|:--:|---|
| **A y B** | Instalar OpenSCAP en un servidor Linux y en un equipo cliente Linux |
| **C** | Escanear el equipo cliente con Nessus en busca de vulnerabilidades; si no las hay, provocarlas |
| **D y E** | Inicializar SCAP Workbench en el servidor, elegir el perfil **PCI-DSS**, escanear el cliente y aplicar los parches de seguridad requeridos |
| **F** | Escanear nuevamente con Nessus y OpenSCAP, y exportar el reporte de vulnerabilidades |

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux), servidor y cliente por separado |
| Herramientas | OpenSCAP + SCAP Workbench, Nessus Essentials |
| Perfil de cumplimiento | PCI-DSS v4.0.1 Control Baseline for Red Hat Enterprise Linux 8 |
| Contenido SCAP | `ssg-rhel8-ds.xml` (SCAP Security Guide para RHEL 8) |
| IP del cliente | `192.168.0.7` |
| Acceso remoto para el escaneo | SSH (puerto 22), con usuario `root` |

> [!NOTE]
> Todo el proceso, incluida la creación deliberada de vulnerabilidades, se realizó en un entorno de laboratorio aislado y con fines académicos.

## Índice

- [Parte A y B: Instalar OpenSCAP en servidor y cliente](#parte-a-y-b-instalar-openscap-en-servidor-y-cliente)
- [Parte C: Primer escaneo de vulnerabilidades con Nessus](#parte-c-primer-escaneo-de-vulnerabilidades-con-nessus)
- [Parte D: SCAP Workbench y primer escaneo con el perfil PCI-DSS](#parte-d-scap-workbench-y-primer-escaneo-con-el-perfil-pci-dss)
- [Parte E: Provocar vulnerabilidades y remediar](#parte-e-provocar-vulnerabilidades-y-remediar)
- [Parte F: Escaneo final y comparación de resultados](#parte-f-escaneo-final-y-comparación-de-resultados)
- [Análisis de vulnerabilidades no corregidas y remediación manual](#análisis-de-vulnerabilidades-no-corregidas-y-remediación-manual)
- [Conclusión de la herramienta y del examen](#conclusión-de-la-herramienta-y-del-examen)
- [Referencia de comandos](#referencia-de-comandos)
- [Demostración en video](#demostración-en-video)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A y B: Instalar OpenSCAP en servidor y cliente

**1.** En el equipo **servidor**, abrimos la terminal en modo de superusuario y ejecutamos:

```bash
dnf install -y openscap scap-workbench
```

`scap-workbench` instala OpenSCAP con interfaz gráfica (GUI).

![Comando dnf install openscap scap-workbench en el servidor](images/a01-dnf-install-servidor.jpg)

![Instalación completada en el servidor](images/a02-instalacion-completa.jpg)


**2.** En el equipo **cliente**, repetimos exactamente el mismo procedimiento.

> [!NOTE]
> OpenSCAP se instala en ambos equipos: en el servidor se usa para ejecutar los escaneos desde la interfaz gráfica de SCAP Workbench, y en el cliente queda disponible como motor local de evaluación, al que el servidor se conecta de forma remota por SSH.

---

## Parte C: Primer escaneo de vulnerabilidades con Nessus

**1.** En el servidor, abrimos el navegador Firefox y entramos a Nessus.

![Navegador Firefox en el equipo servidor](images/c01-firefox-inicio.jpg)


**2.** Ingresamos las credenciales e iniciamos sesión.

![Pantalla de inicio de sesión de Nessus Essentials](images/c02-nessus-login.jpg)


**3.** Podemos descartar las alertas de la licencia de prueba.

**4.** Nessus da la bienvenida y ofrece la opción de escanear un equipo directamente desde ahí, pero en este caso se crea el escaneo manualmente. Cerramos el recuadro.

![Pantalla de bienvenida de Nessus](images/c03-nessus-bienvenida.jpg)


**5.** Hacemos clic en el botón azul **New scan**.

![My Scans sin escaneos creados](images/c04-my-scans-vacio.jpg)


**6.** Seleccionamos el tipo de escaneo **Basic Network Scan**.

![Selección de plantilla Basic Network Scan](images/c05-plantillas-scan.jpg)


**7.** Asignamos un nombre al escaneo. En **Targets**, especificamos la IP del cliente. Para obtenerla, en la terminal del cliente ejecutamos:

```bash
ifconfig
```

![Configuración inicial del nuevo escaneo](images/c06-nuevo-scan-config.jpg)

![ifconfig en el cliente: IP 192.168.0.7](images/c07-ifconfig-cliente.jpg)


La dirección es `192.168.0.7`.

**8.** Volvemos al servidor y establecemos esa IP como *target* del escaneo.

![Dirección IP del cliente establecida como target](images/c08-target-ip.jpg)


**9.** Vamos a la pestaña **Credentials** y seleccionamos **SSH**, para conectarnos de forma remota por el puerto 22.

![Pestaña Credentials sin configurar](images/c09-credentials-vacio.jpg)


**10.** En el método de autenticación seleccionamos **password**. En **Username** colocamos `root` (ya que se trabaja en modo superusuario) y la contraseña correspondiente.

![Credenciales SSH: usuario root y contraseña (oculta)](images/c10-credenciales-ssh.jpg)


**11.** Nos desplazamos hacia abajo y guardamos la configuración con **Save**.

![Desplazamiento hacia el botón Save](images/c11-guardar-escaneo.jpg)


**12.** Con el escaneo guardado, lo ejecutamos.

![Escaneo guardado: primer-escaneo-cliente](images/c12-escaneo-guardado.jpg)

![Escaneo en ejecución](images/c13-escaneo-en-progreso.jpg)


**13.** Esperamos a que finalice.

---

## Parte D: SCAP Workbench y primer escaneo con el perfil PCI-DSS

**1.** En el servidor, inicializamos SCAP Workbench (interfaz gráfica) con:

```bash
scap-workbench
```

![Comando scap-workbench en el servidor](images/d01-scap-workbench-comando.jpg)


**2.** Al abrirse, se muestra el **Open SCAP Security Guide**, donde elegimos la versión de contenido correspondiente a nuestro RHEL (en este caso, **RHEL8**) y hacemos clic en **Load Content**.

![Selección del contenido a cargar (RHEL8)](images/d02-scap-security-guide.jpg)

![Guide to the Secure Configuration of RHEL 8 cargada](images/d03-guia-cargada.png)


> [!TIP]
> Con SCAP Workbench abierto, también es posible ver en la terminal (CLI) cómo se ejecutan los procedimientos en segundo plano mientras se usa la interfaz gráfica.

**3.** Elegimos el perfil **PCI-DSS** en el desplegable **Profile** (el contenido carga, por defecto, con otro perfil preseleccionado, que se cambia aquí).

![Selección del perfil PCI-DSS en la lista de perfiles](images/d04-perfil-pci-dss.jpg)


**4.** Elegimos el método de conexión **SSH / Remote Machine**.

![Target: Remote Machine (over SSH)](images/d05-remote-machine-ssh.png)


**5.** Especificamos el usuario y el host (`root@192.168.0.7`) y marcamos la opción de superusuario.

![Usuario y host remoto: root@192.168.0.7](images/d06-user-host.png)


**6.** Hacemos clic en **Scan**.

**7.** Como es la primera conexión SSH a este equipo, se solicita confirmar el *fingerprint* del cliente. Confirmamos.

![Confirmación del fingerprint ECDSA del cliente](images/d07-fingerprint-ssh.jpg)


**8.** Se solicita la contraseña del equipo cliente. La ingresamos y confirmamos.

![Solicitud de la contraseña del cliente (oculta)](images/d08-password-ssh.jpg)


**9.** Esperamos a que finalicen tanto el escaneo de Nessus como el de OpenSCAP.

![Registro de la conexión y el escaneo en curso](images/d09-escaneo-en-curso.jpg)


**10.** Con ambos escaneos finalizados, revisamos primero los resultados de **Nessus**, abriendo el escaneo guardado (*primer-escaneo-cliente*). Se muestra la variación de vulnerabilidades encontradas y una gráfica circular con el porcentaje por categoría.

**11.** Para los resultados de **SCAP**, hacemos clic en **Save Results**.

![Save Results: exportar como HTML Report](images/d10-guardar-resultados.png)


**12.** Lo guardamos como **HTML Report**, eligiendo la ruta de destino (en este caso, la carpeta *Downloads*).

**13.** Una vez guardado, lo ubicamos en el explorador de archivos (*Files*) y lo abrimos con doble clic.

**14.** Revisamos los detalles del reporte, desplazándonos hacia abajo.

![Encabezado del OpenSCAP Evaluation Report](images/d11-reporte-openscap.jpg)

![Compliance and Scoring del primer escaneo](images/d12-compliance-scoring.jpg)


---

## Parte E: Provocar vulnerabilidades y remediar

> [!NOTE]
> Este paso es **opcional** dentro de la práctica: se realiza quando el primer escaneo no arroja suficientes vulnerabilidades que permitan demostrar el proceso completo de detección y remediación.

**1.** En el equipo cliente, creamos un script con extensión `.sh`:

```bash
nano vunerabilidades.sh
```

Antes de poder editar el archivo, hubo que resolver el acceso SSH entre ambos equipos: los primeros intentos de conexión fallaron por contraseña, hasta ajustar `/etc/ssh/sshd_config` y reiniciar el servicio.

![Intentos fallidos de conexión SSH antes de ajustar sshd_config](images/e01-intentos-ssh-fallidos.jpg)


**2.** El contenido del script, de forma intencional, desactiva el *firewall* de Linux, deshabilita las actualizaciones automáticas, e instala y deja activo el servicio **Telnet** (un protocolo de administración remota sin cifrar, considerado inseguro), entre otras acciones. El objetivo es generar vulnerabilidades adicionales detectables por los escáneres:

```bash
#!/bin/bash

echo "funcionando script"

echo "[1] Desactivando firewalld..."
systemctl stop firewalld
systemctl disable firewalld

echo "[2] Deshabilitando actualizaciones automáticas..."
systemctl disable --now dnf-automatic.timer

echo "[3] Instalando y activando telnet-server..."
dnf install -y telnet-server
systemctl enable telnet-server

echo "[4] Abriendo puertos innecesarios manualmente en firewall (si estuviera activo)..."
```

![Inicio del script vunerabilidades.sh](images/e02-script-inicio.jpg)

![Contenido completo del script: firewall, actualizaciones, telnet](images/e03-script-completo.jpg)


**3.** Le damos permisos de ejecución:

```bash
chmod +x vunerabilidades.sh
```

**4.** Lo ejecutamos y esperamos:

```bash
./vunerabilidades.sh
```

![Resultado de la ejecución del script en el cliente](images/e04-script-ejecutado.jpg)


**5.** Reiniciamos el servicio de *firewall* para que se apliquen los cambios.

**6.** En el servidor, realizamos un **segundo escaneo**, tanto con Nessus como con OpenSCAP, repitiendo el mismo procedimiento del primero.

> [!CAUTION]
> Para repetir el escaneo en SCAP Workbench tras usar el botón **Clear**, es importante confirmar que el **Profile** siga en **PCI-DSS** antes de hacer clic en **Scan**: al limpiar los resultados, el contenido puede volver a mostrar su perfil por defecto. Ejecutar el escaneo con el perfil equivocado invalidaría la comparación con el primer escaneo.

**7.** Esperamos a que termine y revisamos los resultados por ambas herramientas. Hay más vulnerabilidades que en el primer escaneo, como resultado directo del script ejecutado; incluso aparece una vulnerabilidad de categoría **Medium** que no estaba presente antes.

### Generar y aplicar la remediación

**8.** Para aplicar los parches de seguridad requeridos por PCI-DSS, volvemos a SCAP Workbench y hacemos clic en **Generate remediation role**, eligiendo el formato **bash**. Esto crea un script con las soluciones a las vulnerabilidades detectadas.

![Generate remediation role: formato bash](images/f01-generar-remediacion.jpg)


**9.** Elegimos dónde guardarlo (en este caso, la carpeta *Documents*) y confirmamos con **Save**.

![Confirmación: remediation.sh guardado en Documents](images/f02-remediacion-guardada.jpg)


**10.** Verificamos el archivo en el servidor:

```bash
cd Documents
ls
```

![Verificación del archivo remediation.sh](images/f03-ls-remediation.jpg)


**11.** Como el script debe ejecutarse en el equipo **cliente**, lo transferimos con `scp` (*secure copy*, incluido por defecto en Linux):

```bash
scp remediation.sh root@192.168.0.7:/home/Neury0829/Desktop
```

![Transferencia con scp hacia el cliente](images/f04-scp-transferencia.jpg)


**12.** En el equipo cliente, ejecutamos el script con privilegios de superusuario:

```bash
sudo bash remediation.sh
```

![Ejecución de remediation.sh en el cliente](images/f05-aplicar-remediacion.jpg)


**13.** Esperamos a que se apliquen los parches de seguridad sugeridos por el perfil PCI-DSS. El script remedia las reglas una por una (*"Remediating rule 1/133..."*, y así sucesivamente).

---

## Parte F: Escaneo final y comparación de resultados

**1.** Realizamos un **tercer escaneo** por ambas herramientas, replicando el mismo procedimiento de los anteriores, tanto en Nessus...

![Lista de escaneos en Nessus tras el segundo escaneo](images/f06-lista-escaneos.jpg)


...como en SCAP Workbench.

**2.** Esperamos a que ambos finalicen y revisamos los resultados.

### Comparación con Nessus

Prácticamente todas las vulnerabilidades se corrigieron: se pasó de tener vulnerabilidades mixtas, de categorías Medium y Low, a solo 5 hallazgos de categoría **Info** (la más baja, puramente informativa).

### Comparación con OpenSCAP

| Escaneo | Momento | Score de cumplimiento |
|---|---|:--:|
| 1 | Estado inicial del servidor | 65.51 |
| 2 | Tras ejecutar el script de vulnerabilidades (sin remediar) | *(disminuyó respecto al inicial)* |
| 3 | Tras aplicar el script de remediación de SCAP Workbench | 55.49 |

> [!CAUTION]
> **El puntaje de OpenSCAP bajó entre el primer y el tercer escaneo (de 65.51 a 55.49), en vez de subir.** Esto, pese a que Nessus sí mostró una mejora clara. Una posible explicación es que la remediación automática haya desinstalado servicios que OpenSCAP considera necesarios para el cumplimiento, o haya modificado dependencias de red que ahora impiden verificar correctamente ciertas reglas. La remediación automática de SCAP Workbench es un buen punto de partida, pero en RHEL 8 con perfiles PCI-DSS, lograr el 100% de cumplimiento suele requerir intervención manual adicional en el *bootloader* (GRUB) y en el particionado de los registros (`/var/log`, `/var/log/audit`), ya que los scripts de remediación automática no suelen modificar la tabla de particiones por motivos de seguridad.

Como quedaron vulnerabilidades sin corregir, a continuación se detallan y se ofrece una solución para cada una.

---

## Análisis de vulnerabilidades no corregidas y remediación manual

### Integridad del software (AIDE)

La sección *"Software Integrity Checking"* presenta fallas. PCI-DSS exige verificar la integridad de los archivos del sistema para detectar cambios no autorizados.

**Vulnerabilidad:** el sistema no tiene instalado o configurado el motor de verificación de integridad avanzada (**AIDE**).

**Solución:**
```bash
sudo dnf install aide -y
sudo aide --init
sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```
Y programar una revisión periódica mediante un *cron job* o un *timer* de systemd.

### Verificación de integridad con RPM

Aparece como falla en el reporte: indica que hay archivos de paquetes instalados que no coinciden con sus metadatos originales (permisos, *hashes*, etc.).

**Vulnerabilidad:** archivos binarios o de configuración del sistema alterados fuera del gestor de paquetes.

**Solución:**
```bash
# Para identificar los archivos afectados:
rpm -Va

# Si los cambios fueron accidentales, reinstalar el paquete afectado:
sudo dnf reinstall <nombre_del_paquete>
```

### Configuración de auditoría (auditd)

PCI-DSS es sumamente estricto con los registros de auditoría. Es probable que las fallas de *"System Settings"* incluyan parámetros de `auditd`.

**Vulnerabilidad:** el demonio de auditoría no está configurado para registrar eventos críticos (como cambios de privilegios o acceso a archivos sensibles), o el tamaño del log es insuficiente.

**Solución:**
```bash
sudo systemctl enable --now auditd
```
Configurar el tamaño máximo del log en `/etc/audit/auditd.conf`:
```text
max_log_file = 100
```
Y cargar las reglas base de PCI-DSS en `/etc/audit/rules.d/pci-dss.rules`.

### Endurecimiento del acceso SSH

Casi todos los fallos en perfiles de seguridad altos en RHEL 8 involucran el archivo `/etc/ssh/sshd_config`.

**Vulnerabilidad:** uso de protocolos antiguos, falta de *banners* de advertencia o tiempos de inactividad demasiado altos.

**Solución:** editar `/etc/ssh/sshd_config` y asegurar:
```text
PermitRootLogin no
MaxAuthTries 4
ClientAliveInterval 900
Banner /etc/issue
```
Y reiniciar el servicio:
```bash
sudo systemctl restart sshd
```

### Parámetros del kernel (sysctl)

Las reglas de red suelen fallar si no se desactivan funciones como el reenvío de paquetes o los redireccionamientos ICMP.

**Vulnerabilidad:** configuraciones de red inseguras que permiten ataques de intermediario (*Man-in-the-Middle*) o redireccionamiento de tráfico.

**Solución:** crear o editar `/etc/sysctl.d/99-hardening.conf` y añadir:
```ini
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
```
Y aplicar los cambios:
```bash
sudo sysctl --system
```

---

## Conclusión de la herramienta y del examen

Un aspecto importante a considerar sobre Nessus y OpenSCAP: OpenSCAP realiza análisis más profundos y en menor tiempo que Nessus, además de permitir descargar un reporte completo de forma gratuita, mientras que en Nessus ese mismo beneficio requiere una versión de pago. Este examen permitió comprobar que asegurar un sistema bajo estándares estrictos como PCI-DSS no tiene por qué ser un proceso manual propenso a errores: mediante OpenSCAP se logró automatizar buena parte de la auditoría y corrección de un equipo cliente, pasando de un estado vulnerable (confirmado inicialmente por Nessus) a uno que cumple en gran medida con las normativas exigidas para el manejo de datos de tarjetas de pago.

Un punto crítico del proceso fue la validación de la conectividad: una configuración incorrecta entre servidor y cliente puede arrojar resultados "N/A", lo que invalidaría cualquier análisis. Asegurar que SCAP Workbench interprete correctamente los perfiles es tan importante como el escaneo mismo (como se vio con el perfil por defecto al usar **Clear**). Además, usar Nessus como segunda capa de verificación fue clave para contrastar resultados y confirmar que los parches aplicados por OpenSCAP realmente redujeron la superficie de ataque del sistema, incluso en el caso de esta práctica, donde el *score* de OpenSCAP bajó pese a que Nessus mostró una mejora clara.

Finalmente, aunque OpenSCAP es una herramienta potente para la remediación automatizada, no es una solución absoluta: algunas vulnerabilidades detectadas por Nessus pueden persistir porque requieren intervenciones manuales específicas o cambios de arquitectura que el estándar PCI-DSS no corrige de forma automática. Esto confirma la necesidad de que el administrador de seguridad mantenga un monitoreo constante y no dependa exclusivamente de la automatización para sostener la integridad de la infraestructura.

---

## Referencia de comandos

| Comando / herramienta | Función |
|---|---|
| `dnf install -y openscap scap-workbench` | Instala OpenSCAP con interfaz gráfica |
| `scap-workbench` | Inicia la interfaz gráfica de SCAP Workbench |
| `ifconfig` | Muestra la dirección IP de un equipo |
| Nessus → *New Scan* → *Basic Network Scan* | Crea un escaneo de vulnerabilidades |
| Nessus → *Credentials* → SSH | Configura un escaneo autenticado por SSH |
| SCAP Workbench → *Load Content* | Carga el contenido SCAP (p. ej. `ssg-rhel8-ds.xml`) |
| SCAP Workbench → *Profile* | Selecciona el perfil de cumplimiento (p. ej. PCI-DSS) |
| SCAP Workbench → *Remote Machine (SSH)* | Configura el escaneo remoto por SSH |
| SCAP Workbench → *Save Results* (HTML Report) | Exporta el reporte de evaluación |
| SCAP Workbench → *Generate remediation role* (bash) | Genera un script con las soluciones a las vulnerabilidades detectadas |
| `scp <archivo> usuario@ip:ruta` | Transfiere un archivo de forma segura a otro equipo |
| `sudo bash remediation.sh` | Aplica el script de remediación en el equipo objetivo |
| `rpm -Va` | Verifica la integridad de los paquetes instalados |
| `sudo aide --init` | Inicializa la base de datos de integridad de AIDE |

---
## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del examen. Las capturas del procedimiento se encuentran en la carpeta `images/`.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=XeZdCIBIb3E)

---
## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| El escaneo de SCAP Workbench devuelve resultados "N/A" | Falla de conectividad o de autenticación entre servidor y cliente | Verificar la IP, el acceso SSH y las credenciales antes de escanear |
| La conexión SSH es rechazada varias veces seguidas | Contraseña incorrecta, o la configuración de `sshd_config` restringe el acceso | Revisar `/etc/ssh/sshd_config` en el cliente y reiniciar `sshd` tras cualquier cambio |
| El script `.sh` no se ejecuta con `./archivo.sh` | Falta el permiso de ejecución, o la ruta no es la esperada | Confirmar `chmod +x` y ejecutar desde el directorio correcto (evitar `cd` a un directorio inexistente) |
| El segundo o tercer escaneo de SCAP se ejecuta con el perfil equivocado | El botón **Clear** puede restablecer el perfil por defecto | Confirmar el **Profile** seleccionado (PCI-DSS) antes de cada clic en **Scan** |
| `remediation.sh` no se encuentra al ejecutarlo en el cliente | El archivo no estaba en el directorio desde el que se ejecutó el comando | Verificar la ruta de destino del `scp` y hacer `cd` al directorio correcto antes de ejecutar |
| El puntaje de OpenSCAP no mejora tras la remediación automática | El script de remediación pudo afectar servicios o dependencias necesarias para otras reglas | Revisar manualmente las reglas que siguen en falla y aplicar los ajustes específicos (AIDE, auditd, SSH, sysctl, RPM) |

---

[← Volver al laboratorio](../README.md)
