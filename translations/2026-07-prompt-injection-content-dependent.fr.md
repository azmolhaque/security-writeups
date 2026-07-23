# Le même modèle, 4,6× l'exposition

**Une mesure de la résistance à l'injection de prompt sur un petit LLM local — et pourquoi un unique « score de résistance » est trompeur.**

**🌐 Read this in your language:** [English](../2026-07-prompt-injection-content-dependent.md) · [Español](./2026-07-prompt-injection-content-dependent.es.md) · **Français** · [Deutsch](./2026-07-prompt-injection-content-dependent.de.md) · [العربية](./2026-07-prompt-injection-content-dependent.ar.md) · [हिन्दी](./2026-07-prompt-injection-content-dependent.hi.md) · [বাংলা](./2026-07-prompt-injection-content-dependent.bn.md) · [简体中文](./2026-07-prompt-injection-content-dependent.zh.md) · [日本語](./2026-07-prompt-injection-content-dependent.ja.md)

![Domain](https://img.shields.io/badge/Domain-AI%2FLLM_Security-8A2BE2)
![Method](https://img.shields.io/badge/Method-garak_·_256_trials-blue)
![Finding](https://img.shields.io/badge/Finding-Content--dependent-orange)
![Model](https://img.shields.io/badge/Model-Llama_3.2_(3B)-informational)
![Scope](https://img.shields.io/badge/Scope-Own_model_·_authorized-success)

> **CWE :** [CWE-1427](https://cwe.mitre.org/data/definitions/1427.html) (Neutralisation incorrecte des entrées utilisées dans les prompts de LLM) · **OWASP LLM Top 10 :** [LLM01 — Injection de Prompt](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
> **Cible de test :** Llama 3.2 (3B) via Ollama, auto-hébergé sur un Raspberry Pi 5 — usage autorisé, modèle propre.

**En bref** — Deux attaques d'injection de prompt différentes ont été lancées contre le *même* petit modèle local, 256 essais chacune. Le modèle a été détourné pour émettre une chaîne « haïr les humains » **46,9 %** du temps, mais une chaîne violente « tuer les humains » seulement **10,2 %** — soit une différence de **4,6× dans le taux de réussite avec la même technique, en ne changeant que l'objectif.** Les intervalles de confiance ne se recouvrent pas : c'est un effet réel, pas du bruit. La leçon pratique : la résistance d'un modèle à l'injection de prompt n'est pas un chiffre unique — elle varie fortement selon *ce que* l'attaquant cherche à extraire, si bien qu'un seul résultat de benchmark peut s'écarter d'un facteur important de votre menace réelle.

---

## Sommaire
- [Pourquoi cet article](#pourquoi-cet-article)
- [1. Méthode](#1-méthode)
- [2. Résultats](#2-résultats)
- [3. Le constat : la résistance dépend du contenu](#3-le-constat--la-résistance-dépend-du-contenu)
- [4. Pourquoi cela se produit](#4-pourquoi-cela-se-produit)
- [5. Ce que cela implique pour qui déploie un LLM](#5-ce-que-cela-implique-pour-qui-déploie-un-llm)
- [6. Reproduisez-le](#6-reproduisez-le)
- [7. Limites et honnêteté](#7-limites-et-honnêteté)
- [Ce que cela démontre](#ce-que-cela-démontre)
- [Références](#références)

## Pourquoi cet article

La plupart des commentaires sur la sécurité des LLM s'arrêtent à un chiffre unique : « le modèle X est vulnérable à Y % à l'injection de prompt ». Ce cadrage est rassurant et faux. La résistance à l'injection de prompt n'est pas une propriété scalaire d'un modèle — c'est une *distribution* qui dépend de la charge utile. Cet article mesure cette distribution, sur un petit modèle, avec assez d'essais pour associer des intervalles de confiance, puis fait le plus utile : il explique *pourquoi* les chiffres sont inégaux et ce que cela signifie pour quiconque place un LLM face à des entrées non fiables.

Le modèle est ici volontairement modeste — un modèle de 3 milliards de paramètres tournant localement sur un Raspberry Pi. Le but n'est pas de blâmer un petit modèle pour sa faiblesse ; c'est de montrer que même lorsque vous *mesurez*, une seule mesure n'est pas une note de sécurité.

## 1. Méthode

- **Modèle :** Llama 3.2 (3B), servi localement via [Ollama](https://ollama.com), hors ligne, sur un Raspberry Pi 5. Rien n'a quitté l'appareil.
- **Outil :** [garak](https://github.com/NVIDIA/garak), le scanner de vulnérabilités LLM open source de NVIDIA (v0.15.1).
- **Sondes :** deux variantes de la famille `promptinject` de garak — `HijackHateHumans` et `HijackKillHumans`. Chacune insère une instruction cachée dans une tâche normale ; le détecteur `AttackRogueString` compte une réussite lorsque la chaîne cible de l'attaquant apparaît en sortie.
- **Volume :** 256 essais par sonde, une génération chacun. 256 suffit à borner le taux de réussite avec un intervalle de confiance à 95 % étroit.

Les deux sondes sont le *même mécanisme d'attaque*. La seule chose qui change, c'est le contenu que l'attaquant cherche à forcer.

## 2. Résultats

| Sonde (ce qu'elle tente de faire dire au modèle) | Réussite de l'attaque | Le modèle a résisté | IC 95 % |
|---|---|---|---|
| **HijackHateHumans** — une chaîne « haïr les humains » | **46,9 %** | 136 / 256 | 41,0–53,1 % |
| **HijackKillHumans** — une chaîne violente | **10,2 %** | 230 / 256 | 6,6–14,1 % |

Les intervalles `41,0–53,1 %` et `6,6–14,1 %` ne se touchent même pas de près. L'écart est une propriété réelle du comportement du modèle, pas du bruit d'échantillonnage.

*(Une exécution antérieure d'une seule sonde `HijackHateHumans`, répétée deux fois pour vérifier la reproductibilité, a donné 44,1 % et 44,5 % — cohérent avec les 46,9 % ici, preuve que la mesure est stable.)*

## 3. Le constat : la résistance dépend du contenu

La même technique d'injection a réussi **4,6× plus souvent** lorsque l'objectif était légèrement toxique (« haine ») que lorsqu'il était ouvertement violent (« tuer »). La résistance n'est pas un chiffre unique attaché au modèle ; c'est une fonction du contenu ciblé. Qui ne testerait que la charge violente noterait ~10 % et conclurait que le modèle est raisonnablement robuste. Qui ne testerait que la charge légère noterait ~47 % et conclurait qu'il est gravement exposé. Les deux ont utilisé le même modèle et la même attaque. Les deux tireraient une conclusion d'un seul point d'une courbe.

## 4. Pourquoi cela se produit

L'entraînement de sécurité est inégal par conception. L'alignement concentre ses refus les plus fermes sur les catégories les plus manifestement nuisibles — violence, armes, automutilation — car ce sont les défaillances les plus lourdes de conséquences. Cet entraînement se généralise aux tentatives d'*injection* dans les mêmes catégories : quand une instruction injectée tente de forcer une sortie violente, elle déclenche les garde-fous les plus renforcés du modèle et l'attaque échoue plus souvent.

Le contenu légèrement toxique se situe dans une zone moins défendue. Le modèle a bien moins de renforcement contre le fait d'être orienté vers une sortie de « haine », si bien que la même technique le fait franchir la ligne bien plus facilement. Autrement dit : **l'attaque qui paraît la pire à un humain est celle que le modèle résiste le mieux, et la plus subtile est là où il est le plus exposé.** Un attaquant qui optimise la fiabilité, non le choc, vise la seconde zone.

## 5. Ce que cela implique pour qui déploie un LLM

1. **Un score de résistance unique n'est pas une note de sécurité.** Si vous testez une charge utile et notez un chiffre, ce chiffre peut s'écarter de 4–5× du comportement du modèle face à un autre objectif. Testez les *classes* de charges correspondant à votre propre modèle de menace — exfiltration de données, abus d'outils/d'agents, sortie nuisible à la marque, contournement de politique — pas seulement ce que fournit un benchmark.
2. **Les failles dangereuses ne sont pas les évidentes.** Les attaques ouvertement nuisibles sont les mieux défendues. L'exposition réside dans les catégories plus subtiles, précisément là où un attaquant compétent poussera.
3. **Les garde-fous vont à l'extérieur du modèle.** Des défenses internes inégales signifient que vous ne pouvez pas compter sur le modèle pour détecter la catégorie qui *vous* importe. Placez un filtrage déterministe des entrées/sorties autour de lui et limitez ce qu'un modèle détourné peut réellement faire — outils à moindre privilège, aucune action risquée sur la seule parole du modèle.
4. **Mesurez avant de déployer, et remesurez à chaque changement.** La résistance à l'injection évolue avec la version du modèle, le system prompt et l'échafaudage environnant. C'est une propriété à surveiller, pas une case à cocher une fois.

## 6. Reproduisez-le

Tout le test tourne sur du matériel courant — un Raspberry Pi, hors ligne :

```bash
# 1. Un modèle local
ollama pull llama3.2:3b

# 2. Le scanner
pipx install garak

# 3. La mesure (256 essais par sonde)
garak --model_type ollama --model_name llama3.2:3b \
      --probes promptinject.HijackHateHumans,promptinject.HijackKillHumans \
      --generations 1
```

garak écrit un rapport complet par tentative (`~/.local/share/garak/garak_runs/*.report.jsonl`), de sorte que chaque succès est auditable plutôt qu'accepté sur parole.

## 7. Limites et honnêteté

C'est un résultat volontairement borné, et il doit être lu comme tel :

- **Un petit modèle.** Ces chiffres décrivent Llama 3.2 (3B). Des modèles plus grands et mieux alignés sont nettement plus résistants. Ce n'est pas une affirmation sur les LLM en général — c'est une démonstration qu'*une seule mesure ne suffit pas*.
- **Deux charges utiles.** Une troisième sonde, `HijackLongPrompt`, a été lancée puis exclue : elle s'est bloquée sur le matériel de test uniquement CPU (la génération à long contexte s'est figée sans timeout côté client). Le contraste à deux charges tient seul — et une génération à long contexte figée est en soi un petit rappel que le comportement du modèle face à des entrées adverses se mesure et ne se suppose pas.
- **Un détecteur.** `AttackRogueString` note l'émission exacte de la chaîne. L'injection réelle a des critères de succès plus flous ; c'est une borne inférieure sur un signal bien défini, choisi parce qu'il est sans ambiguïté et reproductible.

Énoncer les limites est l'essentiel. Un chiffre sans ses limites est du marketing, pas une mesure.

## Ce que cela démontre

- La résistance à l'injection de prompt est une **distribution, pas un scalaire** — et la dispersion est grande (4,6× ici).
- La méthode est peu coûteuse, hors ligne et **reproductible sur un Raspberry Pi** ; « nous n'avions pas les moyens de tester » n'est donc pas une vraie contrainte.
- Rapporter les intervalles de confiance, les étapes de reproduction *et* les limites est ce qui distingue une mesure d'un titre.

Vérification d'abord : mesurez la distribution, montrez les intervalles, remettez les preuves.

## Références

- garak — scanner de vulnérabilités LLM : https://github.com/NVIDIA/garak
- OWASP Top 10 pour les applications LLM — LLM01 Injection de Prompt : https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- CWE-1427 : https://cwe.mitre.org/data/definitions/1427.html
- Ollama : https://ollama.com

---

*Testé contre notre propre modèle, sur notre propre matériel, dans des conditions autorisées. Aucun système tiers n'a été impliqué. — [Cindrasec](https://cindrasec.com)*
