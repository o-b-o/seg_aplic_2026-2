# Reporte de laboratorio — CVE-2025-32433 en Erlang/OTP SSH

## 1. Resumen ejecutivo

En esta práctica se analizó y validó de forma controlada la vulnerabilidad **CVE-2025-32433**, presente en el servidor SSH de **Erlang/OTP**. El entorno vulnerable se desplegó localmente con **Docker Desktop** usando el escenario de **Vulhub** para este CVE.

Se eligió este caso por tres razones: es un **CVE reciente**, existe **documentación técnica suficiente** para entender la falla sin depender exclusivamente de una demostración ya resuelta, y podía montarse de forma razonable en un entorno aislado mediante contenedores. Además, su impacto es claro y académico: permite estudiar cómo una implementación de SSH puede procesar mensajes de sesión antes de autenticar al usuario.

Durante la fase de reconocimiento se identificó un servicio **SSH** expuesto en el **puerto 2222/tcp**, cuyo banner indicó una implementación **Erlang**. Posteriormente se observó el flujo normal de una conexión SSH para distinguir las fases de **transporte**, **negociación criptográfica** y **autenticación**. A partir de ello se formuló la hipótesis de que el servidor podría estar procesando mensajes de la capa de conexión antes de completar la autenticación del usuario.

El análisis del PoC mostró que este simula un cliente SSH mínimo: intercambia banner, envía un `KEXINIT` válido y, sin completar autenticación, introduce mensajes `SSH_MSG_CHANNEL_OPEN` y `SSH_MSG_CHANNEL_REQUEST` de tipo `exec`. En un servidor sano, dichos mensajes deberían ser rechazados o provocar desconexión. En el entorno vulnerable, en cambio, se observó un cambio real en el sistema objetivo, consistente con **ejecución remota de código sin autenticación previa**.

---

## 2. Dato duro del caso

- **CVE:** CVE-2025-32433
- **Software afectado:** Erlang/OTP SSH
- **Versión vulnerable usada en laboratorio:** `vulhub/erlang:27.3.2-with-ssh`
- **Puerto expuesto en el escenario:** `2222/tcp`
- **Naturaleza del impacto:** Ejecución remota de código sin autenticación
- **Versiones corregidas reportadas públicamente:** 27.3.3, 26.2.5.11 y 25.3.2.20
- **CWE asociado:** **CWE-306 — Missing Authentication for Critical Function**

---

## 3. Objetivo del laboratorio

El objetivo fue reproducir y comprender, en un entorno controlado, cómo una implementación vulnerable del servidor SSH de Erlang/OTP puede procesar mensajes del protocolo de conexión **antes** de completar la autenticación del usuario, permitiendo la ejecución de una función crítica de forma no autenticada.

Más que limitarse a “correr un exploit”, el enfoque fue reconstruir el razonamiento técnico:

1. descubrir el servicio expuesto;
2. identificar su naturaleza;
3. observar el flujo normal del protocolo;
4. detectar qué parte del flujo se rompe;
5. validar que esa ruptura tiene impacto real.

---

## 4. Entorno de pruebas

### 4.1 Infraestructura utilizada

- **Host principal:** Windows
- **Plataforma de contenedores:** Docker Desktop
- **Escenario vulnerable:** Vulhub → `erlang/CVE-2025-32433`
- **Máquina de observación/análisis:** Kali Linux en equipo separado dentro de la misma red local

### 4.2 Ruta del escenario

Se trabajó desde la carpeta:

```powershell
C:\Users\jose_\OneDrive\Documentos\MestriaPCIC\Semestre 2\Seguridad\Labs\vulhub\erlang\CVE-2025-32433
```

### 4.3 Verificación del entorno

La configuración del escenario mostró un único servicio:

- imagen: `vulhub/erlang:27.3.2-with-ssh`
- servicio: `sshd`
- puerto publicado: `0.0.0.0:2222 -> 2222/tcp`

### 4.4 Aislamiento del laboratorio

La práctica se realizó únicamente sobre un contenedor local y una máquina de observación dentro de la misma red privada. No se intentó interactuar con sistemas de terceros. La exposición del servicio se limitó al laboratorio montado para esta actividad.

### 4.5 Evidencia sugerida para anexar

- Captura de `docker compose config`
- Captura de `docker compose ps`
- Captura de `docker ps`
- Captura de Docker Desktop mostrando el contenedor activo

---

## 5. Fase de reconocimiento

### 5.1 Descubrimiento en red

Desde la máquina Kali se realizó reconocimiento del segmento local para identificar hosts vivos. Después se efectuó un escaneo de puertos del objetivo y se encontró:

