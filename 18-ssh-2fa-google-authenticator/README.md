[← Volver al laboratorio](../README.md)

# 18 · Autenticación de dos factores (2FA) en SSH con Google Authenticator

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![PAM](https://img.shields.io/badge/PAM-Google%20Authenticator-f472b6?style=flat-square)
![SSH](https://img.shields.io/badge/SSH-2FA-60a5fa?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Configuración de **autenticación de dos factores (2FA)** para el acceso por SSH en RHEL, usando el módulo **PAM de Google Authenticator**. Tras escribir la contraseña, el servidor exige además un código temporal de 6 dígitos generado por una aplicación en el teléfono.

## Objetivo

Reforzar el acceso remoto por SSH de un usuario, de modo que una contraseña robada o adivinada no baste para entrar: también hace falta el segundo factor, el dispositivo donde está registrado el código.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL 8 (Red Hat Enterprise Linux) |
| Servidor SSH | OpenSSH (`sshd`), IP `192.168.0.16` |
| Módulo de 2FA | `pam_google_authenticator`, desde el repositorio EPEL |
| Usuario protegido | `nmontero` |
| Cliente de prueba | Equipo host, con la aplicación Google Authenticator en el teléfono |

> [!NOTE]
> **Cómo funciona el segundo factor.** La contraseña es algo que el usuario **sabe**. El código es algo que el usuario **tiene**: un secreto compartido entre el servidor y el teléfono, que combinado con la hora actual produce un código de 6 dígitos nuevo cada 30 segundos (algoritmo **TOTP**, estandarizado en el RFC 6238). Un atacante que solo conozca la contraseña no puede generar ese código.

> [!WARNING]
> **Riesgo de quedarse sin acceso.** Una mala configuración de PAM o de `sshd` puede impedir el inicio de sesión de todos los usuarios, incluido el administrador. Antes de empezar, conviene tener abierta una sesión de `root` o acceso por la consola de la máquina virtual, y **no cerrarla** hasta confirmar desde una conexión nueva que el acceso con 2FA funciona.

## Índice

- [Parte A: Verificar el servicio SSH](#parte-a-verificar-el-servicio-ssh)
- [Parte B: Instalar Google Authenticator](#parte-b-instalar-google-authenticator)
- [Parte C: Generar el secreto del usuario](#parte-c-generar-el-secreto-del-usuario)
- [Parte D: Configurar el módulo PAM](#parte-d-configurar-el-módulo-pam)
- [Parte E: Configurar sshd](#parte-e-configurar-sshd)
- [Parte F: SELinux y permisos](#parte-f-selinux-y-permisos)
- [Parte G: Aplicar y probar](#parte-g-aplicar-y-probar)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Verificar el servicio SSH

**1.** Instalamos el servidor SSH (si no lo estaba), lo iniciamos, lo habilitamos al arranque y comprobamos su estado:

```bash
sudo dnf install openssh-server
sudo systemctl start sshd
sudo systemctl enable sshd
sudo systemctl status sshd
```

---

## Parte B: Instalar Google Authenticator

**1.** Habilitamos el repositorio **EPEL**, donde se encuentra el paquete:

```bash
sudo dnf install epel-release -y
```

> [!NOTE]
> En RHEL, `epel-release` no siempre está disponible en los repositorios por defecto. Si el comando anterior no encuentra el paquete, se instala desde la URL oficial, como en las prácticas de ClamAV y Snort de este repositorio:
>
> ```bash
> sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y
> ```

**2.** Instalamos Google Authenticator y las herramientas para mostrar el código QR en la terminal:

```bash
sudo dnf install google-authenticator qrencode qrencode-libs
```

---

## Parte C: Generar el secreto del usuario

**1.** Con la sesión iniciada **como el usuario que tendrá 2FA** (en este caso `nmontero`, sin `sudo`), ejecutamos:

```bash
google-authenticator
```

> [!IMPORTANT]
> Este comando se ejecuta como el usuario que se va a proteger, **no** con `sudo` ni como `root`: crea el archivo `~/.google_authenticator` en el directorio personal de quien lo ejecuta. Si se ejecuta como `root`, el secreto queda asociado a `root` y no al usuario que debía protegerse.

**2.** El asistente hace una serie de preguntas. Se respondió **sí (`y`)** a todas, ya que cada una refuerza la seguridad:

| Pregunta | Qué hace al responder `y` |
|---|---|
| ¿Autenticación basada en tiempo? | Usa códigos TOTP que cambian cada 30 segundos |
| ¿Actualizar el archivo `~/.google_authenticator`? | Guarda el secreto y la configuración en el directorio personal |
| ¿No permitir el uso múltiple del mismo código? | Cada código sirve una sola vez, lo que frena ataques de repetición |
| ¿Ampliar la ventana de tolerancia de tiempo? | Acepta códigos de ±1 intervalo, útil si los relojes difieren un poco |
| ¿Limitar los intentos de inicio de sesión? | Máximo 3 intentos cada 30 segundos, lo que frena la fuerza bruta |

**3.** El asistente muestra un **código QR**, la **clave secreta** y varios **códigos de emergencia**. Escaneamos el QR con la aplicación **Google Authenticator** del teléfono (también sirven Microsoft Authenticator, Authy o FreeOTP), y guardamos los códigos de emergencia en un lugar seguro.

> [!WARNING]
> El código QR, la clave secreta y los códigos de emergencia equivalen a una **contraseña**: quien los tenga puede generar códigos válidos. No deben publicarse en capturas de pantalla ni en repositorios públicos. Si se llegaran a exponer, se regenera el secreto ejecutando `google-authenticator` de nuevo y registrando la nueva clave en el teléfono.

---

## Parte D: Configurar el módulo PAM

**1.** Editamos la configuración de PAM para el servicio SSH:

```bash
sudo nano /etc/pam.d/sshd
```

**2.** El archivo debe quedar así:

```text
#%PAM-1.0
auth        substack      password-auth
auth required pam_google_authenticator.so
auth        include       postlogin
account     required      pam_sepermit.so
account     required      pam_nologin.so
account     include       password-auth
password    include       password-auth
# pam_selinux.so close should be the first session rule
session     required      pam_selinux.so close
session     required      pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session     required      pam_selinux.so open env_params
session     required      pam_namespace.so
session     optional      pam_keyinit.so force revoke
session     optional      pam_motd.so
session     include       password-auth
session     include       postlogin
```

La **única línea agregada** es `auth required pam_google_authenticator.so`, justo después de la verificación de contraseña (`auth substack password-auth`). PAM evalúa las reglas de arriba hacia abajo: primero la contraseña y después el código.

> [!CAUTION]
> Con `required` y sin la opción `nullok`, **todo usuario que no haya generado su `~/.google_authenticator` deja de poder entrar por SSH**, porque el módulo no encuentra su secreto. Antes de aplicar este cambio en un servidor con más usuarios, hay que configurar el 2FA de cada uno, o agregar `nullok` al final de la línea (`auth required pam_google_authenticator.so nullok`) para que quienes aún no tienen secreto puedan seguir entrando, aunque sin segundo factor.

---

## Parte E: Configurar sshd

**1.** Editamos la configuración del servidor SSH:

```bash
sudo nano /etc/ssh/sshd_config
```

**2.** Aseguramos que estas tres directivas estén activas:

```text
PasswordAuthentication yes
ChallengeResponseAuthentication yes
UsePAM yes
```

| Directiva | Para qué sirve |
|---|---|
| `UsePAM yes` | Hace que `sshd` delegue la autenticación en PAM (donde se agregó el módulo de 2FA) |
| `ChallengeResponseAuthentication yes` | Permite el método interactivo por el que se pide el código de verificación |
| `PasswordAuthentication yes` | Mantiene la contraseña como primer factor |

> [!TIP]
> En OpenSSH 8.0 (el de RHEL 8) la directiva se llama `ChallengeResponseAuthentication`; en versiones más nuevas, como la de RHEL 9, el nombre vigente es `KbdInteractiveAuthentication`. Para hacer explícito que el inicio de sesión debe usar siempre el método interactivo (y por tanto el 2FA), se puede añadir además `AuthenticationMethods keyboard-interactive`.

---

## Parte F: SELinux y permisos

**1.** SELinux puede impedir que `sshd` lea el archivo del secreto en el directorio personal del usuario. Ajustamos el contexto de seguridad para que los servicios de autenticación puedan acceder, y lo aplicamos:

```bash
sudo semanage fcontext -a -t auth_home_t "/home/nmontero(/.*)?"
sudo restorecon -Rv /home/nmontero
```

> [!CAUTION]
> Esta regla aplica el tipo `auth_home_t` a **todo** el directorio personal y a su contenido, no solo al archivo de 2FA. Funciona, pero es más amplia de lo necesario y puede afectar a otros servicios o aplicaciones que esperen el tipo habitual. Una alternativa más acotada es limitar la regla al archivo del secreto: `sudo semanage fcontext -a -t auth_home_t "/home/nmontero/\.google_authenticator.*"` seguido de `sudo restorecon -v /home/nmontero/.google_authenticator`.

**2.** Confirmamos el propietario y los permisos del archivo del secreto, y del directorio personal:

```bash
sudo chown nmontero:nmontero /home/nmontero/.google_authenticator
sudo chmod 600 /home/nmontero/.google_authenticator

sudo chmod 700 /home/nmontero
sudo chown nmontero:nmontero /home/nmontero
```

> [!NOTE]
> El módulo se niega a usar el archivo si sus permisos son demasiado abiertos: debe pertenecer al usuario y ser legible solo por él (`600`).

---

## Parte G: Aplicar y probar

**1.** Reiniciamos el servicio SSH para aplicar los cambios:

```bash
sudo systemctl restart sshd
```

**2.** Desde **otra** terminal o desde el equipo host (sin cerrar la sesión administrativa abierta), probamos la conexión:

```bash
ssh nmontero@192.168.0.16
```

**3.** El servidor pide primero la **contraseña** y después el **código de verificación** de la aplicación (ejemplo ilustrativo del flujo):

```text
(nmontero@192.168.0.16) Password:
(nmontero@192.168.0.16) Verification code:
```

Con ambos correctos, se inicia la sesión. Con la contraseña correcta pero un código incorrecto o vencido, el acceso se rechaza.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=ooAtZRofuto&list=PLIJEt4hLYnOXYtfF8MnxM5sMboz-KI20D)

---

## Referencia de comandos

| Comando / archivo | Función |
|---|---|
| `google-authenticator` | Genera el secreto TOTP, el QR y los códigos de emergencia del usuario actual |
| `~/.google_authenticator` | Archivo con el secreto y la configuración del 2FA de cada usuario |
| `/etc/pam.d/sshd` | Pila de autenticación PAM del servicio SSH |
| `pam_google_authenticator.so` | Módulo PAM que valida el código de verificación |
| `/etc/ssh/sshd_config` | Configuración del servidor SSH |
| `semanage fcontext -a -t auth_home_t "<ruta>"` | Define un contexto SELinux para una ruta |
| `restorecon -Rv <ruta>` | Aplica los contextos SELinux definidos a una ruta existente |
| `systemctl restart sshd` | Aplica los cambios de configuración de SSH |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| El código se rechaza aunque es el que muestra la aplicación | El reloj del servidor o del teléfono está desfasado | Sincronizar la hora (`timedatectl`, `chronyc tracking`) y verificar que el teléfono use hora automática |
| Se pide la contraseña pero nunca el código, y el acceso falla | SELinux bloquea la lectura de `~/.google_authenticator` | Revisar `ls -Z ~/.google_authenticator` y los avisos con `sudo ausearch -m avc -ts recent`; reaplicar el contexto con `restorecon` |
| Ningún usuario puede entrar por SSH tras el cambio | La línea `required` está activa y hay usuarios sin secreto | Entrar por la consola de la VM y agregar `nullok` a la línea de PAM, o generar el secreto de cada usuario |
| `semanage: command not found` | Falta el paquete que provee `semanage` | Instalar `policycoreutils-python-utils` |
| El módulo ignora el archivo del secreto | Propietario o permisos incorrectos | `chown usuario:usuario` y `chmod 600` sobre `~/.google_authenticator` |
| Los cambios no surten efecto | No se reinició el servicio | Ejecutar `sudo systemctl restart sshd` |
| Se perdió el teléfono | El segundo factor ya no está disponible | Usar un código de emergencia; si no hay, un administrador elimina `~/.google_authenticator` (desde la consola) y el usuario genera uno nuevo |

---

[← Volver al laboratorio](../README.md)
