[← Volver al laboratorio](../README.md)

# 13 · Clúster de alta disponibilidad con Pacemaker y PCS

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![Pacemaker](https://img.shields.io/badge/Pacemaker-High%20Availability-f472b6?style=flat-square)
![PCS](https://img.shields.io/badge/PCS-Cluster%20Management-60a5fa?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Configuración de un **clúster de alta disponibilidad (HA)** entre dos servidores RHEL usando **Pacemaker** (el gestor de recursos del clúster) y **PCS** (*Pacemaker/Corosync Configuration System*, la herramienta de administración). El clúster gestiona una **IP virtual flotante** que se transfiere automáticamente al nodo que siga disponible si el otro falla.

## Objetivo

Levantar un clúster de dos nodos y verificar que una dirección IP virtual permanece accesible de forma continua, incluso cuando uno de los servidores se reinicia o deja de responder.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Nodos | 2 servidores: principal (`192.168.0.16`) y secundario (`192.168.0.17`) |
| Software de clúster | Pacemaker + PCS (Corosync como capa de mensajería) |
| Nombre del clúster | `cluster-nmontero` |
| Recurso gestionado | IP virtual flotante: `192.168.0.10/24` |

## Índice

- [Parte A: Preparar ambos nodos](#parte-a-preparar-ambos-nodos)
- [Parte B: Autenticar y crear el clúster](#parte-b-autenticar-y-crear-el-clúster)
- [Parte C: Configurar la IP virtual flotante](#parte-c-configurar-la-ip-virtual-flotante)
- [Parte D: Probar la conmutación por error (failover)](#parte-d-probar-la-conmutación-por-error-failover)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Preparar ambos nodos

> [!NOTE]
> Todos los comandos de esta sección se ejecutan **en los dos servidores** (el principal y el secundario), salvo que se indique lo contrario.

**1.** Habilitamos el repositorio de alta disponibilidad, necesario para instalar Pacemaker:

```bash
sudo subscription-manager repos --enable=rhel-8-for-x86_64-highavailability-rpms
```

**2.** Instalamos Pacemaker y PCS:

```bash
sudo dnf install -y pacemaker pcs
```

**3.** Habilitamos e iniciamos el servicio de administración del clúster (`pcsd`):

```bash
sudo systemctl enable pcsd
sudo systemctl start pcsd
```

**4.** Configuramos la contraseña del usuario `hacluster`, creado automáticamente al instalar PCS y usado para autenticar los nodos entre sí:

```bash
sudo passwd hacluster
```

> [!IMPORTANT]
> La contraseña de `hacluster` debe ser la **misma en ambos servidores**, ya que se usa para la autenticación mutua entre nodos en el siguiente paso.

**5.** Permitimos el tráfico del clúster en el firewall:

```bash
sudo firewall-cmd --permanent --add-service=high-availability
sudo firewall-cmd --reload
```

---

## Parte B: Autenticar y crear el clúster

> [!NOTE]
> Los comandos de esta sección se ejecutan **solo en uno de los dos servidores** (el principal); PCS se encarga de aplicar la configuración en ambos nodos.

**1.** Autenticamos los nodos entre sí usando el usuario `hacluster` (se solicitará su contraseña):

```bash
sudo pcs cluster auth 192.168.0.16 192.168.0.17 -u hacluster
sudo pcs host auth 192.168.0.16 192.168.0.17 -u hacluster
```

**2.** Creamos el clúster, asignándole un nombre e indicando los nodos que lo componen:

```bash
sudo pcs cluster setup cluster-nmontero 192.168.0.16 192.168.0.17
```

**3.** Habilitamos el clúster (para que inicie automáticamente en cada nodo al arrancar) y lo iniciamos en ambos servidores:

```bash
sudo pcs cluster enable --all
sudo pcs cluster start --all
```

**4.** Verificamos el estado del clúster (se puede ejecutar desde cualquiera de los dos nodos):

```bash
sudo pcs status
```

**5.** Deshabilitamos **STONITH** (*Shoot The Other Node In The Head*), el mecanismo de aislamiento de nodos fallidos. Este paso es opcional y se usa aquí únicamente para simplificar el laboratorio.

```bash
sudo pcs property set stonith-enabled=false
```

> [!WARNING]
> **STONITH existe por una razón.** Es el mecanismo que evita el problema de **"cerebro dividido"** (*split-brain*): si dos nodos pierden comunicación entre sí pero ambos siguen activos, cada uno podría asumir que es el único disponible y tomar control del mismo recurso (en este caso, la IP flotante) al mismo tiempo, causando conflictos de red. Deshabilitarlo es aceptable en un laboratorio aislado, pero en un clúster de producción debe configurarse un dispositivo de aislamiento real (*fencing*) en vez de desactivarlo.

---

## Parte C: Configurar la IP virtual flotante

**1.** Creamos el recurso de IP flotante, usando el agente `IPaddr2` de Pacemaker, y configuramos una verificación de su disponibilidad cada 30 segundos:

```bash
sudo pcs resource create IP_Flotante ocf:heartbeat:IPaddr2 ip=192.168.0.10 cidr_netmask=24 op monitor interval=30s
```

Este recurso es, en esencia, una dirección IP que Pacemaker asigna a la interfaz de red de **uno** de los dos nodos a la vez, y que mueve automáticamente al otro nodo si el que la tiene deja de responder.

---

## Parte D: Probar la conmutación por error (failover)

**1.** Desde el equipo host (fuera del clúster), hacemos un `ping` continuo a la IP virtual:

```bash
ping -t 192.168.0.10
```

**2.** Mientras el `ping` sigue corriendo, reiniciamos los servidores **de forma intercalada** (uno a la vez, nunca ambos simultáneamente): primero uno, se espera a que la IP se transfiera, y luego el otro.

**3.** Si la configuración es correcta, el `ping` **no se interrumpe**: Pacemaker detecta que el nodo que tenía la IP virtual dejó de estar disponible y la reasigna automáticamente al nodo restante, dentro del intervalo de monitoreo configurado (30 segundos).

> [!NOTE]
> La bandera `-t` de `ping` (en Windows) lo mantiene enviando solicitudes de forma indefinida, en vez de detenerse después de unas pocas; es la que permite observar la transferencia de la IP en tiempo real, en vez de solo comprobar la disponibilidad en un instante puntual. En Linux, `ping` ya se comporta así por defecto (continuo hasta `Ctrl+C`), sin necesidad de ninguna bandera adicional.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio, incluida la prueba de conmutación por error.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=RLBTWSW_8RM&list=PLIJEt4hLYnOVz-2-LYxB4AvbZ1uhmkkcM&index=2)

---

## Referencia de comandos

| Comando | Función |
|---|---|
| `subscription-manager repos --enable=...` | Habilita un repositorio adicional (en este caso, el de alta disponibilidad) |
| `dnf install -y pacemaker pcs` | Instala Pacemaker y la herramienta de administración PCS |
| `systemctl enable/start pcsd` | Habilita e inicia el servicio de administración del clúster |
| `passwd hacluster` | Establece la contraseña del usuario usado para autenticar nodos |
| `firewall-cmd --add-service=high-availability` | Permite el tráfico necesario para la comunicación del clúster |
| `pcs host auth <ips> -u hacluster` | Autentica los nodos entre sí |
| `pcs cluster setup <nombre> <ips>` | Crea el clúster con los nodos indicados |
| `pcs cluster enable/start --all` | Habilita e inicia el clúster en todos los nodos |
| `pcs status` | Muestra el estado actual del clúster y sus recursos |
| `pcs property set stonith-enabled=false` | Deshabilita STONITH (solo para laboratorio) |
| `pcs resource create <nombre> ocf:heartbeat:IPaddr2 ip=<ip> ...` | Crea un recurso de IP virtual flotante |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| `pcs host auth` falla con error de autenticación | La contraseña de `hacluster` no coincide entre los nodos | Verificar que se haya configurado la **misma** contraseña en ambos servidores con `passwd hacluster` |
| `pcs cluster start` no inicia el clúster | El servicio `pcsd` no está activo, o el firewall bloquea la comunicación | Verificar `systemctl status pcsd` y que el servicio `high-availability` esté permitido en el firewall de ambos nodos |
| `pcs status` muestra el recurso `IP_Flotante` en estado de fallo | La interfaz de red no coincide, o la IP ya está en uso en la red | Revisar que `cidr_netmask` sea correcto y que la IP flotante no choque con ningún otro equipo de la red |
| El `ping` se pierde al reiniciar un nodo | STONITH sigue habilitado y aísla ambos nodos, o `stonith-enabled=false` no se aplicó | Confirmar con `pcs property show stonith-enabled`; revisar que el recurso tenga configurado correctamente el intervalo de monitoreo |
| Ambos nodos intentan controlar la IP flotante al mismo tiempo (*split-brain*) | STONITH deshabilitado sin un mecanismo de aislamiento alternativo | Esperado en este laboratorio sin fencing real; en producción, configurar un dispositivo STONITH adecuado |

---

[← Volver al laboratorio](../README.md)
