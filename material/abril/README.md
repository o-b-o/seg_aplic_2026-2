# Material cursado durante el mes de abril

## Unidad 3: Tipos de ataque

1. `2026.04.07`

   - Cruce de límites de confianza: Ejecución de JavaScript del lado del
     cliente ⇒ XSS (_Cross-Site Scripting_), CSRF (Cross-Site Request
     Forgery)
     - **XSS**: Inyección de _Javascript ejecutable_. Típicamente,
       contenido _a desplegar_ en el cliente Web, no sanitizado. _Vector_:
       Hacer que el cliente interprete contenido provisto por el atacante
       como _inicio de un script_ (p.ej. etiqueta `<script>` o un atributo
       `onerror` de elementos como `<img>`).
     - **CSRF**: Engaña a un cliente (típicamente Web, pero no es
       indispensable) para _realizar una petición cuyo destino controla_ el
       atacante. Normalmente, son peticiones `GET` (no `POST`). CSRF no
       permite mucha interactividad (sólo lanzar solicitudes específicas
       _con la autenticación_ del cliente.
     - _Mitigaciones_:
       - XSS → Codificación de todo el contenido proveniente de fuera del
         sistema _evitando caracteres con significado interpretable_ (`< >
         & # ' "`); [Content Security Policy
         (CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)
         desde el servidor HTTP
       - CSRF → _tokens_ ocultos que varíen _y sean validados_ por cada
         consulta (no sirve por sesión o por formulario: Por cada
         invocación). Puede enviarse como parámetro o como cookie (o como
         ambos). ¡Ojo! Si hay vulnerabilidad XSS, ésta puede _robar el
         token anti-CSRF_.
     - Espacio para jugar a romper sitios: [XSS
       Game](https://xss-game.appspot.com/). Y... hay que reconocer que no
       siempre es fácil hacerlo, así que si de a tiro no podemos, no está
       prohibido [hacer trampa aprendiendo cómo lo hizo alguien
       más](https://jh123x.com/blog/2022/google-xss-game/).
	   
	   Menos _lúdico_, pero también interesante →
       [XSSTrain](https://xsstrain.com/)

2. `2026.04.09`

	- Revisión de trabajo entregado: Para esta fecha teníamos la entrega de
      la [Actividad práctica: ¡Intentemos un ataque! (a uno
      mismo)](../../entregas/intentemos_ataque/README.md). Revisamos y
      comentamos el ejemplo desarrollado [por su compañero José
      Lili](../../entregas/intentemos_ataque/LiliBeltranJoseEmiliano/ReporteEjecutivo.md).

3. `2026.04.13`

   - Codificación de datos (particularmente, de texto). Puntos de partida:
     - [The Absolute Minimum Every Software Developer Absolutely,
       Positively Must Know About Unicode and Character Sets (No
       Excuses!)](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)
       (Joel Spolsky, 2003)
     - [The Absolute Minimum Every Software Developer Must Know About
       Unicode in 2023 (Still No
       Excuses!)](https://tonsky.me/blog/unicode/) (Niki Tonsky, 2023)
     - [A Programmer’s Introduction to
       Unicode](https://www.reedbeta.com/blog/programmers-intro-to-unicode/)
       (Nathan Reed, 2017)
   - Cada decodificación _debe ser vista_ como una transformación → un
     _punto de costura_. Debilidades de esperar una representación y
     recibir otra...
   - Normalización de datos antes/después de validación
   - _Overlong sequences_ → Un mismo _codepoint_ Unicode puede ser
     representado por distintas secuencias de bytes, utilizando “overlong
     sequences” → [From Unicode to Exploit: The Security Risks of Overlong
     UTF-8
     Encodings](https://herolab.usd.de/the-security-risks-of-overlong-utf-8-encodings/)
     (Dominique Dittert, 2024)
     - El caracter `/` (ASCII 47, hex 0x2F) puede ser representado con el
       byte `0x2F`, pero también con `0xC0 0xAF`
     - Un filtro que bloquea `..` para prevenir _directory traversals_ va a
       permitir `%C0%AE%C0%AE%C0%AF` (_overlong encoding_ de
       `../`). (CVE-2000-0884 IIS 4.0/5.0; viola RFC 3629)
   - Clases de caracteres equivalentes → Al normalizar caracteres
     _Fullwidth_ U+FF3C (`＼`) se convierte en U+5C (`\`) (CVE 2025-52488)
   - Reordenamiento visual con marcadores `RTL`/`LTR` (U+200F RTL Mark,
     U+202E RTL override, U+202B RTL embedding, U+2067 RTL isolate; U+200E
     LTR mark, U+202A LTR embedding, U+202D LTR override, U+2066 LTR
     isolate)
   - Construcción de cadenas en el RDBMS con codificación numérica para
     lograr XSS → `(...) UNION SELECT
     CHAR(106,97,118,97,115,99,114,105,112,116)`
