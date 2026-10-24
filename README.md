<div align="center">

# Linux / RHEL — Sistemas Operativos

![RHEL](https://img.shields.io/badge/RED%20HAT-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![Linux](https://img.shields.io/badge/LINUX-TERMINAL-f472b6?style=flat-square&logo=linux&logoColor=white)
![Bash](https://img.shields.io/badge/BASH-SCRIPTING-60a5fa?style=flat-square&logo=gnubash&logoColor=white)
![Infrastructure](https://img.shields.io/badge/SYSTEMS-ADMINISTRATION-8b5cf6?style=flat-square)
![Estado](https://img.shields.io/badge/ESTADO-EN%20PROGRESO-fbbf24?style=flat-square)

</div>

---

## Descripción general

**Linux / RHEL — Sistemas Operativos** reúne los comandos y laboratorios documentados durante los módulos de la materia de Sistemas Operativos, centrados en **Red Hat Enterprise Linux (RHEL)**.

El contenido cubre la administración completa de un sistema Linux: gestión de usuarios y permisos, administración de paquetes y servicios, configuración de redes, almacenamiento y particiones, automatización con Bash, y un acercamiento a VoIP e Infraestructura como Código (IaC).

Cada práctica se documenta con los comandos utilizados, capturas de la terminal como evidencia y una explicación de lo realizado.

**Estado del proyecto: en progreso.** Las prácticas se van agregando a medida que se completan y documentan.

---

## Prácticas implementadas

| # | Práctica | Tecnologías y alcance |
|:--:|---|---|
| 01 | [Administración de usuarios y permisos](./01-user-group-management) | Creación y gestión de usuarios y grupos en RHEL, privilegios de `sudo` vía `/etc/sudoers`, eliminación y administración de contraseñas. |
| 02 | [Gestión de paquetes y servicios](./02-package-service-management) | Administración de paquetes y gestión de servicios con `systemctl` en RHEL. |
| 03 | [Redes en Linux](./03-linux-networking) | Configuración y diagnóstico de redes: interfaces, direccionamiento IP, rutas y herramientas de diagnóstico. |
| 04 | [Almacenamiento y particiones](./04-storage-partitions) | Gestión de discos, particiones, sistemas de archivos y puntos de montaje. |
| 05 | [Scripting en Bash](./05-bash-scripting) | Automatización de tareas administrativas mediante scripts de Bash. |
| 06 | [VoIP e IaC](./06-voip-iac) | Configuración de un servicio de VoIP y aprovisionamiento de infraestructura mediante IaC. |
| 07 | [Endurecimiento de SSH, contraseñas y firewall](./07-ssh-password-firewall-hardening) | Restricción del acceso root por SSH, timeout por inactividad, política de contraseñas, expiración y firewall con firewalld. |
| 08 | [OpenSCAP: PCI-DSS con OpenSCAP y Nessus](./08-openscap-nessus-pcidss) | Auditoría de cumplimiento PCI-DSS con OpenSCAP/SCAP Workbench, verificación cruzada con Nessus, y remediación automática y manual de vulnerabilidades en RHEL. |

---

## Demostraciones y evidencias

Los siguientes enlaces permiten acceder directamente a las demostraciones en video de las prácticas disponibles.

| Práctica | Demostración |
|---|---|
| 01 · Administración de usuarios y permisos | [Ver en YouTube](https://www.youtube.com/watch?v=8s3UGKai9P4&list=PLIJEt4hLYnOUAQxFim_nNNvPqBw8Z8VKT) |
| 02 · Antivirus ClamAV | [Ver en YouTube](https://www.youtube.com/watch?v=WTApO2vpP38) |
| 07 · Endurecimiento de SSH, contraseñas y firewall | [Ver en YouTube](https://www.youtube.com/watch?v=NtuHPQcl41I) |
| 08 · Examen Final: PCI-DSS con OpenSCAP y Nessus | [Ver en YouTube](https://www.youtube.com/watch?v=XeZdCIBIb3E) |

Las demostraciones de las prácticas 03 a 06 se agregarán cuando estén disponibles.

---

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Acceso | Terminal local |

> [!NOTE]
> Todas las prácticas se realizaron en un entorno aislado de laboratorio, con fines educativos.

---

## Estructura del repositorio

```text
linux-rhel-sistemas-operativos/
├── 01-user-group-management/
├── 02-package-service-management/
├── 03-linux-networking/
├── 04-storage-partitions/
├── 05-bash-scripting/
├── 06-voip-iac/
├── 07-ssh-password-firewall-hardening
├── 08-openscap-nessus-pcidss/
├── LICENSE
└── README.md
```

Cada carpeta contiene un `README.md` con el objetivo, el entorno, el procedimiento, las evidencias y lo aprendido en la práctica.

---

## Conclusión

Linux / RHEL — Sistemas Operativos reúne el trabajo práctico realizado sobre administración de sistemas Linux, desde la gestión de usuarios y permisos hasta la automatización y la infraestructura como código.

La documentación registra los comandos utilizados, las decisiones técnicas y las evidencias obtenidas en la terminal. El resultado es un repositorio de referencia para revisar los procedimientos realizados y demostrar experiencia práctica con administración de sistemas Linux.

---

## Licencia

Este proyecto se distribuye bajo la [licencia MIT](./LICENSE).

---

## 👨‍💻 Autor

**Neury Jorge Montero Mejía**

*Estudiante de Tecnólogo en Seguridad Informática*

[![LinkedIn](https://img.shields.io/badge/LINKEDIN-NEURY%20MONTERO-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/neuryjmm)
[![GitHub](https://img.shields.io/badge/GITHUB-NEURYJMM-111111?style=flat-square&logo=github&logoColor=white)](https://github.com/NeuryJMM)

---