- **Puerto 2222/tcp abierto**
- **Servicio identificado:** SSH

### 5.2 Fingerprinting del servicio

La observación del banner mostró que el servicio no era un OpenSSH común, sino una implementación Erlang:

```text
SSH-2.0-Erlang/5.2.9
```

Este detalle fue importante porque orientó el análisis hacia una implementación específica del protocolo SSH, distinta de las más comunes en sistemas Linux convencionales.

### 5.3 Interpretación del hallazgo inicial

Hasta este punto, el conocimiento disponible era:

- existe un servicio SSH expuesto;
- la implementación es Erlang;
- el puerto de escucha es no estándar (`2222`);
- el servicio podría ser interesante para análisis más profundo por no tratarse de una implementación típica.

---

## 6. Observación del flujo normal de SSH

Para entender el comportamiento esperado del servidor, se estableció una conexión SSH normal en modo verbose desde Kali.

### 6.1 Fases observadas

La traza permitió distinguir claramente estas fases:

1. **Conexión TCP**
2. **Intercambio de banners**
3. **Negociación criptográfica**
   - `SSH_MSG_KEXINIT`
   - selección de algoritmos
   - intercambio de claves
   - `SSH_MSG_NEWKEYS`
4. **Inicio del servicio de autenticación**
   - `ssh-userauth`
5. **Solicitud de credenciales**
   - métodos ofrecidos: `publickey`, `password`

### 6.2 Qué se aprendió de esa observación

La observación de una sesión normal permitió establecer una línea base:

- antes de autenticación se negocia transporte y criptografía;
- posteriormente se entra al servicio de autenticación;
- las acciones de usuario o de sesión no deberían procesarse todavía.

Este punto fue crucial para formular la hipótesis de vulnerabilidad.

---

## 7. Hipótesis técnica

Una vez identificado el flujo normal, se razonó que las operaciones asociadas a **sesión**, **canales** y **ejecución remota** pertenecen a una fase posterior del protocolo SSH.

Por tanto, la hipótesis de trabajo fue:

> Si esta implementación del servidor SSH acepta mensajes de apertura de canal o de ejecución remota antes de completar la autenticación del usuario, entonces existe una falla crítica de autenticación/estado.

En otras palabras:

- **transporte y negociación** son esperables antes de autenticación;
- **sesión y ejecución** no deberían ser válidas en esa fase.

---

## 8. Comprensión conceptual del PoC

El archivo `exploit.py` no implementa un cliente SSH completo. Lo que hace es **simular lo suficiente** del arranque del protocolo para parecer un cliente legítimo al inicio y después enviar mensajes que pertenecen a una fase posterior.

### 8.1 Fuente del PoC

Se utilizó el script `exploit.py` incluido en el propio escenario de **Vulhub** para `erlang/CVE-2025-32433`. Antes de utilizarlo se revisó su estructura para entender qué hacía y qué mensajes del protocolo construía.

### 8.2 Qué emula

El PoC realiza, de forma conceptual, esta secuencia:

1. abre una conexión TCP al objetivo;
2. envía un banner SSH válido;
3. construye y envía un `SSH_MSG_KEXINIT`;
4. construye un `SSH_MSG_CHANNEL_OPEN` de tipo `session`;
5. construye un `SSH_MSG_CHANNEL_REQUEST` de tipo `exec`;
6. todo ello **sin completar la autenticación del usuario**.

### 8.3 Qué significa eso

El PoC intenta comprobar si el servidor procesa mensajes del **SSH Connection Protocol** antes del punto en que debería exigir una identidad válida.

La lógica del hallazgo no es “romper SSH”, sino esta:

- hacerse pasar por un cliente SSH válido al principio;
- alcanzar la fase donde el servidor aún no ha autenticado al usuario;
- insertar mensajes de sesión/ejecución;
- observar si el servidor los trata como válidos.

### 8.4 Intuición técnica

En un servidor sano, mensajes como:

- `CHANNEL_OPEN`
- `CHANNEL_REQUEST`
- `exec`

deberían **rechazarse o provocar desconexión** si se reciben **antes** de que el usuario quede autenticado.

La vulnerabilidad aparece justamente cuando el servidor **sí procesa** esos mensajes fuera de fase.

---

## 9. Explotación controlada y evidencia

### 9.1 Estado antes del ataque

Antes de la validación se verificó que:

- el contenedor estaba en ejecución;
- el puerto `2222/tcp` respondía como servicio SSH;
- una conexión SSH normal alcanzaba la fase de autenticación y solicitaba credenciales;
- no existía el artefacto que posteriormente se observaría dentro del sistema.

