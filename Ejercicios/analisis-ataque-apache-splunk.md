# Detección de ataques web con Apache y Splunk

> Simular ataques al Apache y mediante SPL hacer reportes y crear dashboard.  

---

## Fase 1 — Atacar Apache

### 1.1 Directory Bruteforcing

Es una forma que tienen los atacantes de encontrar directorios en nuestra web. En vez de instalar herramientas dedicadas a ello, me voy a valer de un simple script en bash para hacer llamadas a rutas inventadas como **/admin**, **/login** y **/admin-login**.

**Script bash:**

```bash
for i in {1..10}; do curl -s http://<IP-APACHE>/login; done
for i in {1..8}; do curl -s http://<IP-APACHE>/admin; done
for i in {1..8}; do curl -s http://<IP-APACHE>/admin-login; done
```

### 1.2 Brute Forcing

Los atacantes pueden intentar entrar en zonas protegidas con autenticación de nuestra web mediante fuerza bruta, es decir, con una combinación de usuario y contraseñas, generalmente utilizando diccionarios. Esto se puede mitigar aplicando **rate limiting**, pero este tutorial no contempla esto.

Al igual que con el ataque anterior, se podria hacer un script, pero en este caso no me interesa la cantidad de logs sino ver diferentes combinaciones de logs con usuarios y contraseñas diferentes, por lo que he metido a mano usuarios y contraseñas.


## Fase 2 — Análisis en Splunk (SPL)

Una vez generados los ataques, abriremos la barra de búsquedas de Splunk (*Search & Reporting*) para analizar los logs e identificar los patrones que delatan al atacante. En esta fase buscamos entender el comportamiento anómalo antes de automatizarlo.

### 2.1 Identificar el Directory Bruteforcing (Peticiones 404)

* **Por qué:** Un usuario común puede equivocarse una o dos veces al escribir una URL, pero un atacante automatizado generará decenas de errores de página no encontrada en pocos segundos.
* **Cómo:** Filtramos por el código de estado HTTP `404` y agrupamos los resultados por la dirección IP de origen para ver quién está escaneando.

```spl
index=apache-logs code=404 
| stats count(code) by host
```

**Explicación**
- index=apache-logs — Le decimos donde buscar, en este caso, en el indice que hemos creado para los logs de la web.
- code=404 — Filtramos para que solo aparezcan logs en los que haya habiado un code 404 (Not Found).
- stats — Sirve para calcular estadísticas y resumir datos, transformando una lista gigante de logs en una tabla limpia.
- count(code) by host — Contamos por Ip cuantas veces ha salido el code 404.

### 2.2 Identificar el Bruteforcing al directorio protegido

* **Por qué:** Un usuario puede poner una o dos veces la contraseña mal, pero un atacante prueba una gran combinación de usuarios y contraseñas de forma automática.
* **Cómo:** Filtramos por el directorio protegido (en mi caso /secure) 

```spl
index="apache-logs" path="/secure/*" code="401"
| stats count(path) by host
```
**Explicación**
- index=apache-logs — Le decimos donde buscar, en este caso, en el indice que hemos creado para los logs de la web.
- path="/secure/*" — Monitorizamos la ruta que sabemos que esta protegida para saber quienes entran ahí.
- code="401" - Es el codigo cuando el servidor web rechaza la solicitud por no tener las credenciales de autenticación correctas.
- stats — Sirve para calcular estadísticas y resumir datos, transformando una lista gigante de logs en una tabla limpia.
- count(path) by host — Contamos por ruta cuantas veces una IP le ha dado el error 401.

## Fase 3 — Dashboard

No tenemos que ir a search y hacer la búsqueda siempre que queramos ver esta información, si queremos acceder de forma frecuente a esta información, podemos crear un dashboard y tener estas searchs de forma gráfica y visual en nuestra home de Splunk.

### 3.1 Crear dashboard

`Menu lateral izquierdo → Dashboard → Create New Dashboard`

**Datos del dashboard**
- Dashboard Title — Monitorización Apache — Dashboard SOC
- Description — Dashboard para detectar patrones de ataque en Apache: escaneo de directorios, fuerza bruta y actividad sospechosa por IP.
- Permissions - Private
- Dashboard type — Dashboard Studio (Grid)

### 3.2 Añadir componentes al Dashboard

Una vez creado el lienzo en Dashboard Studio, añadiremos los tres componentes visuales clave para monitorizar el servidor Apache.

**Componente 1:** Total Peticiones últimas 24h (Single Value)
- Por qué: Permite al analista del SOC conocer el volumen global de tráfico de un vistazo y detectar anomalías o picos de tráfico inusuales de forma inmediata.
- Cómo: Añade un componente de tipo Single Value (Valor único).

```spl
index="apache-logs"
| stats count
```

**Explicación**
- index="apache-logs" — Busca en el índice del servidor web.
- stats count — Cuenta el número total de eventos recolectados en el rango de tiempo seleccionado.

**Componente 2:** Escaneo de Rutas (Bar Chart)
- Por qué: Ayuda a identificar qué directorios o archivos inexistentes están intentando descubrir los atacantes mediante escaneos automatizados.
- Cómo: Añade un gráfico de tipo Bar Chart (Gráfico de barras horizontales).

```spl
index="apache-logs" code="404"
| stats count(path) by path
```

**Explicación**
- code="404" — Filtra exclusivamente los accesos que devolvieron un error de "Página no encontrada".
- stats count(path) by path — Cuenta y agrupa cuántas veces se ha intentado acceder a cada una de las rutas inexistentes.

**Componente 3:** IPs haciendo Fuerza Bruta a /secure (Column Chart)
- Por qué: Permite poner cara al atacante identificando qué direcciones de red están generando una gran cantidad de denegaciones de acceso (401) en zonas protegidas.
- Cómo: Añade un gráfico de tipo Column Chart (Gráfico de columnas verticales).

```spl
index="apache-logs" path="*/secure*" code="401"
| stats count(path) by host
```

**Explicación**
- path="*/secure*" — Monitoriza únicamente los accesos dirigidos al directorio o recursos protegidos.
- code="401" — Filtra los eventos donde el servidor web rechazó las credenciales por ser incorrectas.
- stats count(path) by host — Agrupa y cuenta los intentos fallidos según el host de origen para identificar el foco de la fuerza bruta.

