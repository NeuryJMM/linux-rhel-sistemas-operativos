[← Volver al laboratorio](../README.md)

# 14 · Clúster de alta disponibilidad HTTP con keepalived

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-httpd-f472b6?style=flat-square)
![Keepalived](https://img.shields.io/badge/Keepalived-VRRP-60a5fa?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Configuración de **alta disponibilidad** para un servicio web Apache (`httpd`) entre dos servidores RHEL, usando **keepalived** y el protocolo **VRRP** (*Virtual Router Redundancy Protocol*) para gestionar una IP virtual que conmuta automáticamente entre los dos nodos.

## Objetivo

Que dos servidores Apache independientes compartan una misma IP virtual, de modo que si el servidor activo falla, el otro tome el control de esa IP de forma automática y continúe sirviendo contenido web sin interrupción.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Nodos | Servidor 1 (principal) y Servidor 2 (clon) |
| Servicios | `httpd` (Apache) y `keepalived` |
| Protocolo de redundancia | VRRP, a través de `keepalived` |
| IP virtual (flotante) | `192.168.0.200/24` |
| Interfaz de red | `ens160` |

## Índice

- [Parte A: Instalar Apache y keepalived](#parte-a-instalar-apache-y-keepalived)
- [Parte B: Páginas de identificación por servidor](#parte-b-páginas-de-identificación-por-servidor)
- [Parte C: Firewall y Apache](#parte-c-firewall-y-apache)
- [Parte D: Configurar keepalived (VRRP)](#parte-d-configurar-keepalived-vrrp)
- [Parte E: Verificar la alta disponibilidad](#parte-e-verificar-la-alta-disponibilidad)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Instalar Apache y keepalived

> [!NOTE]
> Todos los pasos de esta práctica, salvo donde se indique lo contrario, se repiten **en ambos servidores**.

**1.** Instalamos el servidor web Apache:

```bash
sudo dnf install httpd
```

**2.** Instalamos `keepalived`, el servicio que implementa VRRP y gestiona la IP virtual:

```bash
sudo dnf install keepalived
```

**3.** Habilitamos e iniciamos ambos servicios:

```bash
sudo systemctl enable httpd && sudo systemctl start httpd
sudo systemctl enable keepalived && sudo systemctl start keepalived
```

---

## Parte B: Páginas de identificación por servidor

**1.** En **cada** servidor, creamos una página de inicio distinta, para poder identificar visualmente cuál de los dos está respondiendo en un momento dado:

```bash
sudo nano /var/www/html/index.html
```

**Servidor 1:**

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body>
  <h1>¡Este es el servidor 1!</h1>
</body>
</html>
```

**Servidor 2:**

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body>
  <h1>¡Este es el servidor 2!</h1>
</body>
</html>
```

> [!TIP]
> Esta página distinta por servidor no es necesaria en un clúster real (donde ambos nodos suelen servir el mismo contenido, sincronizado entre ellos). Aquí cumple un propósito puramente didáctico: permite confirmar a simple vista, desde el navegador, cuál servidor está respondiendo en cada momento durante la prueba de conmutación.

---

## Parte C: Firewall y Apache

**1.** En ambos servidores, permitimos el puerto 80 (HTTP) y el protocolo **VRRP** (necesario para que los dos nodos se comuniquen entre sí y negocien cuál tiene la IP virtual):

```bash
sudo firewall-cmd --permanent --add-port=80/tcp --zone=public && sudo firewall-cmd --permanent --add-protocol=vrrp --zone=public
sudo firewall-cmd --reload
```

**2.** Configuramos Apache para escuchar en el puerto 80 en todas las interfaces:

```bash
sudo nano /etc/httpd/conf/httpd.conf
```

```apache
Listen *:80
```

---

## Parte D: Configurar keepalived (VRRP)

**1.** Editamos la configuración de `keepalived` en cada servidor:

```bash
sudo nano /etc/keepalived/keepalived.conf
```

**Servidor 1** (prioridad más alta: `150`):

```conf
global_defs {
	router_id SERVER1
}
vrrp_script check_service {
        script "/usr/bin/killall -0 httpd" # Cambia según el servicio
        interval 2
        weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.0.200/24
    }

    track_script {
        check_service
    }
}
```

**Servidor 2** (prioridad más baja: `20`):

```conf
global_defs {
	router_id SERVER2
}
vrrp_script check_service {
        script "/usr/bin/killall -0 httpd" # Cambia según el servicio
        interval 2
        weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 20
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.0.200/24
    }

    track_script {
        check_service
    }
}
```

> [!NOTE]
> **Los dos nodos están en `state BACKUP`, y eso es intencional**, no un error de copiar y pegar. En `keepalived`, declarar ambos nodos como `BACKUP` (en vez de marcar uno como `MASTER` a mano) y dejar que la **prioridad** (`priority`) decida quién toma el rol activo es la práctica recomendada: evita que, tras un reinicio, el nodo marcado como `MASTER` recupere el control automáticamente sin pasar por una elección real, lo que podría causar una interrupción innecesaria (dos nodos creyéndose maestros a la vez, aunque sea brevemente). Con esta configuración, el servidor 1 (prioridad 150) gana la elección y se convierte en maestro mientras esté disponible; si falla, el servidor 2 (prioridad 20) toma el control.

> [!CAUTION]
> `auth_type PASS` envía la contraseña de autenticación VRRP (`auth_pass`) **en texto plano** por la red, y únicamente sirve para evitar que un nodo mal configurado se una por accidente al clúster, no como una medida de seguridad real contra un atacante. Además, `1234` es un valor de ejemplo, no apto para producción. En un entorno real conviene usar un valor menos predecible y, sobre todo, no depender de esta autenticación como control de acceso: el protocolo VRRP en general asume una red local confiable.

**2.** `vrrp_script check_service` verifica, cada 2 segundos, que el proceso `httpd` siga activo (`killall -0` no mata el proceso, solo comprueba que exista). Si `httpd` deja de responder en el nodo maestro, su prioridad efectiva baja (`weight 2`) y el otro nodo toma el control de la IP virtual, aunque el servidor en sí siga encendido.

**3.** Reiniciamos ambos servicios en cada servidor para aplicar la configuración:

```bash
sudo systemctl restart httpd
sudo systemctl restart keepalived.service
```

---

## Parte E: Verificar la alta disponibilidad

**1.** Desde el equipo host, abrimos la IP virtual en el navegador:

```text
http://192.168.0.200
```

Debe mostrarse la página del servidor activo en ese momento (por prioridad, inicialmente el Servidor 1).

**2.** Iniciamos un `ping` continuo hacia la IP virtual:

```bash
ping 192.168.0.200
```

**3.** Mientras el `ping` sigue corriendo, reiniciamos el servidor maestro (o deshabilitamos su interfaz de red).

**4.** Si la configuración es correcta, el `ping` **no se pierde**, y al recargar el navegador, ahora se muestra la página del Servidor 2 (*"¡Este es el servidor 2!"*): la IP virtual y el servicio conmutaron automáticamente al nodo restante.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio, incluida la prueba de conmutación.

**Video de la práctica:** [Ver demostración en YouTube](https://youtu.be/nfFdH22lkGo?si=m2CmajhCpSDIFEJG)

---

## Referencia de comandos

| Comando / directiva | Función |
|---|---|
| `dnf install httpd` / `dnf install keepalived` | Instala Apache y keepalived |
| `firewall-cmd --add-protocol=vrrp` | Permite el tráfico del protocolo VRRP entre nodos |
| `Listen *:80` (en `httpd.conf`) | Hace que Apache escuche en el puerto 80 de todas las interfaces |
| `vrrp_instance` | Bloque de configuración de una instancia VRRP en `keepalived.conf` |
| `priority` | Determina qué nodo gana la elección como maestro (mayor valor, mayor prioridad) |
| `virtual_ipaddress` | La IP virtual gestionada por esa instancia VRRP |
| `vrrp_script` + `track_script` | Verifica un servicio externo (aquí, `httpd`) y ajusta la prioridad si falla |
| `systemctl restart keepalived` | Aplica los cambios de configuración de keepalived |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| La IP virtual nunca aparece en ningún nodo | El protocolo VRRP está bloqueado en el firewall | Verificar `firewall-cmd --list-all` en ambos servidores y confirmar que `vrrp` esté permitido |
| Ambos servidores muestran la IP virtual al mismo tiempo (*split-brain*) | `virtual_router_id` no coincide entre los dos nodos, o hay un problema de comunicación de red | Confirmar que `virtual_router_id` sea idéntico en ambas configuraciones y que los nodos puedan verse entre sí |
| El servidor con mayor prioridad no recupera la IP al volver a estar disponible | Es el comportamiento esperado si se configuró `nopreempt`, o puede tardar unos segundos según `advert_int` | Revisar si se agregó `nopreempt` intencionalmente; si no, esperar el intervalo de anuncio configurado |
| La IP conmuta, pero el sitio no carga en el nodo de respaldo | Apache no está activo en ese nodo, o su `index.html` no existe | Verificar `systemctl status httpd` y la presencia del archivo en `/var/www/html/` en ambos servidores |
| `vrrp_script` no detecta que Apache cayó | El servicio real se llama distinto al verificado por `killall -0 httpd` | Ajustar el script de verificación al nombre real del proceso en uso |

---

[← Volver al laboratorio](../README.md)
