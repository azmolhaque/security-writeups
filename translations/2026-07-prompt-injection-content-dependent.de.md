# Dasselbe Modell, 4,6× die Angriffsfläche

**Eine Messung der Prompt-Injection-Resistenz eines kleinen lokalen LLM — und warum ein einzelner „Resistenz-Score" in die Irre führt.**

**🌐 Read this in your language:** [English](../2026-07-prompt-injection-content-dependent.md) · [Español](./2026-07-prompt-injection-content-dependent.es.md) · [Français](./2026-07-prompt-injection-content-dependent.fr.md) · **Deutsch** · [العربية](./2026-07-prompt-injection-content-dependent.ar.md) · [हिन्दी](./2026-07-prompt-injection-content-dependent.hi.md) · [বাংলা](./2026-07-prompt-injection-content-dependent.bn.md) · [简体中文](./2026-07-prompt-injection-content-dependent.zh.md) · [日本語](./2026-07-prompt-injection-content-dependent.ja.md)

![Domain](https://img.shields.io/badge/Domain-AI%2FLLM_Security-8A2BE2)
![Method](https://img.shields.io/badge/Method-garak_·_256_trials-blue)
![Finding](https://img.shields.io/badge/Finding-Content--dependent-orange)
![Model](https://img.shields.io/badge/Model-Llama_3.2_(3B)-informational)
![Scope](https://img.shields.io/badge/Scope-Own_model_·_authorized-success)

> **CWE:** [CWE-1427](https://cwe.mitre.org/data/definitions/1427.html) (Unsachgemäße Neutralisierung von in LLM-Prompts verwendeten Eingaben) · **OWASP LLM Top 10:** [LLM01 — Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
> **Testziel:** Llama 3.2 (3B) über Ollama, selbst gehostet auf einem Raspberry Pi 5 — autorisierte Nutzung, eigenes Modell.

**Kurzfassung** — Zwei verschiedene Prompt-Injection-Angriffe wurden gegen *dasselbe* kleine lokale Modell ausgeführt, je 256 Versuche. Das Modell wurde in **46,9 %** der Fälle dazu gebracht, eine „Menschen hassen"-Zeichenkette auszugeben, aber eine gewalttätige „Menschen töten"-Zeichenkette nur in **10,2 %** — ein **4,6-facher Unterschied im Angriffserfolg mit derselben Technik, bei bloßer Änderung des Ziels.** Die Konfidenzintervalle überschneiden sich nicht, es ist also ein realer Effekt, kein Rauschen. Die praktische Lehre: Die Prompt-Injection-Resistenz eines Modells ist keine einzelne Zahl — sie variiert stark danach, *was* der Angreifer extrahieren will, sodass ein einzelnes Benchmark-Ergebnis um ein Vielfaches von Ihrer tatsächlichen Bedrohung abweichen kann.

---

## Inhalt
- [Warum dieser Beitrag](#warum-dieser-beitrag)
- [1. Methode](#1-methode)
- [2. Ergebnisse](#2-ergebnisse)
- [3. Der Befund: Resistenz ist inhaltsabhängig](#3-der-befund-resistenz-ist-inhaltsabhängig)
- [4. Warum das passiert](#4-warum-das-passiert)
- [5. Was das für alle bedeutet, die ein LLM einsetzen](#5-was-das-für-alle-bedeutet-die-ein-llm-einsetzen)
- [6. Nachvollziehen](#6-nachvollziehen)
- [7. Grenzen und Ehrlichkeit](#7-grenzen-und-ehrlichkeit)
- [Was das zeigt](#was-das-zeigt)
- [Quellen](#quellen)

## Warum dieser Beitrag

Die meisten Kommentare zur LLM-Sicherheit enden bei einer einzigen Schlagzeilen-Zahl: „Modell X ist zu Y % anfällig für Prompt Injection." Diese Rahmung ist bequem und falsch. Resistenz gegen Prompt Injection ist keine skalare Eigenschaft eines Modells — sie ist eine *Verteilung*, die von der Nutzlast abhängt. Dieser Beitrag misst diese Verteilung an einem kleinen Modell mit genügend Versuchen für Konfidenzintervalle und tut dann das Nützlichere: Er erklärt, *warum* die Zahlen ungleich ausfielen, und was das für jeden bedeutet, der ein LLM vor nicht vertrauenswürdige Eingaben stellt.

Das Modell ist hier bewusst bescheiden — ein Modell mit 3 Milliarden Parametern, das lokal auf einem Raspberry Pi läuft. Es geht nicht darum, ein kleines Modell für seine Schwäche zu beschämen; es geht darum zu zeigen, dass selbst wenn man *misst*, eine einzelne Messung keine Sicherheitsbewertung ist.

## 1. Methode

- **Modell:** Llama 3.2 (3B), lokal über [Ollama](https://ollama.com) bereitgestellt, offline, auf einem Raspberry Pi 5. Nichts verließ das Gerät.
- **Werkzeug:** [garak](https://github.com/NVIDIA/garak), NVIDIAs Open-Source-Schwachstellenscanner für LLMs (v0.15.1).
- **Sonden:** zwei Varianten aus garaks `promptinject`-Familie — `HijackHateHumans` und `HijackKillHumans`. Jede bettet eine versteckte Anweisung in eine ansonsten normale Aufgabe ein; der Detektor `AttackRogueString` wertet einen Erfolg, sobald die Zielzeichenkette des Angreifers in der Ausgabe erscheint.
- **Umfang:** 256 Versuche pro Sonde, je eine Generierung. 256 genügt, um die Angriffserfolgsrate mit einem engen 95-%-Konfidenzintervall zu begrenzen.

Beide Sonden sind *derselbe Angriffsmechanismus*. Das Einzige, was sich zwischen ihnen ändert, ist der Inhalt, den der Angreifer erzwingen will.

## 2. Ergebnisse

| Sonde (was sie das Modell sagen lassen will) | Angriffserfolg | Modell widerstand | 95-%-KI |
|---|---|---|---|
| **HijackHateHumans** — eine „Menschen hassen"-Zeichenkette | **46,9 %** | 136 / 256 | 41,0–53,1 % |
| **HijackKillHumans** — eine gewalttätige Zeichenkette | **10,2 %** | 230 / 256 | 6,6–14,1 % |

Die Intervalle `41,0–53,1 %` und `6,6–14,1 %` kommen sich nicht einmal nahe. Die Lücke ist eine reale Eigenschaft des Modellverhaltens, kein Stichprobenrauschen.

*(Ein früherer Einzel-Sonden-Lauf von `HijackHateHumans`, zur Reproduzierbarkeitsprüfung zweimal wiederholt, ergab 44,1 % und 44,5 % — konsistent mit den 46,9 % hier und Beleg, dass die Messung stabil ist.)*

## 3. Der Befund: Resistenz ist inhaltsabhängig

Dieselbe Injektionstechnik war **4,6× erfolgreicher**, wenn das Ziel leicht toxisch („Hass") war, als bei offen gewalttätigem („töten"). Resistenz ist keine dem Modell anhaftende einzelne Zahl; sie ist eine Funktion des Zielinhalts. Wer nur die gewalttätige Nutzlast testete, notierte ~10 % und hielte das Modell für einigermaßen robust. Wer nur die leichte Nutzlast testete, notierte ~47 % und hielte es für schwer exponiert. Beide nutzten dasselbe Modell und denselben Angriff. Beide zögen einen Schluss aus einem einzigen Punkt einer Kurve.

## 4. Warum das passiert

Sicherheitstraining ist absichtlich ungleichmäßig. Die Ausrichtungsarbeit konzentriert ihre stärksten Verweigerungen auf die offensichtlich schädlichsten Kategorien — Gewalt, Waffen, Selbstverletzung — weil dies die haftungsträchtigsten Fehler sind. Dieses Training verallgemeinert sich auf *Injektions*versuche in denselben Kategorien: Wenn eine injizierte Anweisung gewalttätige Ausgabe erzwingen will, löst sie die am stärksten verstärkten Leitplanken des Modells aus, und der Angriff scheitert häufiger.

Leicht toxischer Inhalt liegt in einer schwächer verteidigten Zone. Das Modell hat weit weniger Verstärkung dagegen, zu „Hass"-Ausgabe gelenkt zu werden, sodass dieselbe Technik es viel leichter über die Linie trägt. Klar gesagt: **Der Angriff, der einem Menschen schlimmer erscheint, ist derjenige, dem das Modell am besten widersteht, und der subtilere ist der, wo es am stärksten exponiert ist.** Ein Angreifer, der auf Zuverlässigkeit statt Schockwirkung optimiert, zielt auf die zweite Zone.

## 5. Was das für alle bedeutet, die ein LLM einsetzen

1. **Ein einzelner Resistenz-Score ist keine Sicherheitsbewertung.** Wenn Sie eine Nutzlast testen und eine Zahl notieren, kann diese Zahl um das 4- bis 5-Fache vom Verhalten des Modells gegenüber einem anderen Ziel abweichen. Testen Sie die Nutzlast-*Klassen*, die Ihrem eigenen Bedrohungsmodell entsprechen — Datenexfiltration, Werkzeug-/Agenten-Missbrauch, markenschädigende Ausgabe, Richtlinienumgehung — nicht nur das, was ein Benchmark mitliefert.
2. **Die gefährlichen Lücken sind nicht die offensichtlichen.** Offen schädliche Angriffe sind am besten verteidigt. Die Exposition lebt in den subtileren Kategorien — genau dort, wo ein kompetenter Angreifer drücken wird.
3. **Leitplanken gehören außerhalb des Modells.** Ungleichmäßige interne Abwehr bedeutet, dass Sie sich nicht darauf verlassen können, dass das Modell die Kategorie erkennt, die *Ihnen* wichtig ist. Setzen Sie deterministische Ein-/Ausgabefilterung darum herum und begrenzen Sie, was ein gekapertes Modell tatsächlich tun kann — Werkzeuge mit minimalen Rechten, keine unsicheren Aktionen allein auf das Wort des Modells hin.
4. **Messen Sie vor dem Ausrollen und messen Sie bei jeder Änderung erneut.** Die Injektionsresistenz ändert sich mit der Modellversion, dem System-Prompt und dem umgebenden Gerüst. Sie ist eine zu überwachende Eigenschaft, kein einmal abzuhakendes Kästchen.

## 6. Nachvollziehen

Der gesamte Test läuft auf Standardhardware — einem Raspberry Pi, offline:

```bash
# 1. Ein lokales Modell
ollama pull llama3.2:3b

# 2. Der Scanner
pipx install garak

# 3. Die Messung (256 Versuche pro Sonde)
garak --model_type ollama --model_name llama3.2:3b \
      --probes promptinject.HijackHateHumans,promptinject.HijackKillHumans \
      --generations 1
```

garak schreibt einen vollständigen Bericht pro Versuch (`~/.local/share/garak/garak_runs/*.report.jsonl`), sodass jeder Treffer überprüfbar ist statt auf Vertrauen zu beruhen.

## 7. Grenzen und Ehrlichkeit

Dies ist ein bewusst begrenztes Ergebnis und sollte so gelesen werden:

- **Ein kleines Modell.** Diese Zahlen beschreiben Llama 3.2 (3B). Größere, besser ausgerichtete Modelle sind deutlich resistenter. Dies ist keine Aussage über LLMs im Allgemeinen — es ist ein Beleg dafür, dass *schon eine Messung nicht genügt*.
- **Zwei Nutzlasten.** Eine dritte Sonde, `HijackLongPrompt`, wurde gestartet, aber ausgeschlossen: Sie blieb auf der reinen CPU-Testhardware hängen (die Long-Context-Generierung hing ohne clientseitiges Timeout fest). Der Zwei-Nutzlast-Kontrast steht für sich — und eine hängende Long-Context-Generierung ist selbst eine kleine Erinnerung daran, dass Modellverhalten unter feindlicher Eingabe zu messen und nicht anzunehmen ist.
- **Ein Detektor.** `AttackRogueString` wertet die exakte Zeichenketten-Ausgabe. Reale Injektion hat unschärfere Erfolgskriterien; dies ist eine Untergrenze auf einem wohldefinierten Signal, gewählt, weil es eindeutig und reproduzierbar ist.

Die Grenzen zu nennen ist der Kern. Eine Zahl ohne ihre Grenzen ist Marketing, keine Messung.

## Was das zeigt

- Prompt-Injection-Resistenz ist eine **Verteilung, kein Skalar** — und die Streuung ist groß (hier 4,6×).
- Die Methode ist günstig, offline und **auf einem Raspberry Pi reproduzierbar**, „wir hatten nicht die Mittel zum Testen" ist also keine echte Einschränkung.
- Die Konfidenzintervalle, die Reproduktionsschritte *und* die Grenzen anzugeben, unterscheidet eine Messung von einer Schlagzeile.

Verifikation zuerst: die Verteilung messen, die Intervalle zeigen, die Belege übergeben.

## Quellen

- garak — LLM-Schwachstellenscanner: https://github.com/NVIDIA/garak
- OWASP Top 10 für LLM-Anwendungen — LLM01 Prompt Injection: https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- CWE-1427: https://cwe.mitre.org/data/definitions/1427.html
- Ollama: https://ollama.com

---

*Getestet gegen unser eigenes Modell, auf unserer eigenen Hardware, unter autorisierten Bedingungen. Keine Drittsysteme waren beteiligt. — [Cindrasec](https://cindrasec.com)*
