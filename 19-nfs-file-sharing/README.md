[← Volver al laboratorio](../README.md)

# 19 · Compartir archivos entre Linux con NFS

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![NFS](https://img.shields.io/badge/NFS-Network%20File%20System-f472b6?style=flat-square)
![fstab](https://img.shields.io/badge/fstab-Montaje%20permanente-60a5fa?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Configuración de un servidor **NFS** (*Network File System*) en RHEL para compartir un directorio por la red, y de un cliente RHEL que lo monta, primero de forma manual y después de forma permanente mediante `/etc/fstab`.

## Objetivo

Publicar un directorio desde un servidor Linux y acceder a él desde otro equipo Linux como si fuera una carpeta local, comprobando que el montaje sobrevive a un reinicio.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux), servidor y cliente (un clon) |
| Servidor NFS | `192.168.0.16` |
| Directorio compartido | `/var/nfs/os3` |
| Punto de montaje en el cliente | `/mnt/os3` |
| Paquete | `nfs-utils` (en ambos equipos) |
| Servicio | `nfs-server` |

> [!NOTE]
> **Cómo funciona NFS.** El servidor **exporta** un directorio y el cliente lo **monta** en su propio árbol de directorios. Para las aplicaciones del cliente, los archivos parecen locales, pero residen en el servidor. Es la forma estándar de compartir archivos entre equipos Linux y Unix.

## Índice

- [Parte A: Configurar el servidor NFS](#parte-a-configurar-el-servidor-nfs)
- [Parte B: Configurar el cliente](#parte-b-configurar-el-cliente)
- [Parte C: Montaje permanente](#parte-c-montaje-permanente)
- [Parte D: Verificar tras un reinicio](#parte-d-verificar-tras-un-reinicio)
- [Verificación adicional](#verificación-adicional)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Configurar el servidor NFS

**1.** Instalamos las herramientas de NFS en el servidor:

```bash
sudo dnf install nfs-utils -y
```

**2.** Iniciamos y habilitamos el servicio, y verificamos su estado:

```bash
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
sudo systemctl status nfs-server
```

**3.** Creamos el directorio que se va a compartir y 100 archivos de prueba, con expansión de llaves de Bash:

```bash
sudo mkdir -p /var/nfs/os3
cd /var/nfs/os3
sudo touch Adrian{1..100}.txt
```

**4.** Asignamos propietario y permisos para que el directorio sea accesible por otros usuarios:

```bash
sudo chown nobody:nobody /var/nfs/os3
sudo chmod 755 /var/nfs/os3
```

> [!NOTE]
> `nobody` es la cuenta sin privilegios a la que NFS asigna, por defecto, las operaciones del usuario `root` del cliente (opción `root_squash`). Que el directorio le pertenezca evita que un `root` remoto tenga control total sobre los archivos del servidor.

**5.** Definimos qué se comparte y con quién, en `/etc/exports`:

```bash
sudo nano /etc/exports
```

```text
/var/nfs/os3 192.168.0.16/24(rw,sync,no_subtree_check)
```

| Elemento | Significado |
|---|---|
| `/var/nfs/os3` | Directorio exportado |
| `192.168.0.16/24` | Clientes autorizados: los de esa red |
| `rw` | Lectura y escritura |
| `sync` | El servidor confirma una escritura solo cuando ya está en disco (más seguro, algo más lento) |
| `no_subtree_check` | Desactiva la verificación de subárbol, que ralentiza y causa problemas si se renombran archivos |

> [!TIP]
> Para indicar la red completa, lo habitual es usar la **dirección de red** (`192.168.0.0/24`) en vez de la IP del servidor con máscara. Aquí se dejó tal cual se ejecutó, con la IP del servidor, que en la práctica cubre el mismo rango `/24`.

> [!NOTE]
> La opción `rw` permite escribir, pero **no sustituye los permisos del sistema de archivos**. Con el directorio en modo `755` y propiedad de `nobody`, solo ese usuario puede escribir; el resto de usuarios del cliente solo podrá leer. Si se quiere que más usuarios escriban, hay que ajustar propietario, grupo y permisos en el servidor.

**6.** Guardamos, salimos y aplicamos la exportación:

```bash
sudo exportfs -rav
```

`-r` vuelve a exportar todo, `-a` procesa todas las entradas de `/etc/exports` y `-v` muestra el detalle de lo que se exporta.

**7.** Si `firewalld` está activo, permitimos el servicio NFS:

```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --reload
```

---

## Parte B: Configurar el cliente

**1.** En el cliente (un clon de RHEL), actualizamos el sistema e instalamos las herramientas de NFS:

```bash
sudo dnf update
sudo dnf install nfs-utils -y
```

**2.** Creamos el punto de montaje, es decir, el directorio local donde aparecerá el contenido compartido:

```bash
sudo mkdir -p /mnt/os3
```

**3.** Montamos el recurso manualmente, indicando la IP del servidor y la ruta exportada, y verificamos:

```bash
sudo mount -t nfs 192.168.0.16:/var/nfs/os3 /mnt/os3
df -h
```

En la salida de `df -h` debe aparecer una línea con `192.168.0.16:/var/nfs/os3` montada en `/mnt/os3`.

---

## Parte C: Montaje permanente

**1.** El montaje manual se pierde al reiniciar. Para hacerlo permanente, agregamos una línea a `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

```text
192.168.0.16:/var/nfs/os3 /mnt/os3 nfs defaults 0 0
```

| Campo | Valor | Significado |
|---|---|---|
| 1 | `192.168.0.16:/var/nfs/os3` | Recurso remoto (servidor y ruta) |
| 2 | `/mnt/os3` | Punto de montaje local |
| 3 | `nfs` | Tipo de sistema de archivos |
| 4 | `defaults` | Opciones de montaje por defecto |
| 5 | `0` | No se incluye en copias de seguridad con `dump` |
| 6 | `0` | No se verifica con `fsck` al arrancar (no aplica a NFS) |

> [!WARNING]
> Un error en `/etc/fstab` puede impedir el arranque normal del sistema. Antes de reiniciar, conviene probar la línea con `sudo mount -a`: si hay un fallo, aparece en ese momento y no durante el arranque.

> [!TIP]
> Si el servidor NFS no está disponible cuando arranca el cliente, con `defaults` el arranque puede **retrasarse** esperando a que responda. Para evitarlo, se suelen añadir las opciones `_netdev` (indica que el montaje depende de la red) y `nofail` (no falla el arranque si no se puede montar): `192.168.0.16:/var/nfs/os3 /mnt/os3 nfs defaults,_netdev,nofail 0 0`.

**2.** Guardamos, salimos y listamos los archivos:

```bash
ls /mnt/os3
```

Deben aparecer los 100 archivos `Adrian1.txt` a `Adrian100.txt`, que residen físicamente en el servidor.

---

## Parte D: Verificar tras un reinicio

**1.** Reiniciamos el cliente:

```bash
sudo reboot
```

**2.** Tras el reinicio, comprobamos que el recurso se montó solo y volvemos a listar los archivos:

```bash
df -h
ls /mnt/os3
```

Si el recurso aparece montado y los archivos siguen ahí, el montaje permanente funciona.

---

## Verificación adicional

Comandos opcionales para inspeccionar el estado de cada lado.

**En el servidor**, para ver qué se exporta y con qué opciones:

```bash
sudo exportfs -v
```

**En el cliente**, para ver qué ofrece el servidor y cómo quedó el montaje:

```bash
showmount -e 192.168.0.16
mount | grep nfs
```

> [!NOTE]
> Con el modo de seguridad por defecto (`sec=sys`), NFS **confía en la identidad que declara el cliente** (los UID/GID del usuario) y **no cifra el tráfico**. Es adecuado en una red local de laboratorio o de confianza, pero no en redes abiertas. Para entornos que lo exijan, NFS admite autenticación y cifrado con Kerberos (`sec=krb5p`).

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=uB2a7cCQl5A)

---

## Referencia de comandos

| Comando / archivo | Función |
|---|---|
| `dnf install nfs-utils` | Instala las herramientas de servidor y cliente de NFS |
| `systemctl start/enable/status nfs-server` | Gestiona el servicio de NFS en el servidor |
| `/etc/exports` | Define qué directorios se exportan, a quién y con qué opciones |
| `exportfs -rav` | Aplica y muestra las exportaciones definidas |
| `exportfs -v` | Lista las exportaciones activas con sus opciones |
| `firewall-cmd --add-service=nfs` | Permite el tráfico de NFS en el firewall |
| `mount -t nfs <servidor>:<ruta> <punto>` | Monta un recurso NFS manualmente |
| `/etc/fstab` | Define los montajes permanentes del sistema |
| `mount -a` | Monta todo lo definido en `/etc/fstab` (útil para probarlo) |
| `showmount -e <servidor>` | Lista los recursos que exporta un servidor NFS |
| `df -h` | Muestra los sistemas de archivos montados y su uso |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| `mount.nfs: access denied by server` | El cliente no está dentro de la red autorizada en `/etc/exports` | Revisar la entrada de `/etc/exports` y volver a ejecutar `exportfs -rav` |
| `mount.nfs: No route to host` o tiempo de espera agotado | El firewall del servidor bloquea NFS, o no hay conectividad | Verificar `firewall-cmd --list-services`, confirmar que `nfs` esté permitido y probar `ping` |
| `mount: wrong fs type... missing codepage or helper program` | Falta `nfs-utils` en el cliente | Instalar `nfs-utils` en el cliente |
| El cliente puede leer pero no escribir | Los permisos del directorio no permiten escribir a ese usuario, o `root_squash` convierte a `root` en `nobody` | Ajustar propietario y permisos del directorio exportado en el servidor |
| Los archivos no aparecen tras `exportfs` | No se aplicó la exportación, o se escribió mal la ruta | Ejecutar `exportfs -v` y revisar `/etc/exports` |
| El cliente tarda mucho en arrancar o queda detenido | `/etc/fstab` apunta a un servidor NFS no disponible | Añadir `_netdev,nofail` a las opciones, y probar la línea con `mount -a` antes de reiniciar |
| El montaje no sobrevive al reinicio | La línea de `/etc/fstab` está mal escrita, o no se guardó | Revisar la sintaxis (seis campos separados por espacios) y ejecutar `sudo mount -a` |

---

[← Volver al laboratorio](../README.md)
