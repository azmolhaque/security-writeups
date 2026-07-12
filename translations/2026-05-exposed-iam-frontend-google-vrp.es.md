# Anatomía de un frontend de IAM expuesto

**Un bypass total de autenticación en un activo de una adquisición de Google — y un relato preciso de por qué se corrigió en nueve días, y por qué recompensarlo con $0 fue lo correcto.**

**🌐 Read this in your language:** [English](../2026-05-exposed-iam-frontend-google-vrp.md) · **Español** · [Français](./2026-05-exposed-iam-frontend-google-vrp.fr.md) · [Deutsch](./2026-05-exposed-iam-frontend-google-vrp.de.md) · [العربية](./2026-05-exposed-iam-frontend-google-vrp.ar.md) · [हिन्दी](./2026-05-exposed-iam-frontend-google-vrp.hi.md) · [বাংলা](./2026-05-exposed-iam-frontend-google-vrp.bn.md) · [简体中文](./2026-05-exposed-iam-frontend-google-vrp.zh.md) · [日本語](./2026-05-exposed-iam-frontend-google-vrp.ja.md)

![Program](https://img.shields.io/badge/Program-Google_VRP-4285F4)
![Status](https://img.shields.io/badge/Status-Fixed-success)
![Triage](https://img.shields.io/badge/Triage-P2_%2F_S2-orange)
![Reward](https://img.shields.io/badge/Reward-Credit_%2F_Honorable_Mention-lightgrey)
![Disclosure](https://img.shields.io/badge/Disclosure-Coordinated-blue)

> **CWE:** [CWE-287](https://cwe.mitre.org/data/definitions/287.html) (Autenticación indebida) · [CWE-1188](https://cwe.mitre.org/data/definitions/1188.html) (Uso de credenciales por defecto) · [CWE-319](https://cwe.mitre.org/data/definitions/319.html) (Transmisión en texto claro)
> **Tipo de activo:** adquisición de Google (Photomath), Nivel 1 según `external_domains_acquisitions.asciipb`

**TL;DR** — Una interfaz administrativa de IAM estaba expuesta en la internet pública, en un subdominio de una adquisición de Google. El inicio de sesión aceptaba credenciales por defecto, luego aceptaba *cualquier* contraseña, y la API detrás respondía a peticiones no autenticadas — un fallo completo de la capa de autenticación. El equipo de producto de Google lo clasificó como P2/S2 y lo desmanteló nueve días después de aceptar el informe. El panel de recompensas de VRP, por separado, otorgó crédito y nada de dinero. Este análisis desglosa la exposición y luego hace lo más difícil y útil: explica, a nivel de mecanismo, **por qué ambas decisiones son correctas y no están en conflicto** — y qué evidencia lo habría movido por encima del umbral de recompensa. Calibrar esa distancia es la verdadera habilidad.

---

## Contenido
- [Por qué escribo esto](#por-qué-escribo-esto)
- [1. Descubrimiento — y cómo encontrar esta clase a propósito](#1-descubrimiento--y-cómo-encontrar-esta-clase-a-propósito)
- [2. La debilidad de autenticación](#2-la-debilidad-de-autenticación)
- [3. La capa de API detrás](#3-la-capa-de-api-detrás)
- [4. Remediación](#4-remediación)
- [5. Causa raíz, con precisión: servido desde el edge ≠ operado por Google](#5-causa-raíz-con-precisión-servido-desde-el-edge--operado-por-google)
- [6. Por qué correctamente no se recompensó — y qué lo habría cambiado](#6-por-qué-correctamente-no-se-recompensó--y-qué-lo-habría-cambiado)
- [7. Si yo defendiera esto](#7-si-yo-defendiera-esto)
- [8. Lecciones que me llevo](#8-lecciones-que-me-llevo)
- [Qué demuestra este hallazgo](#qué-demuestra-este-hallazgo)
- [Cronología](#cronología)
- [Referencias](#referencias)

## Por qué escribo esto

La mayoría de los writeups de bug bounty se detienen en «encontré X, aquí está el pago». La historia más útil suele ser la distancia entre lo grave que *parece* un hallazgo y lo grave que *es* — porque juzgar esa distancia correctamente es el verdadero trabajo, a ambos lados de una cola de triaje y en cualquier equipo de seguridad.

Este hallazgo parecía crítico en la superficie: una interfaz administrativa de gestión de identidades y accesos (IAM) no autenticada, expuesta en la internet pública, en un dominio perteneciente a una adquisición de Google. El equipo de producto de Google coincidió en que valía la pena corregirlo (P2/S2) y lo remedió rápido. El panel de recompensas de VRP decidió, por separado, que no merecía una recompensa monetaria — y tras trabajar la evidencia creo que su razonamiento fue exactamente correcto.

Así que esto hace dos cosas: diseca la anatomía técnica y explica — con honestidad, y al nivel de *por qué los sistemas se comportaron así* — por qué «corregir rápido» y «no pagar nada» fueron ambas decisiones correctas. La segunda mitad es la parte que querría que leyera un responsable de contratación.

## 1. Descubrimiento — y cómo encontrar esta clase a propósito

El activo era `rip.photomath.net`, un subdominio ligado a una adquisición de Google (Photomath). No apareció por suerte; surgió de un método repetible para el rincón de mayor rendimiento de un alcance grande y con muchas adquisiciones — **infraestructura obsoleta y sin migrar de empresas adquiridas recientemente**:

1. **Enumera el alcance de adquisiciones, no solo el producto estrella.** El propio archivo de alcance de Google, `external_domains_acquisitions.asciipb`, clasifica los dominios de adquisiciones por nivel. `*.photomath.net` está ahí, en Nivel 1. Los subdominios de adquisiciones son donde vive la deuda de integración.
2. **Expansión pasiva de subdominios** (registros de transparencia de certificados + DNS histórico) revela hosts como `rip.` que nunca aparecen en la navegación del propio producto.
3. **Resuelve y toma huellas de cada host**, luego filtra con dureza por las señales de infraestructura sin migrar en lugar de producción endurecida:

| Observación | Cómo se determinó | Por qué importa |
|---|---|---|
| Servía una UI de administración (`GestionUsersRolesFrontend`) | Carga directa en el navegador | Una superficie privilegiada de gestión de usuarios/roles |
| **Solo HTTP plano; TLS falló en `:443`** | `unexpected eof` en el handshake HTTPS | Un host fuera de la política estándar de terminación TLS del adquirente — una señal de migración (CWE-319) |
| **Servido por infraestructura de Google** | Cabecera `Via: 1.1 google`; IP de GCP en la resolución | Enruta a través del edge de Google — pero, como muestra la Sección 5, servido desde el edge **no** es lo mismo que operado por Google |
| Cadenas de administración no en inglés (`Bienvenue administrateur`), texto de marcador (`users works!`) | Inspección de la UI | Compilación de la empresa de origen, probablemente dev/sandbox, arrastrada sin cambios en la adquisición |

El patrón que debería hacer que un cazador experimentado se detenga a mirar: **un panel de administración de usuarios/roles, accesible por HTTP plano, sin SSO / proxy con reconocimiento de identidad delante, en un dominio de nivel de adquisición.** Cada uno de esos adjetivos es un síntoma de infraestructura que se heredó y nunca se integró en el perímetro de seguridad del adquirente.

## 2. La debilidad de autenticación

La pantalla de inicio de sesión (`Bienvenue administrateur`) aceptaba el par por defecto de manual:

```
email:    admin@photomath.net
password: admin
```

Eso por sí solo es CWE-1188 (credenciales por defecto). Pero al indagar más se reveló algo más fundamental: el portal aceptaba *cualquier* cadena de contraseña arbitraria para la cuenta de administrador. Eso lo mueve de «credenciales débiles» a **CWE-287 (autenticación rota)** — el frontend no realizaba ninguna validación real de credenciales contra un backend. El inicio de sesión era decorativo.

Lo confirmé deliberadamente (inicio de sesión con caracteres aleatorios) en lugar de asumirlo, porque «las credenciales por defecto funcionan» y «la autenticación está completamente ausente» son severidades distintas y quería reclamar solo la que podía demostrar.

**🎥 Prueba — un inicio de sesión exitoso usando una cadena de contraseña aleatoria, que llega al panel de Roles autenticado (grabación de pantalla sin editar):**

https://github.com/user-attachments/assets/dca3d51d-7b64-480f-ab64-b3c625b54832

## 3. La capa de API detrás

Iniciar sesión con una contraseña basura llevaba al panel administrativo completo — el resultado visible de la autenticación rota descrita arriba:

![Panel administrativo de IAM alcanzado sin credenciales válidas](../images/rip-photomath-dashboard.png)

*El panel de IAM (`/dashboard`) alcanzado sin credenciales válidas — «Welcome, administrator», la barra lateral de gestión de Roles/Usuarios, el control de escritura «New role» y sin registros reales (un sandbox vacío).*

Un bypass a nivel de UI es un hallazgo débil si el backend sigue aplicando la autorización de forma independiente. Así que la siguiente pregunta — la que separa una captura de pantalla de un hallazgo real — era: **¿comprueba la API detrás de esta UI la autenticación por su cuenta?** No lo hacía.

```http
GET /api/roles HTTP/1.1
Host: rip.photomath.net
# no auth headers, no session cookie

HTTP/1.1 200 OK
[]
```

![GET /api/roles no autenticado devuelve [] con un 200](../images/rip-api-roles-unauth.png)

Un GET no autenticado devolvía un array JSON vacío, no un `401`/`403`. Un `OPTIONS` no autenticado anunciaba el conjunto completo de métodos con capacidad de escritura:

```console
$ curl -i -s -k -X OPTIONS "http://rip.photomath.net/api/roles"
HTTP/1.1 200 OK
Allow: POST,GET,HEAD,OPTIONS
Via: 1.1 google
```

![OPTIONS devuelve Allow: POST,GET,HEAD,OPTIONS y Via: 1.1 google](../images/rip-api-options-allow.png)

`Allow: POST,GET,HEAD,OPTIONS` muestra que el endpoint acepta escrituras (`POST`) sin autenticación. Un `404` en una ruta no mapeada devolvía la página de error estándar de Spring Boot:

![Página de error Whitelabel de Spring Boot](../images/rip-spring-boot-whitelabel.png)

Así que dos controles independientes — la autenticación del frontend y la autorización del backend — estaban ambos ausentes en la misma superficie. Esa es la parte arquitectónicamente interesante, y es por lo que el hallazgo era *completo*: no me detuve en «el inicio de sesión es falso», demostré que la propia capa de datos estaba abierta.

> **Reproducción (resumen de solo lectura).**
> 1. Resuelve `rip.photomath.net` y cárgalo por HTTP plano (`:443` falla el handshake TLS) — se renderiza la UI de administración (`GestionUsersRolesFrontend`).
> 2. En el inicio de sesión `Bienvenue administrateur`, envía `admin@photomath.net` con **cualquier** cadena de contraseña → llega a `/dashboard`.
> 3. `GET /api/roles` **sin** cookie ni cabecera de autenticación → `200 OK`, cuerpo `[]` (no `401`/`403`).
> 4. `OPTIONS /api/roles` → `Allow: POST,GET,HEAD,OPTIONS` — métodos de escritura anunciados sin autenticación.
>
> No se realizaron escrituras ni se crearon o modificaron registros; los pasos 3–4 confirman el fallo de control sin ejercitar la ruta de escritura.

**Alcance de las pruebas.** Confirmé la accesibilidad de lectura y el conjunto de métodos anunciado. **No** realicé escrituras, ni creé roles, ni modifiqué ningún estado. Demostrar la accesibilidad fue suficiente para probar el fallo de control, y detenerse ahí es lo que exigen las expectativas de puerto seguro (safe harbor). Afirmar que la ruta de escritura *funcionaba* sin ejercitarla habría sido exagerar; señalar que estaba *anunciada* es un hecho.

## 4. Remediación

La gestión de Google fue rápida y limpia:

- **Aceptado en ~24 horas** y asignado al equipo de producto responsable.
- **Marcado como Corregido nueve días después de la aceptación** — el endpoint fue desmantelado y el nombre de host empezó a devolver `NXDOMAIN`. Reverifiqué de forma independiente el `NXDOMAIN` y lo reporté de vuelta.
- Clasificado internamente como **P2 / S2**.

![Google Issue Tracker status: Accepted (comment #5) and Marked as fixed (comment #6)](../images/rip-tracker-accepted-fixed.png)

*Las propias actualizaciones del tracker — aceptado al día siguiente del reporte, marcado como corregido nueve días después. Mantengo la redacción exacta de Google y omito las direcciones del remitente. Y para calibrarlo con honestidad: el festivo «Nice catch!» es la **plantilla de aceptación estándar** del programa, no un elogio personal; lo que de verdad pesa es el triaje P2/S2 y la corrección confirmada.*

![rip.photomath.net ahora devuelve DNS_PROBE_FINISHED_NXDOMAIN](../images/rip-nxdomain-fixed.png)

Nota *cómo* se corrigió: no un parche de código, no un cambio en el middleware de autenticación — se retiró el registro y el host dejó de resolverse. Ese detalle es toda la clave de la decisión de recompensa, y es el tema de la siguiente sección.

## 5. Causa raíz, con precisión: servido desde el edge ≠ operado por Google

Esta es la parte que la mayoría de los writeups omite, y es la parte que realmente lo explica todo.

`rip.photomath.net` resolvía a una dirección detrás del edge de Google y devolvía `Via: 1.1 google`. Es tentador — y al principio me incliné por ello — leer «el tráfico fluye a través del edge de Google» como «esto es un sistema de producción operado por Google». **No es lo mismo**, y la diferencia es todo el hallazgo:

- Un **registro DNS heredado de la adquisición** seguía apuntando a un despliegue que Photomath creó antes de la adquisición y que nunca se desmanteló ni migró. Era un **frontend huérfano**, no un servicio integrado de Google.
- Como nunca se integró en el perímetro del adquirente, quedaba **fuera del proxy con reconocimiento de identidad** (sin puerta BeyondCorp/SSO) y **fuera de la política TLS estándar del edge** (HTTP plano, `:443` fallido). Esos no eran bugs separados — son todos el mismo síntoma: *esta máquina nunca se metió dentro de la valla.*
- La aplicación era casi con certeza una **compilación dev/sandbox** — cadenas de UI en francés, marcador `users works!` y un `/api/roles` vacío. No había usuarios, credenciales ni registros reales detrás.
- Que la corrección fuera **«retirar el registro DNS»** en lugar de **«parchear la aplicación»** confirma la causa raíz: no había ninguna aplicación propia y operada que parchear. La vulnerabilidad vivía en un **artefacto colgante que simplemente resolvía a través de la infraestructura de Google.**

Esta es la forma habitual del **riesgo de integración de adquisiciones**: cuando se adquiere una empresa, sus zonas DNS, proyectos de nube y despliegues medio olvidados se migran en un calendario, y los registros obsoletos sobreviven en el intervalo. Vale absolutamente la pena encontrarlos y corregirlos — pero su causa raíz es *inventario e higiene*, no un defecto en el código de la aplicación del adquirente.

## 6. Por qué correctamente no se recompensó — y qué lo habría cambiado

Es fácil escribir «acceso de administrador no autenticado + autenticación rota + API de escritura expuesta en un activo de Google de Nivel 1» y llamarlo crítico. Al principio lo enmarqué con fuerza. Pero **la severidad es el impacto realizado en sistemas y datos que realmente importan**, y la motivación del panel de recompensas fue precisa (citando su decisión):

> *«…no estaba ubicado dentro de una aplicación de Google, sino que era el resultado de registros DNS obsoletos. Como la vulnerabilidad no afectaba a un sistema bajo nuestro control operativo directo, no cumple los requisitos para una recompensa monetaria…»*

Tres cosas van en contra de la lectura dramática, y las tres se siguen directamente de la Sección 5:

- **(a) No es producción.** Texto de marcador y un `/api/roles` vacío — un panel de administración expuesto sobre un sandbox vacío es un problema genuino de higiene, no una filtración de datos.
- **(b) Higiene, no vulnerabilidad de aplicación.** La corrección fue desmantelar un endpoint colgante. VRP recompensa defectos en sistemas que Google opera, no artefactos huérfanos que casualmente resuelven a través de su edge.
- **(c) Mi afirmación de impacto más fuerte era especulativa.** En mi apelación me apoyé en un **vector de marca/phishing** — que un atacante podría recolectar credenciales del personal que reconociera la interfaz de confianza. Eso es un impacto *hipotético secundario*, no un daño demostrado, y eran adjetivos, no un hecho nuevo. El panel reconsideró y mantuvo la decisión original, correctamente.

### La distinción que lo decidió: la vía de severidad ≠ la vía de recompensa

Un **«Corregido»** P2/S2 es una señal de *ingeniería* — el equipo juzgó que valía la pena limpiarlo. **No** es una señal de *recompensa*. El equipo de producto optimiza para «¿debería limpiarse esto?»; el panel de recompensas optimiza para «¿expuso esto un riesgo real en un sistema que operamos?». Son preguntas distintas con respuestas distintas, y confundirlas es un error temprano común — uno que cometí en la apelación.

### Qué *habría* superado el umbral de recompensa (el contrafáctico útil)

Saber con precisión qué faltaba es más valioso que el propio hallazgo. Cualquiera **una** de estas probablemente habría cambiado el resultado — y cada una es una prueba concreta siguiente, no un deseo:

| Si hubiera demostrado… | Por qué supera el umbral |
|---|---|
| Que `/api/roles` devolvía **registros de usuarios reales** (PII real, no `[]`) | Exposición de datos de la que Google es responsable — impacto realizado, no hipotético |
| Que la **ruta de escritura** (`POST /api/roles`) creaba un rol que **se federaba en un sistema autenticado y operado por Google** (SSO/sesión compartida) | Escalada de privilegios desde un cascarón hacia producción — pivote demostrado |
| Que el host **compartía un dominio de cookie** (`.photomath.net`) o una lista de `redirect_uri` de OAuth con una app de producción *viva y autenticada* | Robo de sesión/token contra usuarios reales — la clásica cadena de subdominio colgante |

Ninguna aplicaba aquí: sandbox vacío, sin superficie de autenticación compartida, sin pivote alcanzable a producción. Reconocer eso *antes* de escalar la apelación es exactamente la calibración que la segunda decisión estaba evaluando — y donde podría haber salvado mi propia credibilidad.

## 7. Si yo defendiera esto

El hallazgo es más útil para un equipo de seguridad como lección de detección y prevención que como anécdota de guerra. Si yo fuera dueño de este perímetro:

**Detectar**
- **Monitorización continua de transparencia de certificados** para cada dominio de adquisición (`*.photomath.net` y hermanos) — tanto los hosts nuevos como los olvidados aparecen en CT.
- Un **barrido programado de resolución y clasificación** de cada subdominio en el archivo de alcance de nivel de adquisición, marcando: UIs de administración por HTTP plano, hosts que *no* están detrás del proxy con reconocimiento de identidad, y cualquier `2xx` en un `/api/*` no autenticado.
- **Reconciliación de DNS colgante**: compara la zona DNS viva con el inventario de despliegues *operados intencionadamente*; cualquier cosa que resuelva sin dueño es un hallazgo por definición.

**Prevenir**
- Pon **todos** los activos de adquisición detrás del proxy con reconocimiento de identidad (estilo BeyondCorp) **antes** del corte de DNS, para que una máquina sin migrar falle cerrada en vez de abierta.
- Impón **solo HTTPS en el edge** y niega el HTTP plano — el `:443` fallido aquí era una señal de alerta temprana gratuita que se ignoró.
- Trata el **off-boarding de adquisiciones** como un elemento de checklist: desmantela la infraestructura y los registros de la empresa de origen en una fecha límite, no «con el tiempo».

El único control que habría prevenido toda la exposición es el proxy con reconocimiento de identidad: con él, un panel de administración huérfano es inalcanzable sin importar lo rota que esté su propia autenticación.

## 8. Lecciones que me llevo

- **Demuestra el impacto; no lo infieras de las etiquetas.** «Nivel 1» describe la sensibilidad *potencial* de un dominio, no la severidad de cualquier hallazgo concreto en él. Lo que cuenta son los datos realmente en riesgo.
- **Distingue higiene de vulnerabilidad — antes de escribir.** DNS colgante, exposición de sandbox y endpoints obsoletos son con frecuencia válidos-pero-solo-crédito. Fijar esa expectativa por adelantado mantiene el informe honesto y la apelación disciplinada.
- **El razonamiento en dos capas supera al de una capa.** Comprobar si el backend aplicaba la autenticación de forma independiente — no solo el formulario de inicio de sesión — es lo que hizo esto completo. Pregunta siempre qué hace el siguiente control por debajo.
- **Las apelaciones necesitan un hecho nuevo, no adjetivos más fuertes.** Si no puedo añadir evidencia nueva y concreta, escalar el lenguaje solo erosiona la credibilidad con los triadores. Reformular el impacto con palabras más fuertes es exactamente por lo que mi apelación no cambió (ni debía cambiar) el resultado.
- **Servido desde el edge no es operado por.** Por dónde se enruta una petición no dice nada sobre quién es dueño del riesgo. Esa única distinción es la diferencia entre un hallazgo recompensable y una nota de higiene.
- **La calibración es la habilidad.** Cualquiera puede encontrar algo que parece alarmante. El movimiento profesional es afirmar con precisión cuánto importa — incluyendo, y especialmente, cuando la respuesta honesta es «menos de lo que parecía al principio».

## Qué demuestra este hallazgo

Leído como una muestra de trabajo, las señales útiles aquí no son el bug en sí — son cómo se gestionó:

- **Reconocimiento dirigido, no a ciegas** — surgió a través del archivo de alcance de nivel de adquisición más los registros CT, un método repetible para encontrar infraestructura sin migrar.
- **Análisis de control en dos capas** — comprobé si el backend aplicaba la autorización de forma independiente del formulario de inicio de sesión, no solo la UI.
- **Pruebas de impacto mínimo** — confirmé la accesibilidad y los métodos de escritura anunciados sin realizar ni una sola escritura ni tocar ningún dato.
- **Calibración de severidad bajo presión** — argumenté el hallazgo con precisión, admití dónde mi apelación era especulativa, y acepté el resultado de solo crédito una vez que la evidencia fue clara.
- **Divulgación coordinada** — reporté por el canal del proveedor, esperé la remediación, reverifiqué la corrección (`NXDOMAIN`) y publiqué solo después.

## Cronología

| Fecha | Día | Evento |
|---|---|---|
| 2026-05-05 | 0 | Reportado a Google VRP; acuse de recibo automático |
| 2026-05-06 | +1 | **Aceptado**; error asignado al equipo de producto |
| 2026-05-15 | +10 | **Marcado como Corregido** — endpoint desmantelado, `NXDOMAIN`; reverifiqué y confirmé |
| 2026-05-29 | +24 | Panel de recompensas: **no alcanza el umbral** → crédito / Honorable Mention |
| 2026-05-29 | +24 | Apelé para reconsideración |
| 2026-06-02 | +28 | Apelación revisada y **mantenida** — solo crédito confirmado (motivo: DNS obsoleto) |

## Referencias

- **MITRE CWE** — las debilidades a las que este hallazgo se asocia: [CWE-287: Autenticación indebida](https://cwe.mitre.org/data/definitions/287.html), [CWE-1188: Uso de credenciales por defecto](https://cwe.mitre.org/data/definitions/1188.html), [CWE-319: Transmisión en texto claro de información sensible](https://cwe.mitre.org/data/definitions/319.html).
- **[Google Bug Hunters (VRP)](https://bughunters.google.com/)** — el programa por el que se reportó; sus reglas definen qué cumple los requisitos para una recompensa monetaria frente a crédito, y sustentan la decisión de recompensa analizada en la Sección 6.
- **DNS colgante / apropiación de subdominios (subdomain takeover)** — la clase de riesgo a la que pertenece: un registro DNS que sobrevive al recurso al que apunta. Los controles de detección y prevención de la Sección 7 son la contraparte del defensor.

---

*Reportado a través del programa Google Bug Hunters (Issue 509594209). El endpoint afectado fue remediado y desmantelado (`NXDOMAIN`) por Google antes de la publicación. No se accedió ni se modificó ningún dato más allá de lo estrictamente necesario para confirmar la exposición. Este writeup refleja mi propio análisis y no está afiliado ni respaldado por Google.*
