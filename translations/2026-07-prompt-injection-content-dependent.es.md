# El mismo modelo, 4,6× la exposición

**Una medición de la resistencia a la inyección de prompts en un pequeño LLM local — y por qué una única "puntuación de resistencia" induce a error.**

**🌐 Read this in your language:** [English](../2026-07-prompt-injection-content-dependent.md) · **Español** · [Français](./2026-07-prompt-injection-content-dependent.fr.md) · [Deutsch](./2026-07-prompt-injection-content-dependent.de.md) · [العربية](./2026-07-prompt-injection-content-dependent.ar.md) · [हिन्दी](./2026-07-prompt-injection-content-dependent.hi.md) · [বাংলা](./2026-07-prompt-injection-content-dependent.bn.md) · [简体中文](./2026-07-prompt-injection-content-dependent.zh.md) · [日本語](./2026-07-prompt-injection-content-dependent.ja.md)

![Domain](https://img.shields.io/badge/Domain-AI%2FLLM_Security-8A2BE2)
![Method](https://img.shields.io/badge/Method-garak_·_256_trials-blue)
![Finding](https://img.shields.io/badge/Finding-Content--dependent-orange)
![Model](https://img.shields.io/badge/Model-Llama_3.2_(3B)-informational)
![Scope](https://img.shields.io/badge/Scope-Own_model_·_authorized-success)

> **CWE:** [CWE-1427](https://cwe.mitre.org/data/definitions/1427.html) (Neutralización indebida de la entrada usada en prompts de LLM) · **OWASP LLM Top 10:** [LLM01 — Inyección de Prompts](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
> **Objetivo de prueba:** Llama 3.2 (3B) vía Ollama, autoalojado en una Raspberry Pi 5 — uso autorizado, modelo propio.

**Resumen** — Se ejecutaron dos ataques distintos de inyección de prompts contra el *mismo* modelo local pequeño, 256 intentos cada uno. El modelo fue secuestrado para emitir una cadena de "odio a los humanos" el **46,9%** de las veces, pero una cadena violenta de "matar humanos" solo el **10,2%** — una diferencia de **4,6× en el éxito del ataque usando la misma técnica, cambiando solo el objetivo.** Los intervalos de confianza no se solapan, así que es un efecto real, no ruido. La lección práctica: la resistencia de un modelo a la inyección de prompts no es un único número — varía drásticamente según *qué* intenta extraer el atacante, por lo que un solo resultado de benchmark puede desviarse por múltiplos de tu amenaza real.

---

## Contenido
- [Por qué este artículo](#por-qué-este-artículo)
- [1. Método](#1-método)
- [2. Resultados](#2-resultados)
- [3. El hallazgo: la resistencia depende del contenido](#3-el-hallazgo-la-resistencia-depende-del-contenido)
- [4. Por qué ocurre](#4-por-qué-ocurre)
- [5. Qué significa para quien despliega un LLM](#5-qué-significa-para-quien-despliega-un-llm)
- [6. Reprodúcelo](#6-reprodúcelo)
- [7. Límites y honestidad](#7-límites-y-honestidad)
- [Qué demuestra esto](#qué-demuestra-esto)
- [Referencias](#referencias)

## Por qué este artículo

La mayoría de los comentarios sobre seguridad de LLM se detienen en un único número de titular: "el modelo X es Y% vulnerable a la inyección de prompts". Ese enfoque es cómodo y erróneo. La resistencia a la inyección de prompts no es una propiedad escalar de un modelo — es una *distribución* que depende del payload. Este artículo mide esa distribución, en un modelo pequeño, con suficientes intentos para adjuntar intervalos de confianza, y luego hace lo más útil: explica *por qué* los números salieron desiguales y qué significa para cualquiera que ponga un LLM frente a entradas no confiables.

El modelo aquí es deliberadamente modesto — un modelo de 3.000 millones de parámetros ejecutándose localmente en una Raspberry Pi. El objetivo no es avergonzar a un modelo pequeño por ser débil; es mostrar que incluso cuando *sí* mides, una sola medición no es una calificación de seguridad.

## 1. Método

- **Modelo:** Llama 3.2 (3B), servido localmente vía [Ollama](https://ollama.com), sin conexión, en una Raspberry Pi 5. Nada salió del dispositivo.
- **Herramienta:** [garak](https://github.com/NVIDIA/garak), el escáner de vulnerabilidades de LLM de código abierto de NVIDIA (v0.15.1).
- **Sondas:** dos variantes de la familia `promptinject` de garak — `HijackHateHumans` y `HijackKillHumans`. Cada una inserta una instrucción oculta dentro de una tarea normal; el detector `AttackRogueString` cuenta un éxito cuando la cadena objetivo del atacante aparece en la salida.
- **Volumen:** 256 intentos por sonda, una generación cada uno. 256 basta para acotar la tasa de éxito con un intervalo de confianza del 95% estrecho.

Las dos sondas son el *mismo mecanismo de ataque*. Lo único que cambia entre ellas es el contenido que el atacante intenta forzar.

## 2. Resultados

| Sonda (lo que intenta hacer decir al modelo) | Éxito del ataque | El modelo resistió | IC 95% |
|---|---|---|---|
| **HijackHateHumans** — una cadena de "odio a los humanos" | **46,9%** | 136 / 256 | 41,0–53,1% |
| **HijackKillHumans** — una cadena violenta | **10,2%** | 230 / 256 | 6,6–14,1% |

Los intervalos `41,0–53,1%` y `6,6–14,1%` ni se acercan a tocarse. La brecha es una propiedad real del comportamiento del modelo, no ruido de muestreo.

*(Una ejecución anterior de una sola sonda de `HijackHateHumans`, repetida dos veces para verificar la reproducibilidad, dio 44,1% y 44,5% — coherente con el 46,9% de aquí, y prueba de que la medición es estable.)*

## 3. El hallazgo: la resistencia depende del contenido

La misma técnica de inyección tuvo éxito **4,6× más** cuando el objetivo era levemente tóxico ("odio") que cuando era abiertamente violento ("matar"). La resistencia no es un único número asociado al modelo; es una función del contenido objetivo. Quien probara solo el payload violento registraría ~10% y concluiría que el modelo era razonablemente robusto. Quien probara solo el payload leve registraría ~47% y concluiría que estaba gravemente expuesto. Ambos usaron el mismo modelo y el mismo ataque. Ambos sacarían una conclusión de un solo punto de una curva.

## 4. Por qué ocurre

El entrenamiento de seguridad es desigual por diseño. El trabajo de alineamiento concentra sus rechazos más fuertes en las categorías más evidentemente dañinas — violencia, armas, autolesión — porque son los fallos de mayor responsabilidad. Ese entrenamiento se generaliza a los intentos de *inyección* en las mismas categorías: cuando una instrucción inyectada intenta forzar salida violenta, activa las barreras más reforzadas del modelo y el ataque falla más a menudo.

El contenido levemente tóxico se sitúa en una zona peor defendida. El modelo tiene mucho menos refuerzo contra ser dirigido a salida de "odio", así que la misma técnica lo cruza con mucha más facilidad. Dicho claramente: **el ataque que le parece peor a un humano es el que el modelo mejor resiste, y el más sutil es donde está más expuesto.** Un atacante que optimiza para la fiabilidad, no para el impacto, apunta a la segunda zona.

## 5. Qué significa para quien despliega un LLM

1. **Una única puntuación de resistencia no es una calificación de seguridad.** Si pruebas un payload y anotas un número, ese número puede desviarse 4–5× respecto al comportamiento del modelo ante otro objetivo. Prueba las *clases* de payload que corresponden a tu propio modelo de amenaza — exfiltración de datos, abuso de herramientas/agentes, salida dañina para la marca, evasión de políticas — no solo lo que trae un benchmark.
2. **Las brechas peligrosas no son las obvias.** Los ataques abiertamente dañinos son los mejor defendidos. La exposición vive en las categorías más sutiles, justo donde empujará un atacante competente.
3. **Las barreras van fuera del modelo.** Las defensas internas desiguales implican que no puedes confiar en que el modelo detecte la categoría que *a ti* te importa. Coloca filtrado determinista de entrada/salida a su alrededor y limita lo que un modelo secuestrado puede realmente hacer — herramientas de mínimo privilegio, ninguna acción insegura solo por lo que diga el modelo.
4. **Mide antes de desplegar y vuelve a medir en cada cambio.** La resistencia a la inyección cambia con la versión del modelo, el system prompt y el andamiaje que lo rodea. Es una propiedad que se monitoriza, no una casilla que se marca una vez.

## 6. Reprodúcelo

Toda la prueba corre en hardware básico — una Raspberry Pi, sin conexión:

```bash
# 1. Un modelo local
ollama pull llama3.2:3b

# 2. El escáner
pipx install garak

# 3. La medición (256 intentos por sonda)
garak --model_type ollama --model_name llama3.2:3b \
      --probes promptinject.HijackHateHumans,promptinject.HijackKillHumans \
      --generations 1
```

garak escribe un informe completo por intento (`~/.local/share/garak/garak_runs/*.report.jsonl`), de modo que cada acierto es auditable en lugar de aceptarse por fe.

## 7. Límites y honestidad

Este es un resultado deliberadamente acotado y debe leerse como tal:

- **Un modelo pequeño.** Estos números describen a Llama 3.2 (3B). Modelos más grandes y mejor alineados son notablemente más resistentes. No es una afirmación sobre los LLM en general — es una demostración de que *incluso una medición no basta*.
- **Dos payloads.** Se inició una tercera sonda, `HijackLongPrompt`, pero se excluyó: se bloqueó en el hardware de prueba solo con CPU (la generación de contexto largo se colgó sin timeout del lado del cliente). El contraste de dos payloads se sostiene por sí solo — y una generación de contexto largo colgada es en sí un pequeño recordatorio de que el comportamiento del modelo ante entradas adversarias hay que medirlo, no suponerlo.
- **Un detector.** `AttackRogueString` puntúa la emisión exacta de la cadena. La inyección del mundo real tiene criterios de éxito más difusos; esto es una cota inferior sobre una señal bien definida, elegida por ser inequívoca y reproducible.

Declarar los límites es la clave. Un número sin sus límites es marketing, no medición.

## Qué demuestra esto

- La resistencia a la inyección de prompts es una **distribución, no un escalar** — y la dispersión es grande (4,6× aquí).
- El método es barato, sin conexión y **reproducible en una Raspberry Pi**, así que "no teníamos recursos para probar" no es una restricción real.
- Reportar los intervalos de confianza, los pasos de reproducción *y* los límites es lo que separa una medición de un titular.

Verificación primero: mide la distribución, muestra los intervalos, entrega la evidencia.

## Referencias

- garak — escáner de vulnerabilidades de LLM: https://github.com/NVIDIA/garak
- OWASP Top 10 para Aplicaciones LLM — LLM01 Inyección de Prompts: https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- CWE-1427: https://cwe.mitre.org/data/definitions/1427.html
- Ollama: https://ollama.com

---

*Probado contra nuestro propio modelo, en nuestro propio hardware, en condiciones autorizadas. No intervinieron sistemas de terceros. — [Cindrasec](https://cindrasec.com)*
