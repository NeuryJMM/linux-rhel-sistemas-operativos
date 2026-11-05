[← Volver al laboratorio](../README.md)

# 15 · Cifrado de archivos con GPG

![RHEL](https://img.shields.io/badge/Red%20Hat-RHEL-c084fc?style=flat-square&logo=redhat&logoColor=white)
![GPG](https://img.shields.io/badge/GnuPG-Cifrado%20Sim%C3%A9trico-f472b6?style=flat-square)
![Seguridad](https://img.shields.io/badge/Seguridad-Confidencialidad-60a5fa?style=flat-square)
![Estado](https://img.shields.io/badge/Estado-Completado-34d399?style=flat-square)

Cifrado y descifrado de un archivo de texto en RHEL usando **GPG2** (GnuPG), con cifrado **simétrico** protegido por una frase de contraseña.

## Objetivo

Proteger la confidencialidad de un archivo cifrándolo con GPG, de forma que su contenido sea ilegible sin la contraseña usada para cifrarlo, y luego recuperar el contenido original descifrándolo.

## Entorno

| Elemento | Valor |
|---|---|
| Distribución | RHEL (Red Hat Enterprise Linux) |
| Herramienta | GPG2 (GnuPG) |
| Tipo de cifrado | Simétrico (una sola contraseña, compartida para cifrar y descifrar) |

## Índice

- [Parte A: Instalar GPG2](#parte-a-instalar-gpg2)
- [Parte B: Crear el archivo de prueba](#parte-b-crear-el-archivo-de-prueba)
- [Parte C: Cifrar el archivo](#parte-c-cifrar-el-archivo)
- [Parte D: Descifrar el archivo](#parte-d-descifrar-el-archivo)
- [Demostración en video](#demostración-en-video)
- [Referencia de comandos](#referencia-de-comandos)
- [Solución de problemas frecuentes](#solución-de-problemas-frecuentes)

---

## Parte A: Instalar GPG2

**1.** Instalamos la herramienta:

```bash
sudo dnf install gnupg2 -y
```

> [!NOTE]
> GPG (*GNU Privacy Guard*) es una implementación libre del estándar OpenPGP. Admite dos modos de cifrado: **simétrico** (una sola contraseña, la misma para cifrar y descifrar, usada en esta práctica) y **asimétrico** (un par de claves pública/privada, donde cualquiera puede cifrar con la clave pública, pero solo el dueño de la clave privada puede descifrar). El cifrado simétrico es más simple y suficiente para proteger un archivo propio; el asimétrico es el que se usa para enviar información cifrada a otra persona sin compartir una contraseña con ella.

---

## Parte B: Crear el archivo de prueba

**1.** Nos ubicamos en el escritorio y creamos un archivo de texto simple:

```bash
cd ./Desktop
echo "Neury" > prueba.txt
```

**2.** Verificamos que el archivo se haya creado:

```bash
ls -l
cat prueba.txt
```

> [!NOTE]
> El comando original de esta práctica decía `cat archivo.txt`, pero el archivo se creó como `prueba.txt`. Como ambos nombres son distintos, `cat archivo.txt` habría devuelto `No such file or directory`. El comando correcto para verificar el contenido, y el que se usa de aquí en adelante, es `cat prueba.txt`.

---

## Parte C: Cifrar el archivo

**1.** Ciframos `prueba.txt` de forma simétrica:

```bash
gpg2 -c prueba.txt
```

El sistema solicita una contraseña para cifrar el archivo, y pide escribirla **dos veces** (para confirmar que no hubo un error de tipeo). Esto genera un nuevo archivo: `prueba.txt.gpg`.

**2.** Verificamos que el archivo cifrado se haya creado correctamente. Deben aparecer tanto el original como el cifrado:

```bash
ls
```

```text
prueba.txt  prueba.txt.gpg
```

**3.** Intentamos leer el archivo cifrado directamente:

```bash
cat prueba.txt.gpg
```

El contenido se muestra como datos binarios ilegibles, confirmando que el cifrado funcionó: sin la contraseña, el archivo no revela ninguna información.

> [!WARNING]
> Cifrar un archivo **no elimina el original**. En este laboratorio, `prueba.txt` (sin cifrar) sigue existiendo en el mismo directorio que `prueba.txt.gpg`, así que cualquiera con acceso a esa carpeta puede simplemente leer `prueba.txt` sin necesidad de la contraseña. En un uso real, después de confirmar que el archivo cifrado se creó correctamente, el original debe eliminarse de forma segura (por ejemplo, con `shred -u prueba.txt`, que sobrescribe el contenido antes de borrarlo) para que el cifrado proteja realmente la información.

---

## Parte D: Descifrar el archivo

**1.** Descifrarnos el archivo para recuperar su contenido original:

```bash
gpg2 -d prueba.txt.gpg
```

Se solicita la **misma** contraseña usada al cifrar. Una vez ingresada correctamente, el contenido original (`Neury`) se muestra en pantalla.

> [!TIP]
> `gpg2 -d` solo **muestra** el contenido descifrado en la terminal (no crea un archivo nuevo). Para guardar el resultado descifrado en un archivo, se usa la bandera `-o` con el nombre de salida deseado: `gpg2 -d -o prueba_descifrado.txt prueba.txt.gpg`.

---

## Demostración en video

Esta práctica cuenta con una demostración en video donde se muestra el proceso completo del laboratorio.

**Video de la práctica:** [Ver demostración en YouTube](https://www.youtube.com/watch?v=uiblI8cKXT0&list=PLIJEt4hLYnOXYtfF8MnxM5sMboz-KI20D&index=4)

---

## Referencia de comandos

| Comando | Función |
|---|---|
| `dnf install gnupg2 -y` | Instala GPG2 |
| `gpg2 -c <archivo>` | Cifra un archivo de forma simétrica (pide y confirma una contraseña), generando `<archivo>.gpg` |
| `gpg2 -d <archivo>.gpg` | Descifra un archivo y muestra su contenido en pantalla |
| `gpg2 -d -o <salida> <archivo>.gpg` | Descifra un archivo y guarda el resultado en un archivo nuevo |
| `shred -u <archivo>` | Sobrescribe y elimina de forma segura un archivo (útil para borrar el original tras cifrarlo) |

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| `cat: archivo.txt: No such file or directory` | El nombre usado no coincide con el del archivo creado | Verificar el nombre exacto del archivo con `ls` antes de usar `cat` |
| `gpg2 -d` pide la contraseña y la rechaza | Se escribió una contraseña distinta a la usada al cifrar | No hay forma de recuperar el archivo sin la contraseña correcta; GPG no tiene una puerta trasera para esto |
| El archivo cifrado sigue siendo legible | Se está leyendo por error el archivo original (`prueba.txt`), no el `.gpg` | Confirmar con `ls` cuál de los dos archivos se está abriendo |
| Se perdió el archivo original sin haber verificado el cifrado | Se eliminó `prueba.txt` antes de confirmar que `prueba.txt.gpg` descifra correctamente | Siempre probar `gpg2 -d` sobre el archivo cifrado **antes** de eliminar el original |

---

[← Volver al laboratorio](../README.md)
