[← Volver al laboratorio](../README.md)

# 16 · Control de tráfico con iptables y firewalld

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![iptables](https://img.shields.io/badge/iptables-Filtrado%20de%20paquetes-f472b6?style=flat-square)
![firewalld](https://img.shields.io/badge/firewalld-Zonas%20y%20servicios-60a5fa?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Permitir y bloquear tráfico hacia tres servicios (HTTP, FTP y SSH) en RHEL con dos herramientas de firewall distintas: **iptables** (reglas manuales, de bajo nivel) y **firewalld** (gestión por zonas y servicios).

## Objetivo

Instalar tres servicios de red y controlar el acceso a sus puertos, primero con reglas de `iptables` y después con `firewalld`, comprendiendo las diferencias de enfoque entre ambas herramientas.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Servicios | `httpd` (puerto 80), `vsftpd` (puerto 21) y `sshd` (puerto 22) |
| Herramientas | `iptables` y `firewalld` |

## Índice

- [Parte A: Preparar los servicios](#parte-a-preparar-los-servicios)
- [Parte B: Control de tráfico con iptables](#parte-b-control-de-tráfico-con-iptables)
- [Parte C: Control de tráfico con firewalld](#parte-c-control-de-tráfico-con-firewalld)
- [Verificación](#verificación)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Preparar los servicios

**1.** Detenemos `firewalld` para poder trabajar con `iptables` directamente, sin que ambas herramientas gestionen reglas al mismo tiempo:

```bash
sudo systemctl stop firewalld
```

**2.** Instalamos los tres servicios que se van a controlar:

```bash
sudo dnf install httpd vsftpd openssh-server -y
```

**3.** Los iniciamos y habilitamos para que arranquen con el sistema:

```bash
sudo systemctl start httpd vsftpd sshd
sudo systemctl enable httpd vsftpd sshd
```

**4.** Verificamos que los tres estén activos:

```bash
systemctl status httpd
systemctl status vsftpd
systemctl status sshd
```

> [!TIP]
> FTP transmite el usuario y la contraseña **sin cifrar**. Aquí se usa únicamente como servicio de prueba para practicar el filtrado de puertos; para transferir archivos de forma real, la alternativa segura es SFTP, que viaja sobre SSH.

---

## Parte B: Control de tráfico con iptables

**1. Permitir tráfico.** Agregamos una regla `ACCEPT` por cada puerto (`-A` agrega la regla al final de la cadena `INPUT`):

```bash
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

> [!NOTE]
> Con `firewalld` detenido y sin reglas previas, la política por defecto de `iptables` es aceptar todo el tráfico entrante. Estas reglas `ACCEPT` dejan **documentado** explícitamente qué se permite, pero por sí solas no cambian el comportamiento: el efecto visible aparece en el siguiente paso, al bloquear.

**2. Bloquear tráfico.** Primero eliminamos las reglas `ACCEPT` (`-D`) y luego agregamos reglas `REJECT`:

```bash
sudo iptables -D INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport 22 -j ACCEPT

sudo iptables -A INPUT -p tcp --dport 80 -j REJECT
sudo iptables -A INPUT -p tcp --dport 21 -j REJECT
sudo iptables -A INPUT -p tcp --dport 22 -j REJECT
```

> [!IMPORTANT]
> **El orden de las reglas importa.** `iptables` evalúa las reglas de arriba hacia abajo y aplica la primera que coincide. Por eso se elimina primero el `ACCEPT`: si se agregara el `REJECT` al final con el `ACCEPT` todavía presente, el tráfico seguiría permitido, porque el `ACCEPT` coincide antes.

> [!WARNING]
> **Cuidado con bloquear el puerto 22 (SSH).** Si se ejecuta esta regla estando conectado al servidor **por SSH**, la propia sesión queda bloqueada y se pierde el acceso remoto. En este laboratorio conviene trabajar desde la consola de la máquina virtual (no desde una sesión SSH) antes de bloquear ese puerto.

**3. Volver a permitir tráfico.** Eliminamos las reglas `REJECT` y restauramos las `ACCEPT`:

```bash
sudo iptables -D INPUT -p tcp --dport 80 -j REJECT
sudo iptables -D INPUT -p tcp --dport 21 -j REJECT
sudo iptables -D INPUT -p tcp --dport 22 -j REJECT

sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

> [!NOTE]
> Las reglas de `iptables` creadas con estos comandos viven **solo en memoria**: se pierden al reiniciar el servidor. Para conservarlas hay que guardarlas (por ejemplo, con `iptables-save` o el paquete `iptables-services`). `firewalld`, en cambio, las guarda de forma permanente con la opción `--permanent`.

---

## Parte C: Control de tráfico con firewalld

**1.** Volvemos a iniciar `firewalld`:

```bash
sudo systemctl start firewalld
```

**2. Bloquear tráfico.** Quitamos el puerto 80 y los servicios FTP y SSH de la zona activa, y recargamos para aplicar los cambios:

```bash
sudo firewall-cmd --permanent --remove-port=80/tcp
sudo firewall-cmd --permanent --remove-service=ftp
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload
```

**3. Permitir tráfico nuevamente:**

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-service=ftp
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

> [!CAUTION]
> En la guía original, el último comando aparecía escrito como `firewall-cmd –reload`, con un **guion largo** (`–`) en vez de dos guiones normales (`--`). Ese carácter es un error común al copiar texto desde editores de documentos, y hace que el comando **falle**: la opción `--reload` no se reconoce y los cambios permanentes no se aplican hasta recargar correctamente.

> [!NOTE]
> **`--add-port` frente a `--add-service`.** `--add-port=80/tcp` abre un puerto concreto; `--add-service=ftp` abre los puertos asociados a un servicio predefinido por nombre (definidos en `/usr/lib/firewalld/services/`). Los servicios son más legibles y se mantienen solos si cambian los puertos que usa el protocolo.

---

## Verificación

Comandos opcionales para comprobar el estado de cada herramienta.

**Reglas de iptables** (con números de línea, útil para ver el orden):

```bash
sudo iptables -L INPUT -n --line-numbers
```

**Estado de firewalld:**

```bash
sudo firewall-cmd --list-all
```

**Prueba de conectividad** desde otro equipo (por ejemplo, el host Windows):

```powershell
Test-NetConnection <IP-del-servidor> -Port 80
```

> [!TIP]
> Con `REJECT`, el cliente recibe de inmediato una respuesta de "conexión rechazada", porque el servidor le contesta con un mensaje ICMP de puerto inalcanzable. Con `DROP`, en cambio, el servidor descarta el paquete en silencio y el cliente espera hasta agotar el tiempo de espera. `DROP` revela menos información a quien escanea el servidor; `REJECT` hace los fallos más fáciles de diagnosticar.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=YKF_xFRcbp4&list=PLIJEt4hLYnOXYtfF8MnxM5sMboz-KI20D&index=3)

---

## Referencia de comandos

| Comando | Función |
|---|---|
| `iptables -A INPUT -p tcp --dport <puerto> -j ACCEPT` | Agrega una regla que acepta el tráfico TCP entrante a un puerto |
| `iptables -A INPUT -p tcp --dport <puerto> -j REJECT` | Agrega una regla que rechaza el tráfico y avisa al cliente |
| `iptables -D INPUT ...` | Elimina una regla existente (se repite la misma regla con `-D` en vez de `-A`) |
| `iptables -L INPUT -n --line-numbers` | Lista las reglas de la cadena `INPUT` con su número de orden |
| `firewall-cmd --permanent --add-port=<puerto>/tcp` | Abre un puerto de forma permanente |
| `firewall-cmd --permanent --add-service=<servicio>` | Permite un servicio predefinido de forma permanente |
| `firewall-cmd --permanent --remove-port=...` / `--remove-service=...` | Quita un puerto o servicio de forma permanente |
| `firewall-cmd --reload` | Aplica los cambios permanentes a la configuración activa |
| `firewall-cmd --list-all` | Muestra la configuración activa de la zona |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| El bloqueo con `REJECT` no surte efecto | La regla `ACCEPT` anterior sigue presente y coincide primero | Eliminarla con `-D` antes de agregar el `REJECT`, y revisar el orden con `--line-numbers` |
| Se perdió la sesión SSH al aplicar las reglas | Se bloqueó el puerto 22 estando conectado por SSH | Entrar por la consola de la VM y eliminar la regla `REJECT` del puerto 22 |
| Las reglas de `iptables` desaparecieron tras reiniciar | Se guardan solo en memoria | Guardarlas con `iptables-save` o usar `iptables-services` |
| `firewall-cmd --reload` devuelve un error de opción desconocida | Se escribió con un guion largo (`–`) en vez de dos guiones (`--`) | Reescribir el comando a mano con dos guiones normales |
| `firewall-cmd --remove-service=...` muestra `NOT_ENABLED` | Ese servicio no estaba habilitado en la zona | Es una advertencia, no un error: confirmar con `firewall-cmd --list-all` qué hay realmente habilitado |
| El servicio está permitido, pero no responde | El servicio no está en ejecución | Verificar con `systemctl status <servicio>` |

---

[← Volver al laboratorio](../README.md)