### 9.2 Resultado del PoC

Se ejecutó el PoC dentro del entorno controlado y se observó:

- intercambio de banner con el servicio Erlang;
- envío de `KEXINIT`;
- envío de `CHANNEL_OPEN`;
- envío de `CHANNEL_REQUEST (exec)` en fase pre-auth;
- recepción de respuesta binaria correspondiente al protocolo SSH.

La respuesta en hexadecimal no constituyó, por sí sola, una salida legible del comando enviado. Sin embargo, sí demostró que el servidor continuó procesando la interacción en vez de rechazarla de inmediato.

### 9.3 Estado durante la explotación

Durante la prueba se registró la salida del PoC y se documentó que la secuencia enviada no seguía un flujo normal de autenticación, sino que introducía mensajes de sesión antes del login.

### 9.4 Evidencia de impacto en el sistema objetivo

Para validar si la secuencia había tenido efecto real, se revisó el sistema dentro del contenedor. Se encontró que una acción enviada remotamente había producido un cambio observable en el sistema de archivos: **la creación de una carpeta**.

Ese cambio permitió afirmar que:

- sí hubo procesamiento efectivo del mensaje de ejecución;
- el efecto ocurrió **sin autenticación previa**;
- la vulnerabilidad no era solo teórica ni una anomalía de negociación.

### 9.5 Estado después del ataque

Después de la prueba, el sistema mostró un artefacto nuevo que no existía en la línea base previa. Esto se utilizó como evidencia del impacto real y controlado de la explotación.

### 9.6 Comandos y registros clave

Se recomienda incluir en anexos o como bloque de evidencia:

- escaneo de red para identificar el host activo;
- escaneo de puertos para identificar `2222/tcp`;
- conexión SSH en modo verbose para observar el flujo normal;
- salida del PoC utilizado;
- verificación del sistema de archivos antes y después de la prueba.

### 9.7 Conclusión técnica de la validación

La evidencia reunida es consistente con:

> ejecución remota de una función crítica antes de completar autenticación de usuario.

Esto valida en laboratorio la naturaleza del CVE como **RCE no autenticada**.

---

## 10. Explicación técnica de la vulnerabilidad

La vulnerabilidad consiste en que el servidor SSH de Erlang/OTP procesa mensajes pertenecientes a la capa de conexión del protocolo antes de que el cliente haya completado la fase de autenticación.

En una implementación sana, el orden esperado es:

1. transporte;
2. negociación criptográfica;
3. autenticación del usuario;
4. apertura de sesión/canales;
5. solicitudes de ejecución.

En la implementación vulnerable, ese orden se rompe porque el servidor permite que un cliente no autenticado alcance operaciones reservadas para una sesión ya autorizada.

### 10.1 Condiciones de explotabilidad

Para que la vulnerabilidad sea explotable deben darse, al menos, estas condiciones:

- servicio SSH de Erlang/OTP expuesto en red;
- versión vulnerable;
- posibilidad de conexión al puerto del servicio;
- capacidad de construir mensajes del protocolo en el orden malicioso requerido.

### 10.2 Impacto CIA

- **Confidencialidad:** alta, por posibilidad de acceder a datos o ejecutar acciones de lectura
- **Integridad:** alta, por posibilidad de alterar archivos o estado del sistema
- **Disponibilidad:** alta, por posibilidad de ejecutar acciones que afecten el servicio o el host

---

## 11. CWE asociado

El caso corresponde de forma natural a:

### **CWE-306 — Missing Authentication for Critical Function**

Justificación:

- la función crítica es la capacidad de procesar una solicitud que lleva a ejecución remota;
- esa función se vuelve accesible sin que el usuario haya completado autenticación;
- el servidor falla al imponer el requisito de identidad antes de permitir una operación privilegiada.

---

## 12. Mapeo a modelos de ataque

### 12.1 STRIDE

- **Spoofing:** el cliente malicioso se presenta inicialmente como un cliente SSH válido.
- **Tampering:** la ejecución remota permite modificar el estado del sistema, como se observó con la creación de un directorio.
- **Information Disclosure:** la misma debilidad podría usarse para obtener información sensible del sistema.
- **Denial of Service:** la capacidad de ejecutar funciones críticas sin autenticación podría emplearse para afectar la disponibilidad.
- **Elevation of Privilege:** el procesamiento de una función crítica sin identidad validada constituye un bypass del control de autorización.

### 12.2 Cyber Kill Chain

