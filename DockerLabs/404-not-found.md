# Máquina 404-not-found

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/cCU02YzZ#UQgH3W1lHg-K8PbRRMokRwzP-hKpYI1ae6MWNAQkOj8)

![imagen](https://github.com/user-attachments/assets/6d07b817-7094-41f2-87da-6b95ba21c993)

- **Nombre de la Máquina:** 404-not-found  
- **Sistema Operativo:** Linux  
- **Dificultad:** Medio  
- **Plataforma:** DockerLabs  
- **Dirección IP:** 172.17.0.2  
- **Vulnerabilidades Explotadas:**
  - Inyección LDAP en el formulario de inicio de sesión para eludir la autenticación y obtener credenciales de usuario.
- **Escalada de Privilegios:**
  - Ejecución de comandos a través de un script de calculadora con privilegios elevados, obteniendo acceso a la cuenta de `200-ok` y utilizando una contraseña encontrada en el sistema para obtener acceso root.

## Reconocimiento y Enumeración

Comenzamos realizando un escaneo general utilizando Nmap para identificar los puertos abiertos en la IP del objetivo:

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/ae1d1185-71a7-406e-a55b-876f24ab8cfe)

El escaneo reveló que los puertos 22 (SSH) y 80 (HTTP) están abiertos. Para un análisis más profundo, realizamos un escaneo enfocado en estos puertos para identificar los servicios y versiones que se ejecutan en ellos:

```bash
sudo nmap -p 22,80 -sCV 172.17.0.2 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/89d57419-9ae0-42bb-9b66-df4895ecbd87)

### Puerto 80 HTTP

Utilizamos `whatweb` para obtener más detalles sobre el servicio que corre en el puerto 80:

![imagen](https://github.com/user-attachments/assets/bf3d3928-9bb3-4ca4-911f-5c063d81d3b7)

Observamos que la página web redirige a un dominio llamado `404-not-found.hl`, por lo que lo añadimos al archivo `/etc/hosts`:

```bash
nano /etc/hosts
```

![imagen](https://github.com/user-attachments/assets/3ae8e486-7c6c-4d52-99a5-bc383594980d)

Una vez agregado el dominio, accedemos a la web. Tras investigar la página y realizar fuzzing para buscar directorios interesantes, encontramos un directorio `/participar.html`, aunque inicialmente parecía ser un "rabbit hole".

![imagen](https://github.com/user-attachments/assets/aba678d1-cd2d-40f3-a7b7-5bf11e5a7466)

Al descifrar una clave base64, obtuvimos el mensaje "Que haces?, mira en la URL". Esto nos llevó a buscar subdominios mediante fuzzing con `wfuzz`:

```bash
wfuzz -c -w /usr/share/wordlists/dirb/common.txt -H "Host: FUZZ.404-not-found.hl" -u http://404-not-found.hl/ -t 100 --hc 301,400
```

![imagen](https://github.com/user-attachments/assets/e47e53f0-22e4-40c9-9008-a74d1f90a828)

Descubrimos un subdominio llamado `info`, que también agregamos al archivo `/etc/hosts`:

![imagen](https://github.com/user-attachments/assets/189710aa-829a-4aca-affe-bd3ed6dfc0f2)

Al acceder a `info.404-not-found.hl`, encontramos una página de inicio de sesión, junto con un comentario en el código fuente indicando que el login usa LDAP:

```html
<!-- I believe this login works with LDAP -->
```

![imagen](https://github.com/user-attachments/assets/cebbb98e-f884-45cd-bde6-cf487e5a63a9)

## Explotación

Utilizamos una [inyección LDAP](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/LDAP%20Injection/README.md) para eludir la autenticación. Ingresando `*)(uid=*))(|(uid=*` como nombre de usuario y cualquier contraseña, logramos iniciar sesión exitosamente:

![imagen](https://github.com/user-attachments/assets/d0b4adf0-2317-4efc-95fe-eb8ec5fff322)

![imagen](https://github.com/user-attachments/assets/2ba6a0db-af92-415c-bd5c-e2f400366fab)

Esto nos dio acceso a credenciales SSH para el usuario `404-page`:

```
user: 404-page
password: not-found-page-secret
```

Nos conectamos al servidor mediante SSH utilizando dichas credenciales:

```bash
ssh 404-page@172.17.0.2
```

![imagen](https://github.com/user-attachments/assets/bb0fe776-37b3-4edf-acab-5a0fae82c0dd)

## Escalada de Privilegios

Una vez dentro del sistema, verificamos los usuarios del sistema con:

```bash
cat /etc/passwd | grep bash
```

![imagen](https://github.com/user-attachments/assets/cec1acf8-c07c-4eed-bed2-7e23f71d6274)

Luego, utilizamos `sudo -l` para comprobar los permisos sudo disponibles. Descubrimos que podíamos ejecutar el script `/home/404-page/calculator.py` como el usuario `200-ok`:

![imagen](https://github.com/user-attachments/assets/f8bded7f-17f9-4537-b2af-36a18648a72a)

Ejecutamos el script como `200-ok` y descubrimos que se trataba de una calculadora funcional:

![imagen](https://github.com/user-attachments/assets/65182ed0-44a6-4ee1-a715-7bc8c4c97a4f)

Investigando más el sistema, encontramos un archivo en `/var/www` que contenía un mensaje indicando que el símbolo "!" seguido de un comando podía ser ejecutado en la calculadora, pero solo `200-ok` lo sabía. Probamos ingresando `!bash` y obtuvimos una shell como el usuario `200-ok`:

```bash
sudo -u 200-ok ./calculator.py
!bash
```

![imagen](https://github.com/user-attachments/assets/8a710a91-7934-49b4-b3e4-75d3c7a10fb3)

Finalmente, para escalar a root, encontramos dos archivos `.txt` en el directorio personal de `200-ok`. Uno de ellos contenía un hash y el otro una pista. Probamos con la palabra `rooteable` como contraseña para `root`, y con ello obtuvimos acceso root, completando el reto:

![imagen](https://github.com/user-attachments/assets/5621354e-9aa3-4d72-ade7-e4fce75d7c25)
