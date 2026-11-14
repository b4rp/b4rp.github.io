---
# Cómo usar esta plantilla:
# 1. Copia este archivo a la carpeta _posts/
# 2. Renómbralo con el formato AAAA-MM-DD-titulo.md (ej: 2026-10-24-htb-nombre-maquina.md)
# 3. Reemplaza todo lo que esté entre < >
# 4. Borra las secciones que no uses y estos comentarios
# 5. No publiques flags ni hashes reales: muestra el comando `cat`, pero oculta el contenido
# 6. Solo máquinas/retos retirados
title: "HTB: <Nombre de la máquina>"
date: <AAAA-MM-DD HH:MM:SS> -0300
categories: [Hack The Box, Machines] # o [TryHackMe, Rooms] / [CTF, <Nombre del evento>]
tags: [<linux|windows>, <easy|medium|hard|insane>, <web>, <active-directory>]
description: <Resumen de una línea: vector de entrada y escalada.>
media_subpath: /assets/img/posts/<nombre-maquina>/ # carpeta donde guardas las imágenes de este post
image:
  path: banner.png # imagen de portada (opcional, borra estas 3 líneas si no tienes)
  alt: <Nombre de la máquina>
---

## Información

| Campo | Valor |
| :---- | :---- |
| Plataforma | Hack The Box |
| Sistema operativo | <Linux / Windows> |
| Dificultad | <Easy / Medium / Hard / Insane> |
| IP | `10.10.11.XXX` |
| Fecha de retiro | <AAAA-MM-DD> |

> Resumen: <2-3 líneas con el camino completo. Ej: "Un LFI en la aplicación web permite leer credenciales, que se reutilizan por SSH. Un binario SUID mal configurado lleva a root.">
{: .prompt-info }

## Reconocimiento

### Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.11.XXX -oG allPorts
nmap -sCV -p<puertos> 10.10.11.XXX -oN targeted
```

```text
<pega aquí el output relevante de nmap>
```

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 22 | ssh | <versión> |
| 80 | http | <versión> |

### Enumeración web

<Qué hay en la web, tecnologías detectadas, virtual hosts, directorios.>

```bash
whatweb http://10.10.11.XXX
ffuf -u http://10.10.11.XXX/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

![Página principal](web-index.png)
_<Descripción de la captura>_

## Explotación

### <Nombre de la vulnerabilidad>

<Explica qué encontraste, por qué es vulnerable y cómo lo explotaste.>

```http
<request de Burp, si aplica>
```

```bash
<comandos / exploit>
```

> <Tip o detalle importante que te costó descubrir.>
{: .prompt-tip }

## Acceso inicial

<Cómo obtuviste la shell: reverse shell, credenciales, etc.>

```bash
nc -lvnp 443
```

```bash
<usuario>@<host>:~$ cat user.txt
```

## Movimiento lateral

<Si aplica: cómo pasaste de un usuario a otro. Si no, borra esta sección.>

## Escalada de privilegios

### Enumeración

```bash
id
sudo -l
find / -perm -4000 2>/dev/null
```

<Qué encontraste: SUID, sudo, capabilities, cron, servicios internos, etc.>

### Explotación

```bash
<comandos para escalar>
```

```bash
root@<host>:~# cat root.txt
```

## Lecciones aprendidas

- <Qué aprendiste en esta máquina>
- <Qué técnica nueva usaste>
- <Qué harías distinto>

## Mitigación

- <Cómo se corrige la vulnerabilidad de entrada>
- <Cómo se corrige la escalada>

## Referencias

- [<Título>](<https://enlace>)