- **Reconnaissance:** descubrimiento del host, del puerto expuesto y del servicio SSH.
- **Weaponization / Preparation:** análisis o preparación de un cliente capaz de construir mensajes del protocolo de forma no estándar.
- **Delivery:** envío de la secuencia maliciosa al puerto 2222.
- **Exploitation:** procesamiento indebido de mensajes `CHANNEL_OPEN` y `exec` en fase pre-auth.
- **Actions on Objectives:** ejecución de acciones sobre el sistema remoto.

### 12.3 MITRE ATT&CK

Mapeo razonable a nivel conceptual:

- **Initial Access** — uso de un servicio expuesto
- **Execution** — ejecución remota de acciones/comandos
- **Discovery** — posible obtención de información sobre el sistema afectado
- **Impact** — alteración del estado del sistema

---


## 14. Mitigaciones

### 13.1 Actualización del software

La defensa principal es actualizar a una versión corregida de Erlang/OTP SSH.

### 13.2 Segmentación y reducción de exposición

El servicio SSH vulnerable no debería estar expuesto innecesariamente a redes amplias. La segmentación reduce la posibilidad de explotación remota.

### 13.3 Principio de mínimo privilegio

El proceso del servicio debería ejecutarse con los privilegios mínimos necesarios, para limitar el impacto de una ejecución indebida.

### 13.4 Filtrado y control de acceso

Restringir acceso al puerto del servicio mediante firewall, listas de control de acceso o exposición solo a redes de administración.

### 13.5 Monitoreo y detección

Registrar y alertar sobre:

- conexiones anómalas,
- cierres inesperados,
- secuencias inválidas del protocolo,
- actividad inusual en el servicio SSH.

### 13.6 ¿Podría corregirse?

Sí, al menos a nivel conceptual. La solución de fondo consiste en reforzar la máquina de estados del servidor SSH para impedir que mensajes del protocolo de conexión sean aceptados antes de que el usuario complete la autenticación. En otras palabras, `CHANNEL_OPEN`, `CHANNEL_REQUEST` y operaciones equivalentes deberían validarse estrictamente contra el estado de sesión y rechazarse o provocar desconexión cuando se reciben en fase pre-auth.

### 13.7 ¿Sería posible contribuir un parche?

En teoría sí, porque Erlang/OTP es software libre. Sin embargo, para esta práctica no se desarrolló ni propuso un parche concreto. Lo que sí puede afirmarse es que la estrategia correcta de corrección consiste en endurecer la validación de estado y asegurar que ninguna función crítica se invoque sin autenticación previa.

---

## 14. Reflexión personal

Este ejercicio dejó claro que entender una vulnerabilidad implica mucho más que ejecutar una prueba de concepto. El aprendizaje más importante fue distinguir el **flujo normal** de un protocolo del **punto exacto donde se rompe** su modelo de seguridad.

Antes de este laboratorio, una conexión SSH podía verse como una “caja negra” donde simplemente se pedían credenciales. Sin embargo, al separar transporte, negociación y autenticación, se hizo evidente que el verdadero hallazgo del CVE está en el momento en que el servidor permite avanzar a operaciones de sesión sin haber verificado identidad.

También resultó valioso comprobar que una explotación no siempre se manifiesta con una shell interactiva o una salida legible inmediata. A veces la evidencia consiste en un cambio observable en el sistema y en el hecho de que el servidor aceptó una secuencia que nunca debió procesar.

La diferencia entre leer sobre un ataque y ejecutarlo en laboratorio es muy clara: al reproducirlo, se entiende no solo **qué** falla, sino **por qué** falla y **cómo** se razona hasta llegar al hallazgo.

---

## 15. Conclusión

La práctica permitió recorrer el ciclo completo de una vulnerabilidad real:

- descubrimiento del servicio;
- identificación de una implementación no estándar;
- observación del flujo legítimo del protocolo;
- formulación de una hipótesis de falla;
- análisis conceptual del PoC;
- validación de impacto en un entorno controlado.

La evidencia obtenida muestra que el servidor SSH vulnerable de Erlang/OTP puede procesar mensajes de sesión y ejecución antes de la autenticación del usuario, lo que habilita una función crítica sin validación previa. Eso convierte al caso en un ejemplo claro de **RCE no autenticada** y de una debilidad correctamente entendida a través de **CWE-306**.

---

## 16. Referencias

- Advisory oficial de Erlang/OTP para CVE-2025-32433
- Escenario de Vulhub: `erlang/CVE-2025-32433`
- RFC 4254 — The Secure Shell (SSH) Connection Protocol
- Traza de conexión SSH observada durante el laboratorio
- Código PoC analizado (`exploit.py`)
