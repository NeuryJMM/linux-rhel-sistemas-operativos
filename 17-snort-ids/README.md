[← Volver al laboratorio](../README.md)

# 17 · Detección de intrusos con Snort 3 (IDS)

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![Snort](https://img.shields.io/badge/Snort-3-f472b6?style=flat-square)
![IDS](https://img.shields.io/badge/IDS-Detecci%C3%B3n%20de%20intrusos-60a5fa?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Instalación de **Snort 3**, un sistema de detección de intrusos (IDS) de código abierto, compilado desde su código fuente en RHEL, con reglas personalizadas que generan una alerta cada vez que se detecta tráfico ICMP, FTP, SSH o HTTP dirigido al servidor.

## Objetivo

Compilar e instalar Snort 3 con sus dependencias, definir la red a proteger, escribir reglas de detección propias y ponerlo a monitorear una interfaz de red en tiempo real.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL 8 (Red Hat Enterprise Linux) |
| Software | Snort 3, compilado desde el código fuente (`snort3` y `libdaq` de GitHub) |
| Ruta de instalación | `/usr/local/snort` |
| Interfaz monitoreada | `ens160` |
| Red protegida (`HOME_NET`) | `192.168.0.16/24` |
| Modo de operación | IDS: solo **detecta y alerta**, no bloquea tráfico |

> [!NOTE]
> **IDS frente a IPS.** Un **IDS** (*Intrusion Detection System*) observa el tráfico y avisa cuando algo coincide con una regla, pero no lo detiene. Un **IPS** (*Intrusion Prevention System*) va un paso más allá y puede bloquearlo en línea. Snort 3 puede funcionar en ambos modos; en esta práctica se usa como IDS.

## Índice

- [Parte A: Preparar el sistema](#parte-a-preparar-el-sistema)
- [Parte B: Instalar las dependencias](#parte-b-instalar-las-dependencias)
- [Parte C: Compilar e instalar Snort 3](#parte-c-compilar-e-instalar-snort-3)
- [Parte D: Configurar la red a proteger](#parte-d-configurar-la-red-a-proteger)
- [Parte E: Crear las reglas de detección](#parte-e-crear-las-reglas-de-detección)
- [Parte F: Ejecutar Snort y verificar las alertas](#parte-f-ejecutar-snort-y-verificar-las-alertas)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Preparar el sistema

**1.** Actualizamos el sistema:

```bash
sudo dnf update -y
```

**2.** Habilitamos los repositorios que contienen varias de las dependencias: **EPEL** (*Extra Packages for Enterprise Linux*) y **CodeReady Builder** (herramientas de desarrollo):

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y
sudo subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms
```

> [!NOTE]
> El repositorio CodeReady Builder requiere una **suscripción activa de Red Hat**. En derivadas de RHEL sin suscripción, como Rocky Linux, se puede obtener un repositorio equivalente (llamado *CRB* o *PowerTools*) sin ese requisito.

**3.** Indicamos al sistema dónde buscar las bibliotecas que se instalarán manualmente en `/usr/local`, creando un archivo de configuración para el enlazador dinámico:

```bash
sudo nano /etc/ld.so.conf.d/local.conf
```

```text
/usr/local/lib
/usr/local/lib64
```

**4.** Guardamos, cerramos y actualizamos la caché de bibliotecas:

```bash
sudo ldconfig
```

> [!TIP]
> Esto es necesario porque `/usr/local/lib` no siempre está en la ruta de búsqueda de bibliotecas compartidas de RHEL. Sin él, Snort compilaría bien pero fallaría al ejecutarse con errores del tipo `error while loading shared libraries`.

---

## Parte B: Instalar las dependencias

**1.** Instalamos las herramientas de compilación y las bibliotecas de desarrollo desde los repositorios:

```bash
sudo dnf install flex bison gcc gcc-c++ make cmake autoconf libtool git nano unzip wget \
libpcap-devel pcre-devel hwloc-devel openssl-devel zlib-devel luajit-devel \
pkgconfig pkgconf libunwind-devel libnfnetlink-devel libnetfilter_queue-devel \
libmnl-devel xz-devel gperftools libuuid-devel hyperscan hyperscan-devel -y
```

| Dependencia | Para qué la usa Snort |
|---|---|
| `libpcap` | Captura de paquetes de la red |
| `luajit` | Snort 3 se configura con archivos en lenguaje Lua |
| `hyperscan` | Motor de coincidencia de patrones de alto rendimiento (Intel) |
| `pcre` / `pcre2` | Expresiones regulares en las reglas |
| `gperftools` (`tcmalloc`) | Gestor de memoria optimizado |
| `openssl`, `zlib`, `xz` | Cifrado y compresión |

**2.** Instalamos **libdnet** (utilidades de red de bajo nivel) desde su repositorio de GitHub, ya que no está disponible en los repositorios oficiales:

```bash
cd ~
git clone https://github.com/dugsong/libdnet.git
cd libdnet
./configure --without-check
make
sudo make install
sudo ldconfig
```

**3.** Instalamos **libdaq** (*Data Acquisition Library*), la biblioteca que usa Snort para capturar paquetes de la red, también clonando su repositorio:

```bash
git clone https://github.com/snort3/libdaq.git
cd libdaq && ./bootstrap && ./configure && make && sudo make install
sudo ldconfig && cd ..
```

**4.** Instalamos **PCRE2** y su paquete de desarrollo, necesarios para procesar expresiones regulares:

```bash
sudo dnf install pcre2 pcre2-devel -y
```

---

## Parte C: Compilar e instalar Snort 3

**1.** Clonamos el repositorio de Snort 3:

```bash
git clone https://github.com/snort3/snort3.git
cd snort3
```

**2.** Indicamos dónde buscar las bibliotecas recién instaladas y las opciones de compilación (optimización y exclusión de información de tipos en tiempo de ejecución, que reduce el tamaño del binario):

```bash
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH
export PKG_CONFIG_PATH=/usr/local/lib64/pkgconfig:$PKG_CONFIG_PATH
export CFLAGS="-O3"
export CXXFLAGS="-O3 -fno-rtti"
```

**3.** Configuramos la compilación, indicando el directorio de instalación y habilitando `tcmalloc`:

```bash
./configure_cmake.sh --prefix=/usr/local/snort --enable-tcmalloc
```

**4.** Compilamos e instalamos, usando todos los núcleos del procesador (`nproc`) para acelerar el proceso:

```bash
cd build/ && make -j$(nproc) && sudo make -j$(nproc) install
cd ../.. && sudo ldconfig
```

> [!NOTE]
> La compilación de Snort 3 es larga (puede tardar varios minutos, según los núcleos y la memoria asignados a la máquina virtual). Es normal que la terminal permanezca ocupada un buen rato.

**5.** Creamos un enlace simbólico del binario hacia una ruta que ya está en el `PATH`, para poder ejecutar `snort` desde cualquier lugar:

```bash
sudo ln -s /usr/local/snort/bin/snort /usr/bin/snort
```

**6.** Verificamos la instalación:

```bash
snort -V
```

Debe mostrar la versión de Snort y la de las bibliotecas que utiliza (`DAQ`, `LuaJIT`, `PCRE`, etc.).

---

## Parte D: Configurar la red a proteger

**1.** Abrimos el archivo de configuración principal de Snort (escrito en Lua):

```bash
sudo nano /usr/local/snort/etc/snort/snort.lua
```

**2.** Buscamos la variable `HOME_NET` y la editamos con la dirección y máscara de nuestra red. Esta variable define qué se considera "la red interna" que Snort debe proteger, y es la que usan las reglas como destino:

```lua
HOME_NET = '192.168.0.16/24'
```

> [!TIP]
> Para indicar la red completa, la notación habitual es la dirección de red (`192.168.0.0/24`) y no la IP del servidor con su máscara. En esta práctica se usó la dirección del servidor con `/24`; conviene tenerlo en cuenta si se reutiliza la configuración en otra red.

**3.** Validamos la sintaxis del archivo de configuración (`-T` prueba la configuración sin ponerse a monitorear):

```bash
snort -T -c /usr/local/snort/etc/snort/snort.lua
```

Si todo está en orden, termina indicando que la validación fue exitosa.

**4.** Activamos el **modo promiscuo** en la tarjeta de red. En este modo, la interfaz procesa **todos** los paquetes que le llegan, y no solo los dirigidos a su propia dirección, que es lo que necesita un sistema de detección:

```bash
sudo ip link set dev ens160 promisc on
```

> [!NOTE]
> Este cambio **no es permanente**: se pierde al reiniciar la máquina o la interfaz. Además, en una red conmutada (con *switch*), el modo promiscuo solo permite ver el tráfico que el *switch* ya envía a esa interfaz (el propio y el de difusión); para observar el tráfico de otros equipos hace falta un puerto espejo (*port mirroring*).

---

## Parte E: Crear las reglas de detección

**1.** Abrimos el archivo de reglas locales:

```bash
sudo nano /usr/local/snort/etc/snort/local.rules
```

**2.** Agregamos cuatro reglas, que generan una alerta por cada paquete dirigido a la red protegida por ICMP o por los puertos de FTP, SSH y HTTP:

```text
alert icmp any any -> $HOME_NET any (msg:"ICMP Traffic Detected"; sid:10000001;)
alert tcp any any -> $HOME_NET 21 (msg:"FTP Traffic Detected on port 21"; sid:10000002;)
alert tcp any any -> $HOME_NET 22 (msg:"SSH Traffic Detected on port 22"; sid:10000003;)
alert tcp any any -> $HOME_NET 80 (msg:"HTTP Traffic Detected on port 80"; sid:10000004;)
```

Anatomía de una regla:

| Parte | Significado |
|---|---|
| `alert` | Acción: generar una alerta |
| `icmp` / `tcp` | Protocolo |
| `any any` | Cualquier IP y cualquier puerto de origen |
| `->` | Dirección del tráfico (de origen a destino) |
| `$HOME_NET 22` | Destino: la red protegida, puerto 22 |
| `msg:"..."` | Texto de la alerta |
| `sid:10000001` | Identificador único de la regla |

> [!NOTE]
> Los identificadores (`sid`) de las reglas personalizadas deben ser mayores o iguales a `1000000`, para no chocar con los de las reglas oficiales, que ocupan los rangos inferiores. Además, estas reglas solo inspeccionan el tráfico **dirigido hacia** `$HOME_NET`: las conexiones iniciadas desde el propio servidor hacia fuera no las activan.

---

## Parte F: Ejecutar Snort y verificar las alertas

**1.** Ejecutamos Snort con la configuración y las reglas creadas:

```bash
sudo snort -c /usr/local/snort/etc/snort/snort.lua \
-R /usr/local/snort/etc/snort/local.rules \
-i ens160 -A alert_fast -s 65535 -k none
```

| Opción | Función |
|---|---|
| `-c` | Archivo de configuración principal (`snort.lua`) |
| `-R` | Archivo de reglas a cargar |
| `-i ens160` | Interfaz de red a monitorear |
| `-A alert_fast` | Muestra cada alerta en una sola línea, en la consola |
| `-s 65535` | Tamaño máximo de captura por paquete (*snaplen*), para analizarlo completo |
| `-k none` | Desactiva la verificación de sumas de control (*checksums*) |

> [!TIP]
> `-k none` es habitual en máquinas virtuales: con la descarga de *checksums* a la tarjeta de red (*offloading*), los paquetes salientes pueden aparecer con sumas incorrectas al capturarlos, y Snort los descartaría. Desactivar la verificación evita falsos descartes, pero también significa que Snort no detectará paquetes con sumas de control manipuladas intencionalmente.

**2.** Con Snort en ejecución, generamos tráfico hacia el servidor desde otro equipo (por ejemplo, el host):

- `ping <IP-del-servidor>`, para activar la regla de ICMP.
- Abrir `http://<IP-del-servidor>` en un navegador, para la regla HTTP.
- Una conexión `ssh <usuario>@<IP-del-servidor>`, para la regla SSH.
- Una conexión FTP al puerto 21, para la regla FTP.

**3.** Cada paquete que coincide con una regla aparece en la consola como una alerta de una sola línea. Con el formato `alert_fast`, tiene esta forma (ejemplo ilustrativo):

```text
[**] [1:10000001:0] "ICMP Traffic Detected" [**] [Priority: 0] {ICMP} 192.168.0.9 -> 192.168.0.16
```

Se identifican el identificador de la regla, el mensaje, el protocolo y el origen y destino del tráfico detectado.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo de instalación y detección.

**Video de la práctica:** [Ver demostración en YouTube](https://youtu.be/ByXoL5YEQG8?si=PNLeWr4usH0o8ycU)

---

## Referencia de comandos

| Comando / archivo | Función |
|---|---|
| `/etc/ld.so.conf.d/local.conf` + `ldconfig` | Registra rutas de bibliotecas adicionales y actualiza la caché |
| `./configure_cmake.sh --prefix=... --enable-tcmalloc` | Configura la compilación de Snort 3 |
| `make -j$(nproc)` | Compila usando todos los núcleos disponibles |
| `snort -V` | Muestra la versión de Snort y de sus bibliotecas |
| `snort -T -c <snort.lua>` | Valida la configuración sin iniciar el monitoreo |
| `ip link set dev <interfaz> promisc on` | Activa el modo promiscuo de una interfaz de red |
| `/usr/local/snort/etc/snort/snort.lua` | Configuración principal de Snort (incluye `HOME_NET`) |
| `/usr/local/snort/etc/snort/local.rules` | Reglas personalizadas |
| `snort -c ... -R ... -i ... -A alert_fast` | Ejecuta Snort en modo de detección con las reglas indicadas |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| `error while loading shared libraries` al ejecutar `snort` | La ruta de las bibliotecas instaladas manualmente no está registrada | Revisar `/etc/ld.so.conf.d/local.conf` y ejecutar `sudo ldconfig` |
| `configure_cmake.sh` no encuentra `libdaq` | `PKG_CONFIG_PATH` no está configurado en esa sesión de terminal | Volver a ejecutar los `export PKG_CONFIG_PATH=...` antes de configurar (se pierden al cerrar la terminal) |
| Falla la configuración con `--enable-tcmalloc` | Falta el paquete de desarrollo de `gperftools` | Instalar `gperftools-devel`, o quitar la opción `--enable-tcmalloc` |
| `snort -T` informa errores de sintaxis | Error al editar `snort.lua`, por ejemplo comillas o valores mal escritos en `HOME_NET` | Revisar la línea indicada por el mensaje de error |
| Snort no muestra ninguna alerta | El tráfico no está dirigido a `HOME_NET`, la interfaz es incorrecta, o falta el modo promiscuo | Verificar el nombre de la interfaz con `ip a`, confirmar `HOME_NET` y generar tráfico hacia el servidor |
| Las alertas dejaron de aparecer tras reiniciar | El modo promiscuo no es permanente | Volver a activarlo con `ip link set dev ens160 promisc on` |
| `make` falla por falta de memoria | La máquina virtual tiene poca RAM para compilar con todos los núcleos | Usar `make -j2` (menos tareas en paralelo) o aumentar la memoria de la VM |

---

[← Volver al laboratorio](../README.md)
