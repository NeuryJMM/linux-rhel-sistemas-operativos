[← Volver al laboratorio](../README.md)

# 21 · Controlador de dominio Active Directory con Samba

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![Samba](https://img.shields.io/badge/Samba%204.16-Active%20Directory%20DC-f472b6?style=flat-square)
![Windows](https://img.shields.io/badge/Cliente-Windows-60a5fa?style=flat-square&logo=windows&logoColor=white)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Compilación e instalación de **Samba 4.16** desde su código fuente en RHEL, configurado como **controlador de dominio (DC) de Active Directory** completo (con su propio servidor DNS y Kerberos integrados), con un cliente Windows unido al dominio.

> [!NOTE]
> Esta práctica no usa el paquete `samba` de los repositorios de RHEL (ese paquete solo ofrece el rol de servidor de archivos, como en la [práctica 20](../20-samba-fileserver)). El rol de **controlador de dominio** requiere compilar Samba desde el código fuente con soporte de Active Directory (`--with-ads`), por eso el proceso es más largo.

## Objetivo

Levantar un dominio Active Directory (`OS3.inet`) sobre Samba, unir un cliente Windows a ese dominio, resolver el dominio por DNS en ambos sentidos, y crear un usuario de dominio funcional.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Samba | 4.16.2, compilado desde el código fuente |
| Nombre del controlador | `dc1.inet` |
| IP del servidor | `192.168.0.16` |
| Dominio | `OS3.inet` (NetBIOS: `OS3`) |
| Backend DNS | `SAMBA_INTERNAL` |
| Interfaz de red | `ens160` |
| Cliente Windows | `192.168.0.11` |
| Usuario de dominio creado | `lanegracubana` |

> [!WARNING]
> Esta configuración abre un rango amplio de puertos (incluido todo el rango RPC dinámico, 49152–65535) y en algún punto pone **SELinux en modo permisivo**. Es una forma común de salir de paso de un problema de permisos en un laboratorio, pero equivale a desactivar una capa completa de protección del sistema. Antes de repetir esta práctica en un entorno que no sea un laboratorio aislado, conviene investigar el motivo exacto del bloqueo (por ejemplo, con `ausearch -m avc -ts recent`) en vez de desactivar SELinux por completo.

## Índice

- [Parte A: Preparar el sistema](#parte-a-preparar-el-sistema)
- [Parte B: Compilar e instalar Samba](#parte-b-compilar-e-instalar-samba)
- [Parte C: Aprovisionar el dominio](#parte-c-aprovisionar-el-dominio)
- [Parte D: Iniciar Samba y probar el DNS](#parte-d-iniciar-samba-y-probar-el-dns)
- [Parte E: Kerberos y firewall](#parte-e-kerberos-y-firewall)
- [Parte F: Configurar el DNS del servidor](#parte-f-configurar-el-dns-del-servidor)
- [Parte G: Unir el cliente Windows al dominio](#parte-g-unir-el-cliente-windows-al-dominio)
- [Parte H: Crear un usuario de dominio](#parte-h-crear-un-usuario-de-dominio)
- [Parte I: Si el inicio de sesión sigue sin funcionar](#parte-i-si-el-inicio-de-sesión-sigue-sin-funcionar)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Preparar el sistema

**1.** Asignamos el nombre de host del servidor. **No debe coincidir con el nombre del dominio** que se creará más adelante (`OS3.inet`), o el aprovisionamiento fallará:

```bash
hostnamectl set-hostname dc1.inet
```

**2.** Registramos ese nombre en `/etc/hosts`, con la IP real del servidor:

```bash
echo "192.168.0.16 dc1.inet" >> /etc/hosts
```

**3.** Verificamos que el nombre haya quedado aplicado:

```bash
hostname
hostname -f
```

**4.** Actualizamos el sistema y habilitamos los repositorios con las herramientas de compilación (**EPEL** y **CodeReady Builder**):

```bash
dnf -y update
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y
sudo subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms
dnf update
sudo dnf makecache
dnf -y install wget tar gcc make python3-devel
```

---

## Parte B: Compilar e instalar Samba

**1.** Descargamos y extraemos el código fuente de Samba:

```bash
mkdir -p /samba && cd /samba
wget https://download.samba.org/pub/samba/stable/samba-4.16.2.tar.gz
tar -zxvf samba-4.16.2.tar.gz && cd samba-4.16.2/
```

**2.** Instalamos las bibliotecas de desarrollo necesarias para compilar con soporte de Active Directory:

```bash
sudo dnf -y install \
  docbook-style-xsl python3-markdown bison dbus-devel flex gdb \
  gnutls-devel jansson-devel keyutils-libs-devel krb5-workstation \
  libacl-devel libaio-devel libarchive-devel libattr-devel libblkid-devel \
  libtasn1 libtasn1-tools libxml2-devel libxslt lmdb-devel openldap-devel \
  pam-devel perl perl-ExtUtils-MakeMaker perl-Parse-Yapp popt-devel \
  python3-cryptography python3-dns python3-gpg readline-devel rpcgen \
  systemd-devel zlib-devel perl-JSON gpgme-devel screen
```

**3.** Configuramos, compilamos e instalamos:

```bash
./configure --prefix=/usr/local/samba --enable-fhs --with-ads --with-systemd
make -j$(nproc)
make install
```

| Opción | Significado |
|---|---|
| `--prefix=/usr/local/samba` | Instala en una ruta propia, separada del sistema |
| `--enable-fhs` | Organiza los archivos según el estándar de jerarquía de Linux (*Filesystem Hierarchy Standard*) |
| `--with-ads` | Habilita el soporte de Active Directory (indispensable para el rol de controlador de dominio) |
| `--with-systemd` | Integra Samba con `systemd` |

> [!NOTE]
> La compilación es larga (puede tardar bastante según los recursos de la VM). Es un paso que requiere paciencia, no intervención.

**4.** Agregamos los binarios de Samba al `PATH` de la sesión actual:

```bash
export PATH=/usr/local/samba/bin:/usr/local/samba/sbin:$PATH
```

**5.** Para que quede disponible en cada sesión, lo agregamos también a `~/.bash_profile`:

```bash
nano ~/.bash_profile
```

```bash
PATH=$PATH:$HOME/bin:/usr/local/samba/bin:/usr/local/samba/sbin:$PATH
export PATH
```

> [!NOTE]
> Si el archivo ya tenía una línea `PATH=...`, se reemplaza por la de arriba, en vez de duplicarla.

---

## Parte C: Aprovisionar el dominio

**1.** Antes de ejecutar el siguiente comando, verificamos el nombre real de la interfaz de red con `ip a` (en este caso, `ens160`) y lo usamos en `--option="interfaces=..."`.

**2.** Aprovisionamos el dominio:

```bash
samba-tool domain provision --use-rfc2307 --interactive --option="interfaces= lo ens160" --option="bind interfaces only=yes"
```

El asistente pide, de forma interactiva:

| Campo | Valor usado |
|---|---|
| Dominio (*realm*) | `OS3.inet` |
| Nombre de dominio (NetBIOS) | `OS3` |
| Rol del servidor | Controlador de dominio (DC) |
| Backend DNS | `SAMBA_INTERNAL` |
| Contraseña de administrador | Mínimo 7 caracteres, cumpliendo los requisitos de complejidad de Windows |

> [!TIP]
> `--use-rfc2307` agrega los atributos de identidad de Unix (UID, GID, *shell*) al esquema del directorio, útil si más adelante se quiere usar el mismo dominio para autenticar también clientes Linux.

---

## Parte D: Iniciar Samba y probar el DNS

**1.** Iniciamos el servicio:

```bash
samba
```

> [!NOTE]
> Tras un reinicio del servidor, este comando por sí solo ya no alcanza (el `PATH` de una sesión nueva puede no tenerlo registrado hasta que se vuelva a cargar `~/.bash_profile`). En ese caso, se inicia con la ruta completa: `sudo /usr/local/samba/sbin/samba`.

**2.** Probamos la resolución de nombres, tanto hacia Internet como hacia el propio dominio:

```bash
ping google.com
ping OS3.inet
ping dc1.inet
```

> [!NOTE]
> Si `OS3.inet` no responde desde el propio Linux, se prueba con el nombre de host (`dc1.inet`); lo importante es que `OS3.inet` sí resuelva correctamente más adelante, **desde el cliente Windows**.

---

## Parte E: Kerberos y firewall

**1.** Copiamos la configuración de Kerberos que generó Samba al aprovisionar el dominio, hacia la ubicación estándar del sistema:

```bash
cp /usr/local/samba/var/lib/samba/private/krb5.conf /etc/krb5.conf
```

**2.** Abrimos en el firewall los servicios y puertos que usa un controlador de dominio Active Directory (LDAP, Kerberos y DNS, entre otros):

```bash
firewall-cmd --permanent --add-service={ldap,ldaps,kerberos,dns}
firewall-cmd --permanent --add-port={53,88,135,139,389,445,464,636,3268,3269}/tcp
firewall-cmd --permanent --add-port={53,88,123,138,389,464}/udp
firewall-cmd --reload
```

---

## Parte F: Configurar el DNS del servidor

**1.** Hacemos que el propio servidor use su servidor DNS interno de Samba, en vez del que entregue DHCP:

```bash
sudo nmcli con mod ens160 ipv4.dns 192.168.0.16
sudo nmcli con mod ens160 ipv4.ignore-auto-dns yes
sudo nmcli con up ens160
```

> [!IMPORTANT]
> Este paso es imprescindible para que los clientes del dominio tengan salida a Internet: el servidor DNS de Samba debe poder resolver también nombres externos (reenviándolos), no solo los del dominio. Sin este cambio, los clientes configurados para usar el DNS del servidor pueden quedarse sin acceso a sitios externos.

**2.** Registramos un nombre adicional (`www`) dentro de la zona del dominio, apuntando a la IP del cliente Windows:

```bash
samba-tool dns add 192.168.0.16 OS3.inet www A 192.168.0.11 -U Administrator
```

---

## Parte G: Unir el cliente Windows al dominio

**1.** En la máquina Windows, cambiamos el DNS de la tarjeta de red para que apunte al servidor: `Win + R` → `ncpa.cpl` → clic derecho en el adaptador → **Propiedades** → **Protocolo de Internet versión 4 (TCP/IPv4)** → **Propiedades**. Se deja la IP como está y se cambia el DNS a manual, con el servidor **preferido** apuntando a la IP del servidor Linux y el **alternativo vacío**.

> [!WARNING]
> Dejar un DNS alternativo (por ejemplo, uno público como `8.8.8.8`) puede hacer que Windows use ese DNS para resolver el dominio en algunos casos, lo que rompe la unión al dominio de forma intermitente y difícil de diagnosticar. Por eso el campo alternativo debe quedar vacío.

**2.** Limpiamos la caché de DNS de Windows:

```cmd
ipconfig /flushdns
```

**3.** Verificamos que el dominio resuelva correctamente desde Windows (debe responder con la IP del servidor):

```cmd
ping OS3.inet
ping www.OS3.inet
```

**4.** Unimos el equipo al dominio: `Win + R` → `sysdm.cpl` → **Nombre del equipo** → **Cambiar** → **Miembro de un dominio**: `os3.inet`.

**5.** Al pedir credenciales, se ingresa el usuario como:

```text
Usuario: OS3\Administrator
Contraseña: [la contraseña definida al aprovisionar el dominio]
```

> [!IMPORTANT]
> Escribir solo `Administrator` (sin el prefijo `OS3\`) hace que Windows lo interprete como una cuenta **local** de la propia máquina, no como la cuenta de administrador del dominio, y rechaza la contraseña aunque sea correcta.

**6.** Windows pide reiniciar el equipo para aplicar el cambio.

---

## Parte H: Crear un usuario de dominio

**1.** Desde el servidor, creamos un usuario del dominio:

```bash
sudo /usr/local/samba/bin/samba-tool user create lanegracubana <contraseña> --given-name="La Negra" --surname="Cubana" --mail=lanegracubana@os3.inet
```

> [!CAUTION]
> **Dos detalles de este comando en la guía original merecen atención.**
>
> Primero, las comillas de `"20250829"` y de `--given-name=”La Negra”` eran comillas **tipográficas** (`" "`, `” ”`), no las comillas rectas (`"`) que entiende la terminal. Es un problema típico de copiar comandos desde un procesador de texto como Word: la terminal no las reconoce como delimitadoras y el comando falla o se interpreta mal. Hay que escribirlas a mano o pegarlas en un editor de texto plano antes de copiarlas a la terminal.
>
> Segundo, pasar la contraseña como argumento en la línea de comandos (como se muestra en la guía original) la deja visible en el **historial de la shell** (`history`) y en la lista de procesos del sistema mientras el comando se ejecuta (visible, por ejemplo, con `ps aux` desde cualquier otro usuario). Es preferible omitir la contraseña del comando y dejar que `samba-tool` la solicite de forma interactiva y oculta:
>
> ```bash
> sudo /usr/local/samba/bin/samba-tool user create lanegracubana --given-name="La Negra" --surname="Cubana" --mail=lanegracubana@os3.inet
> ```

> [!NOTE]
> El correo del comando original usaba el dominio `so3.inet` (con las letras invertidas), mientras que el dominio real del laboratorio es `os3.inet`. Es probable que sea un error de tipeo; aquí se corrigió a `os3.inet`, el dominio correcto. El valor de `--mail` no afecta el inicio de sesión del usuario, así que el typo no rompe nada, pero sí deja una dirección de correo inconsistente en el directorio.

---

## Parte I: Si el inicio de sesión sigue sin funcionar

Si todo lo anterior está correcto pero el inicio de sesión del dominio sigue fallando, puede deberse al firewall o a SELinux.

**1.** Abrimos el rango estándar de puertos dinámicos que usa RPC:

```bash
sudo firewall-cmd --permanent --add-port=49152-65535/tcp
sudo firewall-cmd --reload
```

**2.** Ponemos SELinux en modo **permisivo** de forma temporal (registra lo que bloquearía, pero no bloquea nada):

```bash
sudo setenforce 0
```

**3.** Verificamos el estado (debe decir `Permissive`):

```bash
sestatus
```

> [!CAUTION]
> `setenforce 0` desactiva la protección de SELinux **hasta el próximo reinicio**, en todo el sistema. Es una herramienta de diagnóstico, no una solución: su propósito real es permitir comprobar si SELinux es la causa del problema (si el error desaparece en modo permisivo, lo era). El paso correcto después de confirmarlo es identificar la regla exacta que bloqueaba el acceso con `ausearch -m avc -ts recent` y crear una excepción puntual (por ejemplo, con `audit2allow`), en vez de dejar SELinux desactivado de forma indefinida.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio, incluida la unión del cliente Windows al dominio.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=fUXbHzmvjdM)

---

## Referencia de comandos

| Comando / archivo | Función |
|---|---|
| `hostnamectl set-hostname <nombre>` | Asigna el nombre de host del sistema |
| `./configure --with-ads --with-systemd` | Configura la compilación de Samba con soporte de Active Directory |
| `samba-tool domain provision` | Aprovisiona un nuevo dominio de Active Directory |
| `samba` / `/usr/local/samba/sbin/samba` | Inicia el servicio de Samba (controlador de dominio) |
| `samba-tool dns add` | Agrega un registro DNS a la zona del dominio |
| `samba-tool user create` | Crea un usuario del dominio |
| `nmcli con mod <interfaz> ipv4.dns <ip>` | Configura el servidor DNS de una conexión de red |
| `firewall-cmd --add-service=... / --add-port=...` | Permite servicios o puertos en el firewall |
| `setenforce 0` / `sestatus` | Cambia y consulta el modo de SELinux (temporalmente) |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| `samba-tool domain provision` falla de inmediato | El nombre de host coincide con el del dominio, o la interfaz indicada no existe | Verificar `hostname -f` y el nombre real de la interfaz con `ip a` |
| El cliente Windows no encuentra el dominio | El DNS del cliente no apunta al servidor, o la caché de DNS está desactualizada | Configurar el DNS preferido del cliente y ejecutar `ipconfig /flushdns` |
| Windows rechaza la contraseña del administrador del dominio | Se escribió `Administrator` sin el prefijo del dominio | Usar `OS3\Administrator`, no solo `Administrator` |
| El cliente del dominio se queda sin Internet | El servidor DNS de Samba no está configurado como resolución del propio servidor, o no reenvía consultas externas | Verificar la configuración de DNS del paso de la Parte F |
| El inicio de sesión del dominio falla sin un motivo claro | Puertos RPC dinámicos bloqueados, o SELinux bloqueando el proceso | Abrir el rango `49152-65535/tcp` y probar temporalmente con `setenforce 0`; si el problema era SELinux, crear una excepción puntual en vez de dejarlo permisivo |
| `samba` no inicia tras reiniciar el servidor | El `PATH` de la nueva sesión no incluye los binarios de Samba | Usar la ruta completa: `sudo /usr/local/samba/sbin/samba` |
| Un comando con comillas falla de forma extraña | Se copiaron comillas tipográficas (`" "`) en vez de comillas rectas (`"`) | Reescribir las comillas a mano en la terminal |

---

[← Volver al laboratorio](../README.md)
