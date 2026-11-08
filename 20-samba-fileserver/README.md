[← Volver al laboratorio](../README.md)

# 20 · Servidor de archivos compatible con Windows usando Samba

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![Samba](https://img.shields.io/badge/Samba-SMB%2FCIFS-f472b6?style=flat-square)
![Windows](https://img.shields.io/badge/Cliente-Windows-60a5fa?style=flat-square&logo=windows&logoColor=white)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Configuración de un servidor de archivos en RHEL con **Samba**, que implementa el protocolo **SMB/CIFS** de Windows, de modo que un equipo Windows pueda acceder a una carpeta del servidor Linux, autenticarse con un usuario propio y editar archivos como si estuvieran en una unidad de red.

## Objetivo

Compartir un directorio de un servidor Linux con un cliente Windows, con autenticación por usuario, y comprobar que los cambios hechos desde Windows se reflejan en el servidor.

## Entorno

| Elemento | Valor |
|---|---|
| Servidor | RHEL (Red Hat Enterprise Linux), IP `192.168.0.21` |
| Servicios | `smb` (compartición de archivos) y `nmb` (resolución de nombres NetBIOS) |
| Directorio compartido | `/samba/compartir` |
| Nombre del recurso | `compartir` |
| Usuario de Samba | `zamba` |
| Cliente | Windows (Explorador de archivos, unidad de red `Z:`) |

> [!NOTE]
> **Samba y los usuarios.** Samba mantiene su **propia base de contraseñas**, separada de la del sistema Linux. Por eso un usuario debe existir en Linux y, además, registrarse en Samba con `smbpasswd -a`. Esa contraseña es la que se escribe desde Windows.

## Índice

- [Parte A: Instalar y habilitar Samba](#parte-a-instalar-y-habilitar-samba)
- [Parte B: Crear el directorio compartido](#parte-b-crear-el-directorio-compartido)
- [Parte C: Usuario de Samba y SELinux](#parte-c-usuario-de-samba-y-selinux)
- [Parte D: Definir el recurso compartido](#parte-d-definir-el-recurso-compartido)
- [Parte E: Acceder desde Windows](#parte-e-acceder-desde-windows)
- [Parte F: Verificar en Linux](#parte-f-verificar-en-linux)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Instalar y habilitar Samba

**1.** Instalamos Samba:

```bash
sudo dnf install -y samba*
```

> [!TIP]
> El comodín `samba*` instala todos los paquetes cuyo nombre empieza por `samba` (cliente, herramientas, bibliotecas, utilidades de dominio...), muchos innecesarios para un servidor de archivos. Para este caso bastan `samba` y, si se quiere probar desde Linux, `samba-client`: `sudo dnf install -y samba samba-client`.

**2.** Habilitamos e iniciamos los servicios `smb` y `nmb`:

```bash
sudo systemctl enable --now smb nmb
sudo systemctl start nmb smb
```

> [!NOTE]
> `enable --now` ya habilita **e inicia** los servicios, así que el segundo comando (`start`) es redundante: no causa ningún problema, simplemente no hace nada nuevo. En cuanto a los servicios, `smb` es el que comparte archivos; `nmb` ofrece resolución de nombres NetBIOS, y los clientes Windows modernos pueden conectarse por la dirección IP sin necesitarlo.

**3.** Permitimos Samba en el firewall:

```bash
sudo firewall-cmd --add-service=samba --permanent
sudo firewall-cmd --reload
```

> [!CAUTION]
> En la lista de comandos original, el último aparecía como `firewall-cmd –reload`, con un **guion largo** (`–`) en vez de dos guiones normales (`--`). Es un error habitual al copiar desde editores de texto: el comando falla y la regla permanente no se aplica hasta recargar. Debe escribirse con dos guiones normales, como arriba.

---

## Parte B: Crear el directorio compartido

**1.** Creamos el directorio y 100 archivos de prueba, con expansión de llaves de Bash:

```bash
sudo mkdir -p /samba/compartir
cd /samba/compartir
sudo touch Adrian{1..100}.txt
```

> [!NOTE]
> Como `/samba/compartir` se crea con `sudo`, pertenece a `root`. Para crear archivos dentro con `touch` desde un usuario normal hace falta `sudo`; sin él, el comando devuelve `Permission denied`.

**2.** Ajustamos permisos y propietario:

```bash
sudo chmod -R 777 /samba/compartir
sudo chown -R nobody:nobody /samba/compartir
```

> [!WARNING]
> `chmod 777` da permiso de lectura, escritura y ejecución a **cualquier usuario del sistema**, y no solo al usuario de Samba. Es cómodo para una demostración, pero en un servidor real conviene un permiso más estricto, por ejemplo `chmod -R 775` con un grupo dedicado (`chgrp`) al que pertenezca el usuario de Samba.

---

## Parte C: Usuario de Samba y SELinux

**1.** Permitimos que Samba lea y escriba en directorios fuera de los habituales, mediante un booleano de SELinux (`-P` lo hace permanente):

```bash
sudo setsebool -P samba_export_all_rw on
```

> [!TIP]
> `samba_export_all_rw` es un interruptor **amplio**: permite que Samba lea y escriba en cualquier archivo del sistema que sus permisos autoricen. Una alternativa más acotada es etiquetar solo la carpeta compartida con el tipo de SELinux previsto para Samba:
>
> ```bash
> sudo semanage fcontext -a -t samba_share_t "/samba/compartir(/.*)?"
> sudo restorecon -Rv /samba/compartir
> ```
>
> Con esa etiqueta, el booleano amplio deja de ser necesario. Si se usa esta variante, conviene probarla en el propio laboratorio antes de darla por buena.

**2.** Creamos el usuario en Linux y lo registramos en Samba, con su contraseña (se pide dos veces):

```bash
sudo useradd zamba
sudo smbpasswd -a zamba
```

> [!TIP]
> Si `zamba` solo va a usarse para acceder al recurso compartido, es mejor que **no pueda iniciar sesión** en el servidor, creándolo sin directorio personal ni *shell*: `sudo useradd -M -s /sbin/nologin zamba`. Así, una contraseña de Samba comprometida no da acceso a una terminal.

---

## Parte D: Definir el recurso compartido

**1.** Abrimos la configuración de Samba:

```bash
sudo nano /etc/samba/smb.conf
```

**2.** Al final del archivo, agregamos la definición del recurso:

```ini
[compartir]
   path = /samba/compartir
   browseable = yes
   writeable = yes
   valid users = zamba
   create mask = 0777
   directory mask = 0777
   guest ok = yes
```

| Parámetro | Significado |
|---|---|
| `[compartir]` | Nombre con el que el recurso aparece en la red |
| `path` | Directorio del servidor que se comparte |
| `browseable = yes` | El recurso aparece en la lista al explorar el servidor |
| `writeable = yes` | Permite escribir (no solo leer) |
| `valid users = zamba` | Solo ese usuario puede acceder |
| `create mask` / `directory mask` | Permisos con los que se crean los archivos y carpetas nuevos |
| `guest ok = yes` | Permite acceso de invitados, sin contraseña |

> [!WARNING]
> **Dos de estos parámetros se contradicen en intención.** `valid users = zamba` restringe el acceso a un usuario concreto, mientras que `guest ok = yes` declara que se admiten invitados sin contraseña. Con ambos, el resultado depende de cómo se conecte el cliente y es difícil de razonar. Si el objetivo es **exigir usuario y contraseña**, como en esta práctica, conviene quitar `guest ok = yes`. Además, `create mask = 0777` y `directory mask = 0777` hacen que todo lo que se cree desde Windows quede legible y escribible por cualquiera en el servidor; valores como `0664` (archivos) y `0775` (carpetas) son más prudentes.

**3.** Guardamos, y reiniciamos los servicios para aplicar la configuración:

```bash
sudo systemctl restart smb nmb
```

> [!TIP]
> Antes de reiniciar, `testparm` valida la sintaxis de `smb.conf` y muestra la configuración efectiva. Detecta errores de escritura que, de otro modo, impedirían arrancar el servicio.

---

## Parte E: Acceder desde Windows

**1.** Abrimos el **Explorador de archivos** y, en la barra de direcciones, escribimos la dirección del servidor:

```text
\\192.168.0.21
```

**2.** Cuando se solicite, ingresamos el usuario `zamba` y la contraseña definida con `smbpasswd`.

**3.** Mapeamos el recurso como unidad de red:

- Clic derecho sobre la carpeta `compartir`.
- **Conectar a unidad de red**.
- Elegimos una letra, por ejemplo `Z:`.
- Marcamos **Volver a conectar al iniciar sesión**.
- Clic en **Finalizar**.

**4.** Abrimos el archivo `Adrian99.txt` desde la unidad `Z:`, escribimos algo de texto y guardamos.

---

## Parte F: Verificar en Linux

**1.** En el servidor, comprobamos que el cambio hecho desde Windows llegó al archivo:

```bash
cat /samba/compartir/Adrian99.txt
```

Debe mostrarse el texto escrito desde Windows, lo que confirma que el recurso compartido funciona en ambos sentidos.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=jjoOv4EGQgc)

---

## Referencia de comandos

| Comando / archivo | Función |
|---|---|
| `dnf install samba` | Instala el servidor Samba |
| `systemctl enable --now smb nmb` | Habilita e inicia los servicios de Samba |
| `firewall-cmd --add-service=samba --permanent` | Permite el tráfico de Samba en el firewall |
| `setsebool -P samba_export_all_rw on` | Permite a Samba leer y escribir en directorios fuera de los predeterminados |
| `smbpasswd -a <usuario>` | Registra un usuario de Linux en la base de contraseñas de Samba |
| `/etc/samba/smb.conf` | Configuración de Samba y definición de recursos compartidos |
| `testparm` | Valida la sintaxis de `smb.conf` |
| `smbclient -L //<servidor> -U <usuario>` | Lista los recursos que ofrece un servidor (desde Linux, requiere `samba-client`) |
| `\\<IP>` (Explorador de Windows) | Accede a un servidor SMB desde Windows |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| Windows no encuentra el servidor (`\\192.168.0.21`) | El firewall bloquea Samba, o los servicios no están activos | Verificar `firewall-cmd --list-services` y `systemctl status smb` |
| Se pide la contraseña y se rechaza | El usuario no está registrado en Samba, o la contraseña es la del sistema, no la de Samba | Ejecutar `sudo smbpasswd -a <usuario>` y usar esa contraseña |
| Se accede al recurso, pero no se puede guardar | SELinux bloquea la escritura, o los permisos del directorio no lo permiten | Revisar `ausearch -m avc -ts recent`, el booleano de SELinux y los permisos |
| `firewall-cmd --reload` devuelve un error de opción desconocida | Se escribió con un guion largo (`–`) en vez de dos guiones (`--`) | Reescribirlo a mano con dos guiones normales |
| Samba no arranca tras editar `smb.conf` | Error de sintaxis en el archivo | Ejecutar `testparm` y corregir la línea indicada |
| Los cambios de `smb.conf` no se reflejan | No se reiniciaron los servicios | Ejecutar `sudo systemctl restart smb nmb` |
| `touch` devuelve `Permission denied` al crear los archivos | El directorio pertenece a `root` | Usar `sudo touch ...` |

---

[← Volver al laboratorio](../README.md)
