# Anatomie eines exponierten IAM-Frontends

**Eine vollständige Umgehung der Authentifizierung bei einem Asset aus einer Google-Übernahme — und eine präzise Darstellung, warum es in neun Tagen behoben wurde und warum eine Belohnung von 0 $ die richtige Entscheidung war.**

**🌐 Read this in your language:** [English](../2026-05-exposed-iam-frontend-google-vrp.md) · [Español](./2026-05-exposed-iam-frontend-google-vrp.es.md) · [Français](./2026-05-exposed-iam-frontend-google-vrp.fr.md) · **Deutsch** · [العربية](./2026-05-exposed-iam-frontend-google-vrp.ar.md) · [हिन्दी](./2026-05-exposed-iam-frontend-google-vrp.hi.md) · [বাংলা](./2026-05-exposed-iam-frontend-google-vrp.bn.md) · [简体中文](./2026-05-exposed-iam-frontend-google-vrp.zh.md) · [日本語](./2026-05-exposed-iam-frontend-google-vrp.ja.md)

![Program](https://img.shields.io/badge/Program-Google_VRP-4285F4)
![Status](https://img.shields.io/badge/Status-Fixed-success)
![Triage](https://img.shields.io/badge/Triage-P2_%2F_S2-orange)
![Reward](https://img.shields.io/badge/Reward-Credit_%2F_Honorable_Mention-lightgrey)
![Disclosure](https://img.shields.io/badge/Disclosure-Coordinated-blue)

> **CWE:** [CWE-287](https://cwe.mitre.org/data/definitions/287.html) (Fehlerhafte Authentifizierung) · [CWE-1188](https://cwe.mitre.org/data/definitions/1188.html) (Verwendung von Standard-Zugangsdaten) · [CWE-319](https://cwe.mitre.org/data/definitions/319.html) (Klartextübertragung)
> **Asset-Typ:** Google-Übernahme (Photomath), Stufe 1 gemäß `external_domains_acquisitions.asciipb`

**TL;DR** — Eine administrative IAM-Oberfläche lag offen im öffentlichen Internet, auf einer Subdomain aus einer Google-Übernahme. Der Login akzeptierte Standard-Zugangsdaten, dann *jedes beliebige* Passwort, und die dahinterliegende API beantwortete nicht authentifizierte Anfragen — ein vollständiges Versagen der Authentifizierungsschicht. Das Produktteam von Google stufte es als P2/S2 ein und legte es neun Tage nach Annahme des Berichts still. Das VRP-Belohnungsgremium sprach separat eine Anerkennung und kein Geld zu. Dieser Artikel schlüsselt die Exposition auf und tut dann das Schwierigere und Nützlichere: Er erklärt auf Mechanismus-Ebene, **warum beide Entscheidungen richtig sind und sich nicht widersprechen** — und welche Belege es über die Belohnungsschwelle gehoben hätten. Diese Lücke zu kalibrieren ist die eigentliche Fähigkeit.

---

## Inhalt
- [Warum ich das schreibe](#warum-ich-das-schreibe)
- [1. Entdeckung — und wie man diese Klasse gezielt findet](#1-entdeckung--und-wie-man-diese-klasse-gezielt-findet)
- [2. Die Authentifizierungsschwäche](#2-die-authentifizierungsschwäche)
- [3. Die dahinterliegende API-Schicht](#3-die-dahinterliegende-api-schicht)
- [4. Behebung](#4-behebung)
- [5. Grundursache, genau: über Edge bereitgestellt ≠ von Google betrieben](#5-grundursache-genau-über-edge-bereitgestellt--von-google-betrieben)
- [6. Warum dies zu Recht nicht belohnt wurde — und was das geändert hätte](#6-warum-dies-zu-recht-nicht-belohnt-wurde--und-was-das-geändert-hätte)
- [7. Wenn ich dies verteidigen würde](#7-wenn-ich-dies-verteidigen-würde)
- [8. Lektionen, die ich mitnehme](#8-lektionen-die-ich-mitnehme)
- [Was dieser Fund zeigt](#was-dieser-fund-zeigt)
- [Zeitleiste](#zeitleiste)
- [Referenzen](#referenzen)

## Warum ich das schreibe

Die meisten Bug-Bounty-Writeups enden bei „Ich habe X gefunden, hier ist die Auszahlung". Die nützlichere Geschichte ist meist die Lücke zwischen dem, wie schwerwiegend ein Fund *aussieht*, und dem, wie schwerwiegend er *ist* — denn diese Lücke richtig einzuschätzen ist die eigentliche Arbeit, auf beiden Seiten einer Triage-Warteschlange und in jedem Sicherheitsteam.

Dieser Fund wirkte an der Oberfläche kritisch: eine nicht authentifizierte administrative Oberfläche für Identitäts- und Zugriffsverwaltung (IAM), offen im öffentlichen Internet, auf einer Domain, die zu einer Google-Übernahme gehört. Das Produktteam von Google stimmte zu, dass es behoben werden sollte (P2/S2), und remedierte schnell. Das VRP-Belohnungsgremium entschied separat, dass es keine finanzielle Belohnung verdiente — und nachdem ich die Belege durchgearbeitet habe, halte ich ihre Begründung für genau richtig.

Dieser Artikel tut also zweierlei: Er seziert die technische Anatomie, und er erklärt — ehrlich und auf der Ebene des *Warum die Systeme sich so verhielten* — warum „schnell beheben" und „nichts zahlen" beide die richtigen Entscheidungen waren. Die zweite Hälfte ist der Teil, den eine Personalverantwortliche lesen sollte.

## 1. Entdeckung — und wie man diese Klasse gezielt findet

Das Asset war `rip.photomath.net`, eine Subdomain aus einer Google-Übernahme (Photomath). Sie tauchte nicht durch Glück auf; sie stammt aus einer wiederholbaren Methode für die ertragreichste Ecke eines großen, übernahmelastigen Geltungsbereichs — **veraltete, nicht migrierte Infrastruktur kürzlich übernommener Unternehmen**:

1. **Zählen Sie den Übernahme-Geltungsbereich auf, nicht nur das Flaggschiff.** Googles eigene Scope-Datei `external_domains_acquisitions.asciipb` klassifiziert Übernahme-Domains nach Stufe. `*.photomath.net` steht dort auf Stufe 1. In Übernahme-Subdomains steckt die Integrationsschuld.
2. **Passive Subdomain-Erweiterung** (Certificate-Transparency-Logs + historisches DNS) bringt Hosts wie `rip.` zum Vorschein, die in der Navigation des Produkts nie auftauchen.
3. **Jeden Host auflösen und mit Fingerprinting versehen**, dann hart nach den Anzeichen nicht migrierter Infrastruktur statt gehärteter Produktion filtern:

| Beobachtung | Wie es ermittelt wurde | Warum es zählt |
|---|---|---|
| Lieferte eine Admin-UI (`GestionUsersRolesFrontend`) aus | Direktes Laden im Browser | Eine privilegierte Verwaltungsfläche für Benutzer/Rollen |
| **Nur einfaches HTTP; TLS scheiterte auf `:443`** | `unexpected eof` beim HTTPS-Handshake | Ein Host außerhalb der standardmäßigen TLS-terminierenden Edge-Richtlinie des Erwerbers — ein Migrationszeichen (CWE-319) |
| **Von Google-Infrastruktur bereitgestellt** | Header `Via: 1.1 google`; GCP-IP bei der Auflösung | Leitet über Googles Edge — aber, wie Abschnitt 5 zeigt, ist über Edge bereitgestellt **nicht** dasselbe wie von Google betrieben |
| Nicht-englische Admin-Texte (`Bienvenue administrateur`), Platzhaltertext (`users works!`) | UI-Inspektion | Build des Ursprungsunternehmens, wahrscheinlich Dev/Sandbox, unverändert in die Übernahme übernommen |

Das Muster, bei dem ein erfahrener Jäger innehalten und hinsehen sollte: **ein Admin-Panel für Benutzer/Rollen, erreichbar über einfaches HTTP, ohne SSO / identitätsbewussten Proxy davor, auf einer Domain der Übernahme-Stufe.** Jedes dieser Merkmale ist ein Symptom für Infrastruktur, die geerbt und nie in den Sicherheitsperimeter des Erwerbers eingegliedert wurde.

## 2. Die Authentifizierungsschwäche

Der Login-Bildschirm (`Bienvenue administrateur`) akzeptierte das lehrbuchmäßige Standardpaar:

```
email:    admin@photomath.net
password: admin
```

Das allein ist CWE-1188 (Standard-Zugangsdaten). Doch weiteres Prüfen offenbarte etwas Grundlegenderes: Das Portal akzeptierte *jede beliebige* Passwortzeichenkette für das Admin-Konto. Das verschiebt es von „schwachen Zugangsdaten" zu **CWE-287 (defekte Authentifizierung)** — das Frontend führte überhaupt keine sinnvolle Prüfung der Zugangsdaten gegen ein Backend durch. Der Login war reine Dekoration.

Ich habe das bewusst bestätigt (Login mit zufälligen Zeichen), statt es anzunehmen, weil „Standard-Zugangsdaten funktionieren" und „Authentifizierung fehlt völlig" unterschiedliche Schweregrade sind und ich nur den beanspruchen wollte, den ich beweisen konnte.

**🎥 Nachweis — ein erfolgreicher Login mit einer zufälligen Passwortzeichenkette, der auf dem authentifizierten Rollen-Dashboard landet (unbearbeitete Bildschirmaufnahme):**

https://github.com/user-attachments/assets/dca3d51d-7b64-480f-ab64-b3c625b54832

## 3. Die dahinterliegende API-Schicht

Der Login mit einem Müll-Passwort führte zum vollständigen Admin-Dashboard — das sichtbare Ergebnis der oben beschriebenen defekten Authentifizierung:

![Administratives IAM-Dashboard ohne gültige Zugangsdaten erreicht](../images/rip-photomath-dashboard.png)

*Das IAM-Dashboard (`/dashboard`) ohne gültige Zugangsdaten erreicht — „Welcome, administrator", die Verwaltungs-Seitenleiste für Rollen/Benutzer, das Schreib-Steuerelement „New role" und keine echten Datensätze (eine leere Sandbox).*

Eine Umgehung auf UI-Ebene ist ein schwacher Fund, wenn das Backend die Autorisierung noch unabhängig durchsetzt. Die nächste Frage — die eine, die einen Screenshot von einem echten Fund trennt — war also: **Prüft die API hinter dieser UI die Authentifizierung selbst?** Tat sie nicht.

```http
GET /api/roles HTTP/1.1
Host: rip.photomath.net
# no auth headers, no session cookie

HTTP/1.1 200 OK
[]
```

![Nicht authentifiziertes GET /api/roles gibt [] mit einem 200 zurück](../images/rip-api-roles-unauth.png)

Ein nicht authentifiziertes GET gab ein leeres JSON-Array zurück, kein `401`/`403`. Ein nicht authentifiziertes `OPTIONS` bewarb den vollständigen schreibfähigen Methodensatz:

```console
$ curl -i -s -k -X OPTIONS "http://rip.photomath.net/api/roles"
HTTP/1.1 200 OK
Allow: POST,GET,HEAD,OPTIONS
Via: 1.1 google
```

![OPTIONS gibt Allow: POST,GET,HEAD,OPTIONS und Via: 1.1 google zurück](../images/rip-api-options-allow.png)

`Allow: POST,GET,HEAD,OPTIONS` zeigt, dass der Endpunkt Schreibvorgänge (`POST`) ohne Authentifizierung akzeptiert. Ein `404` auf einem nicht zugeordneten Pfad lieferte die Standard-Fehlerseite von Spring Boot:

![Spring-Boot-Whitelabel-Fehlerseite](../images/rip-spring-boot-whitelabel.png)

Somit fehlten zwei unabhängige Kontrollen — die Frontend-Authentifizierung und die Backend-Autorisierung — beide auf derselben Oberfläche. Das ist der architektonisch interessante Teil, und deshalb war der Fund *vollständig*: Ich habe nicht bei „der Login ist eine Attrappe" aufgehört, ich habe gezeigt, dass die Datenschicht selbst offen war.

> **Reproduktion (nur-lesende Zusammenfassung).**
> 1. Lösen Sie `rip.photomath.net` auf und laden Sie es über einfaches HTTP (`:443` scheitert am TLS-Handshake) — die Admin-UI (`GestionUsersRolesFrontend`) wird gerendert.
> 2. Senden Sie beim Login `Bienvenue administrateur` `admin@photomath.net` mit **einer beliebigen** Passwortzeichenkette → landet auf `/dashboard`.
> 3. `GET /api/roles` **ohne** Cookie oder Auth-Header → `200 OK`, Rumpf `[]` (nicht `401`/`403`).
> 4. `OPTIONS /api/roles` → `Allow: POST,GET,HEAD,OPTIONS` — Schreibmethoden ohne Authentifizierung beworben.
>
> Es wurden keine Schreibvorgänge ausgeführt und keine Datensätze erstellt oder verändert; die Schritte 3–4 belegen das Kontrollversagen, ohne den Schreibpfad auszuführen.

**Testumfang.** Ich bestätigte die Lese-Erreichbarkeit und den beworbenen Methodensatz. Ich habe **keine** Schreibvorgänge ausgeführt, keine Rollen erstellt und keinen Zustand verändert. Die Erreichbarkeit nachzuweisen genügte, um das Kontrollversagen zu belegen, und dort aufzuhören ist es, was die Safe-Harbor-Erwartungen verlangen. Zu behaupten, der Schreibpfad *funktioniere*, ohne ihn auszuführen, wäre übertrieben gewesen; festzuhalten, dass er *beworben* wurde, ist eine Tatsache.

## 4. Behebung

Googles Handhabung war schnell und sauber:

- **Innerhalb von ~24 Stunden angenommen** und an das zuständige Produktteam übergeben.
- **Neun Tage nach der Annahme als Behoben markiert** — der Endpunkt wurde stillgelegt und der Hostname begann, `NXDOMAIN` zurückzugeben. Ich habe das `NXDOMAIN` unabhängig erneut verifiziert und zurückgemeldet.
- Intern als **P2 / S2** eingestuft.

![rip.photomath.net gibt jetzt DNS_PROBE_FINISHED_NXDOMAIN zurück](../images/rip-nxdomain-fixed.png)

Beachten Sie, *wie* es behoben wurde: kein Code-Patch, keine Änderung an einer Auth-Middleware — der Eintrag wurde gezogen und der Host löste sich nicht mehr auf. Dieses Detail ist der ganze Schlüssel zur Belohnungsentscheidung und Gegenstand des nächsten Abschnitts.

## 5. Grundursache, genau: über Edge bereitgestellt ≠ von Google betrieben

Das ist der Teil, den die meisten Writeups auslassen, und der Teil, der tatsächlich alles erklärt.

`rip.photomath.net` löste sich zu einer Adresse hinter Googles Edge auf und gab `Via: 1.1 google` zurück. Es ist verlockend — und anfangs neigte ich dazu —, „der Verkehr läuft über Googles Edge" als „das ist ein von Google betriebenes Produktionssystem" zu lesen. **Das ist nicht dasselbe**, und der Unterschied ist der gesamte Fund:

- Ein **aus der Übernahme geerbter DNS-Eintrag** zeigte noch auf ein Deployment, das Photomath vor der Übernahme aufgesetzt und nie stillgelegt oder migriert hatte. Es war ein **verwaistes Frontend**, kein integrierter Google-Dienst.
- Weil es nie in den Perimeter des Erwerbers eingegliedert wurde, saß es **außerhalb des identitätsbewussten Proxys** (kein BeyondCorp/SSO-Tor) und **außerhalb der Standard-Edge-TLS-Richtlinie** (einfaches HTTP, gescheitertes `:443`). Das waren keine getrennten Bugs — sie sind alle dasselbe Symptom: *Diese Maschine wurde nie hinter den Zaun geholt.*
- Die Anwendung war fast sicher ein **Dev-/Sandbox-Build** — französische UI-Texte, Platzhalter `users works!` und ein leeres `/api/roles`. Dahinter gab es keine echten Benutzer, Zugangsdaten oder Datensätze.
- Dass die Behebung **„den DNS-Eintrag ziehen"** statt **„die App patchen"** war, bestätigt die Grundursache: Es gab keine eigene, betriebene Anwendung zum Patchen. Die Schwachstelle lebte in einem **losen Artefakt, das lediglich über Googles Infrastruktur aufgelöst wurde.**

Das ist die typische Form des **Übernahme-Integrationsrisikos**: Wird ein Unternehmen übernommen, werden seine DNS-Zonen, Cloud-Projekte und halb vergessenen Deployments nach einem Zeitplan migriert, und veraltete Einträge überleben in der Lücke. Sie zu finden und zu beheben lohnt sich absolut — aber ihre Grundursache ist *Inventar und Hygiene*, kein Defekt im Anwendungscode des Erwerbers.

## 6. Warum dies zu Recht nicht belohnt wurde — und was das geändert hätte

Es ist leicht, „nicht authentifizierter Admin-Zugang + defekte Authentifizierung + exponierte Schreib-API auf einem Google-Asset der Stufe 1" zu schreiben und es kritisch zu nennen. Ich habe es anfangs stark gerahmt. Aber **Schweregrad ist die realisierte Auswirkung auf Systeme und Daten, die wirklich zählen**, und die Begründung des Belohnungsgremiums war präzise (Zitat der Entscheidung):

> *„…befand sich nicht innerhalb einer Google-Anwendung, sondern war das Ergebnis veralteter DNS-Einträge. Da die Schwachstelle kein System unter unserer direkten operativen Kontrolle betraf, qualifiziert sie sich nicht für eine finanzielle Belohnung…"*

Drei Punkte sprechen gegen die dramatische Lesart, und alle drei folgen direkt aus Abschnitt 5:

- **(a) Keine Produktion.** Platzhaltertext und ein leeres `/api/roles` — ein exponiertes Admin-Panel über einer leeren Sandbox ist ein echtes Hygieneproblem, kein Datenleck.
- **(b) Hygiene, keine Anwendungsschwachstelle.** Die Behebung war das Stilllegen eines losen Endpunkts. Das VRP belohnt Defekte in Systemen, die Google betreibt, nicht verwaiste Artefakte, die zufällig über seinen Edge aufgelöst werden.
- **(c) Meine stärkste Auswirkungsbehauptung war spekulativ.** In meiner Berufung stützte ich mich auf einen **Marken-/Phishing-Vektor** — dass ein Angreifer Zugangsdaten von Mitarbeitern abgreifen könnte, die die vertraute Oberfläche wiedererkennen. Das ist eine *hypothetische sekundäre* Auswirkung, kein nachgewiesener Schaden, und es waren Adjektive, keine neue Tatsache. Das Gremium überdachte es und hielt die ursprüngliche Entscheidung zu Recht aufrecht.

### Die Unterscheidung, die entschied: Schweregrad-Schiene ≠ Belohnungs-Schiene

Ein **„Behoben"** mit P2/S2 ist ein *Engineering*-Signal — das Team hielt es für aufräumenswert. Es ist **kein** *Belohnungs*-Signal. Das Produktteam optimiert für „Sollte das aufgeräumt werden?"; das Belohnungsgremium optimiert für „Hat das ein echtes Risiko in einem System, das wir betreiben, offengelegt?". Das sind unterschiedliche Fragen mit unterschiedlichen Antworten, und sie zu vermengen ist ein häufiger früher Fehler — einer, den ich in der Berufung machte.

### Was die Belohnungsschwelle überschritten hätte (das nützliche Kontrafaktische)

Genau zu wissen, was fehlte, ist wertvoller als der Fund selbst. Bereits **eine** davon hätte das Ergebnis wahrscheinlich verändert — und jede ist ein konkreter nächster Test, kein Wunsch:

| Hätte ich nachgewiesen… | Warum es die Schwelle überschreitet |
|---|---|
| dass `/api/roles` **echte Benutzerdatensätze** zurückgab (tatsächliche PII, nicht `[]`) | Datenoffenlegung, für die Google verantwortlich ist — realisierte, nicht hypothetische Auswirkung |
| dass der **Schreibpfad** (`POST /api/roles`) eine Rolle erstellte, die **in ein authentifiziertes, von Google betriebenes System föderierte** (gemeinsames SSO/Session) | Rechteausweitung aus einer Hülle in die Produktion — bewiesener Pivot |
| dass der Host eine **Cookie-Domain** (`.photomath.net`) oder eine OAuth-`redirect_uri`-Allowlist mit einer *lebenden, authentifizierten* Produktions-App teilte | Sitzungs-/Token-Diebstahl gegen echte Benutzer — die klassische Dangling-Subdomain-Kette |

Nichts davon traf hier zu: leere Sandbox, keine gemeinsame Authentifizierungsfläche, kein erreichbarer Pivot in die Produktion. Das *vor* dem Eskalieren der Berufung zu erkennen, ist genau die Kalibrierung, die die zweite Entscheidung prüfte — und wo ich meine eigene Glaubwürdigkeit hätte wahren können.

## 7. Wenn ich dies verteidigen würde

Der Fund ist für ein Sicherheitsteam als Erkennungs- und Präventionslektion nützlicher denn als Kriegsgeschichte. Wenn mir dieser Perimeter gehörte:

**Erkennen**
- **Kontinuierliche Certificate-Transparency-Überwachung** für jede Übernahme-Domain (`*.photomath.net` und Geschwister) — sowohl neue als auch vergessene Hosts tauchen in CT auf.
- Ein **geplanter Auflösungs- und Klassifizierungs-Sweep** jeder Subdomain in der Scope-Datei der Übernahme-Stufe, der markiert: Klartext-HTTP-Admin-UIs, Hosts, die *nicht* hinter dem identitätsbewussten Proxy stehen, und jedes `2xx` auf einem nicht authentifizierten `/api/*`.
- **Abgleich verwaister DNS-Einträge**: Vergleichen Sie die lebende DNS-Zone mit dem Inventar der *absichtlich betriebenen* Deployments; alles, was ohne Eigentümer auflöst, ist per Definition ein Fund.

**Verhindern**
- Stellen Sie **alle** Übernahme-Assets **vor** der DNS-Umstellung hinter den identitätsbewussten Proxy (BeyondCorp-Stil), damit eine nicht migrierte Maschine geschlossen statt offen ausfällt.
- Erzwingen Sie **nur HTTPS am Edge** und verweigern Sie einfaches HTTP — das gescheiterte `:443` hier war ein kostenloses Frühwarnsignal, das ignoriert wurde.
- Behandeln Sie das **Off-Boarding von Übernahmen** als Checklistenpunkt: legen Sie Infrastruktur und Datensätze des Ursprungsunternehmens zu einer Frist still, nicht „irgendwann".

Die einzige Kontrolle, die die gesamte Exposition verhindert hätte, ist der identitätsbewusste Proxy: Mit ihm ist ein verwaistes Admin-Panel unerreichbar, egal wie defekt seine eigene Authentifizierung ist.

## 8. Lektionen, die ich mitnehme

- **Beweisen Sie die Auswirkung; leiten Sie sie nicht aus Etiketten ab.** „Stufe 1" beschreibt die *potenzielle* Sensibilität einer Domain, nicht den Schweregrad eines bestimmten Funds darauf. Was zählt, sind die tatsächlich gefährdeten Daten.
- **Unterscheiden Sie Hygiene von Schwachstelle — vor dem Schreiben.** Verwaistes DNS, Sandbox-Exposition und veraltete Endpunkte sind häufig gültig-aber-nur-Anerkennung. Diese Erwartung vorab zu setzen, hält den Bericht ehrlich und die Berufung diszipliniert.
- **Zweischichtiges Denken schlägt einschichtiges.** Zu prüfen, ob das Backend die Authentifizierung unabhängig durchsetzte — nicht nur das Login-Formular — hat dies vollständig gemacht. Fragen Sie stets, was die nächste Kontrolle darunter tut.
- **Berufungen brauchen eine neue Tatsache, keine lauteren Adjektive.** Wenn ich keine konkreten, neuen Belege hinzufügen kann, untergräbt lautere Sprache nur die Glaubwürdigkeit bei den Triagierenden. Die Auswirkung mit stärkeren Worten neu zu formulieren, ist genau der Grund, warum meine Berufung das Ergebnis nicht änderte (und nicht ändern sollte).
- **Über Edge bereitgestellt ist nicht betrieben von.** Wohin eine Anfrage geleitet wird, sagt nichts darüber aus, wem das Risiko gehört. Diese eine Unterscheidung ist der Unterschied zwischen einem belohnbaren Fund und einer Hygienenotiz.
- **Kalibrierung ist die Fähigkeit.** Jeder kann etwas finden, das alarmierend aussieht. Der professionelle Schritt ist, genau zu benennen, wie sehr es zählt — einschließlich, und besonders, wenn die ehrliche Antwort „weniger, als es zunächst schien" lautet.

## Was dieser Fund zeigt

Als Arbeitsprobe gelesen, sind die nützlichen Signale hier nicht der Bug selbst — sondern wie er behandelt wurde:

- **Gezielte Aufklärung, kein Gießkannenprinzip** — aufgetaucht über die Scope-Datei der Übernahme-Stufe plus CT-Logs, eine wiederholbare Methode, um nicht migrierte Infrastruktur zu finden.
- **Zweischichtige Kontrollanalyse** — ich prüfte, ob das Backend die Autorisierung unabhängig vom Login-Formular durchsetzte, nicht nur die UI.
- **Tests mit minimalem Eingriff** — ich bestätigte Erreichbarkeit und die beworbenen Schreibmethoden, ohne einen einzigen Schreibvorgang auszuführen oder Daten zu berühren.
- **Schweregrad-Kalibrierung unter Druck** — ich argumentierte den Fund präzise, räumte ein, wo meine Berufung spekulativ war, und akzeptierte das Ergebnis „nur Anerkennung", sobald die Belege klar waren.
- **Koordinierte Offenlegung** — über den Herstellerkanal gemeldet, auf die Behebung gewartet, den Fix erneut verifiziert (`NXDOMAIN`) und erst danach veröffentlicht.

## Zeitleiste

| Datum | Tag | Ereignis |
|---|---|---|
| 2026-05-05 | 0 | An Google VRP gemeldet; automatische Empfangsbestätigung |
| 2026-05-06 | +1 | **Angenommen**; Bug an das Produktteam übergeben |
| 2026-05-15 | +10 | **Als Behoben markiert** — Endpunkt stillgelegt, `NXDOMAIN`; erneut verifiziert und bestätigt |
| 2026-05-29 | +24 | Belohnungsgremium: **erreicht die Schwelle nicht** → Anerkennung / Honorable Mention |
| 2026-05-29 | +24 | Ich legte Berufung zur erneuten Prüfung ein |
| 2026-06-02 | +28 | Berufung geprüft und **bestätigt** — nur Anerkennung bestätigt (Begründung: veraltetes DNS) |

## Referenzen

- **MITRE CWE** — die Schwächen, denen dieser Fund entspricht: [CWE-287: Fehlerhafte Authentifizierung](https://cwe.mitre.org/data/definitions/287.html), [CWE-1188: Verwendung von Standard-Zugangsdaten](https://cwe.mitre.org/data/definitions/1188.html), [CWE-319: Klartextübertragung sensibler Informationen](https://cwe.mitre.org/data/definitions/319.html).
- **[Google Bug Hunters (VRP)](https://bughunters.google.com/)** — das Programm, über das dies gemeldet wurde; seine Regeln definieren, was sich für eine finanzielle Belohnung gegenüber einer Anerkennung qualifiziert, und untermauern die in Abschnitt 6 analysierte Belohnungsentscheidung.
- **Verwaistes DNS / Subdomain-Übernahme (subdomain takeover)** — die Risikoklasse, zu der dies gehört: ein DNS-Eintrag, der die Ressource überlebt, auf die er zeigt. Die Erkennungs- und Präventionskontrollen in Abschnitt 7 sind das Gegenstück des Verteidigers.

---

*Gemeldet über das Google-Bug-Hunters-Programm (Issue 509594209). Der betroffene Endpunkt wurde von Google vor der Veröffentlichung behoben und stillgelegt (`NXDOMAIN`). Es wurden keine Daten über das zur Bestätigung der Exposition strikt Notwendige hinaus abgerufen oder verändert. Dieser Writeup gibt meine eigene Analyse wieder und steht in keiner Verbindung zu Google und wird von Google nicht unterstützt.*
