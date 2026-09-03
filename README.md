# Konsensomat

Geheime Abstimmung für kleine Gruppen. Alle stimmen blind ab, das Ergebnis
erscheint erst, wenn alle Stimmen vorliegen.

Live: **https://konsensomat.happyharry.art** (Cloudflare Worker)

## Was der Administrator einstellt

| Einstellung | Möglichkeiten |
|---|---|
| Auswertung | einstimmig · einfache Mehrheit · Zweidrittelmehrheit |
| Antworten | vorgegebene Optionen (Ja/Nein, Kandidaten …) oder freie Eingabe |
| Enthaltung | optional, zählt nicht zu den gültigen Stimmen |
| Teilnehmer | 2 bis 50 |
| Zugangscodes | optional, ein Einmal-Code je Person, als QR-Wahlkarte druckbar |

Mehrheiten gibt es nur mit vorgegebenen Optionen. Bei freier Eingabe scheitert
jede Auszählung an Schreibweisen („Müller" vs. „mueller"), deshalb ist dort nur
Einstimmigkeit möglich (Schreibweise, Groß/Klein und Satzzeichen am Ende werden
dabei angeglichen).

## Wahlkarten mit QR-Code

Mit Zugangscodes erzeugt der Administrator unter `/a/<id>/karten` für jeden
Berechtigten eine eigene Karte: QR-Code, Code im Klartext und die Frage.
Ausdrucken, auseinanderschneiden, verteilen. Wer seine Karte scannt, landet
direkt bei der Abstimmung, der Code trägt sich selbst ein (er wandert aus der
Adresse sofort in ein Cookie, damit er nicht im Browserverlauf stehen bleibt)
und gilt genau einmal. Ohne Zugangscodes gibt es dort einen einzelnen QR-Code
für alle. Die Kartenseite sieht nur, wer die Abstimmung angelegt hat.

Ein Code belegt die Berechtigung, er sagt nichts darüber aus, wie jemand
gestimmt hat: die Verbindung zwischen Code und Stimme wird nirgends gespeichert.

Nach dem Anlegen gibt es einen Link zum Weitergeben. Die Adminrechte hängen an
einem Cookie im Browser des Anlegenden, nicht am Link: wer den Link
weiterschickt, verschenkt keine Rechte.

## Was gespeichert wird und was nicht

Nicht gespeichert:
- die Verbindung zwischen Person und Stimme (das war die Hauptschwäche der
  ersten Fassung),
- bei freier Eingabe der Klartext, solange die Runde läuft,
- nach dem Abschluss der Vergleichs-Hash und der Salt.

Gespeichert wird während der Runde nur, wer schon abgestimmt hat (als
sortierte, anonyme Kennungen) und bei Optionen ein Zähler je Option.

Restrisiko, ehrlich benannt: wer die Datenbank live mitlesen kann (also der
Betreiber selbst), sieht bei Optionen den Zwischenstand mitwachsen und könnte
daraus auf eine einzelne Stimme schließen, wenn er weiß, wer gerade abgestimmt
hat. Dagegen hilft nur ein Verfahren mit Schlüsseln auf den Geräten der
Teilnehmer (Commit-Reveal). Für Vereins- und Familienabstimmungen ist der
jetzige Stand angemessen.

## Härtung für den Vereinseinsatz (27.08.2026)

- **Zugangscodes**: 8 Stellen aus einem Alphabet mit 32 Zeichen ohne
  Verwechslungspaare (von O/0, I/1 und B/8 steht höchstens ein Zeichen drin),
  2^40 = über eine Billion Möglichkeiten. Bindestrich und Groß/Klein sind beim
  Eintippen egal, ein O statt 0 oder B statt 8 wird stillschweigend geheilt.
- **Bremse gegen Durchprobieren**: nach 12 falschen Codes sperrt die Abstimmung
  die Eingabe, nur der Wahlleiter kann sie wieder freigeben. Vorher war ein
  Code rechnerisch in Stunden zu erraten, jetzt nicht mehr.
- **Wahlquittungen**: jede Stimme erzeugt einen kurzen Beleg, den nur die
  Person sieht (Cookie). Nach Abschluss steht die sortierte Liste aller
  Quittungen im Ergebnis: jeder kann prüfen, dass die eigene Stimme gezählt
  wurde, ohne dass der Beleg den Inhalt verrät.
- **Ergebnis unveränderlich**: eine abgeschlossene Abstimmung lässt sich nicht
  zurücksetzen („neue Runde" wurde entfernt). Für eine Wiederholung legt man
  eine neue Abstimmung an, das alte Ergebnis bleibt stehen.
- **Selbstlöschung**: 60 Tage nach dem Anlegen räumt sich jede Abstimmung per
  Alarm selbst weg (Datensparsamkeit).
- **Sicherheitsköpfe**: strikte Content-Security-Policy, kein Einbetten in
  fremde Seiten (frame-ancestors 'none'), kein Referrer, kein Caching.
- **Nur eine Adresse**: in der Referenz-Installation ist die automatische
  workers.dev-Adresse abgeschaltet (der Standard dieses Repos lässt sie an,
  siehe „Selbst betreiben“).

Grenzen, offen benannt: der Betreiber des Cloudflare-Kontos kann den
Speicherinhalt einsehen (bei Optionen: Zwischenstände). Wer dem Wahlleiter
nicht vertrauen will, braucht ein Verfahren mit Schlüsseln auf den Geräten der
Teilnehmer, das ist bewusst nicht Teil dieses Werkzeugs.

## Teilnehmer-Landingpage und Probe-Abstimmung (27.08.)

- Wer seine Wahlkarte scannt, landet zuerst auf `/info`: der Erklärung mit einer
  Einverständniserklärung. Erst mit gesetztem Haken wird der Knopf „Zur
  Abstimmung" aktiv; der Zugangscode wartet solange im Cookie.
- Ohne Wahlkarte bietet `/info` eine **Probe-Abstimmung** (`/a/demo`) an: Frage
  „Willst du den Quellcode?", Ja/Nein, drei Teilnehmer je Runde, danach startet
  mit der nächsten Stimme automatisch eine neue Runde.
- **Jedes Gerät probt in seiner eigenen Runde** (Objekt `demo-<Wählerkennung
  aus dem Cookie>`). Teilen sich alle Besucher eine Runde, stimmt jeder in eine
  fremde hinein und verliert seine Stimme, sobald ein Fremder die Runde
  vollmacht; wer nachbaut, sollte das so lassen.
- Jede Demo-Stimme zählt zusätzlich in eine anonyme Strichliste über alle
  Geräte und Runden. Sie liegt getrennt von jeder Runde im Objekt `demo` unter
  dem Schlüssel `strichliste`. `/demo-stand` zeigt sie als Balkengrafik mit
  Knopf zum Leeren (`?json=1` liefert die nackten Zahlen). Die Probe-Abstimmung trägt
  persönliche Texte des Autors (Erläuterung, Screenshot der Anlege-Seite als
  eingebettetes Bild `DEMO_ADMIN_BILD`, Foto `DON_FOTO`) und lässt sich in
  `src/worker.js` leicht anpassen oder entfernen.
- Sagt eine Proberunde mehrheitlich Ja, zeigt das Ergebnis einen Gruß vom Don
  mit dem Link zu diesem Repository: die Demo hält, was die Frage verspricht.
- Layout für Tablet und Laptop: breitere Karte, größere Schrift, Antwortknöpfe
  nebeneinander.

## Technik

- Ein Durable Object je Abstimmung. Stimmen werden dadurch nacheinander
  verarbeitet und können sich nicht überschreiben. Die Vorgängerfassung nutzte
  KV ohne Sperre, dort konnten gleichzeitige Stimmen verlorengehen.
- Keine Abhängigkeiten, eine Datei: `src/worker.js`.

## Zugriffsschutz (empfohlen: Cloudflare Access)

Der Worker selbst hat keinen Login. Die Referenz-Installation schützt die
Anlege-Seite mit zwei Zero-Trust-Anwendungen (Cloudflare Access, im Free-Plan
enthalten); für die eigene Installation gleich nachbauen:

- **Konsensomat Teilnahme** (Bypass, alle): Pfade `/a/*` und `/info`. Teilnehmer
  brauchen keinen Login, ihre Legitimation ist die Wahlkarte.
- **Konsensomat Verwaltung** (Allow, nur die eigene E-Mail-Adresse): alles andere,
  insbesondere die Anlege-Seite `/`. Login per E-Mail-Einmalcode, Sitzung 24 h.
  Weitere Wahlleiter = weitere E-Mail-Adressen in der Allow-Policy.

Damit kann kein Fremder Abstimmungen unter der eigenen Domain anlegen, die wie
offizielle Abstimmungen aussehen. Ohne diesen Schutz funktioniert alles auch,
dann kann eben jeder mit dem Link Abstimmungen anlegen.

## Selbst betreiben

Voraussetzungen: ein Cloudflare-Konto (der kostenlose Plan reicht, Durable
Objects laufen dort als SQLite-Variante) und Node 20 oder neuer.

```
git clone https://github.com/HappyHarryArt/konsensomat
cd konsensomat
npm install
npx wrangler dev --port 8797 --local    # lokal ausprobieren
npx wrangler deploy                     # veroeffentlicht auf <name>.workers.dev
```

Vor dem Veröffentlichen den Block `BETREIBER` am Anfang von `src/worker.js`
ausfüllen: Name, Anschrift, Telefon, E-Mail, Ort, Copyright-Zeile und die eigene
Domain. Daraus baut die Anwendung das Impressum und die Datenschutzerklärung auf
`/info`, die Fußzeilen und `/llms.txt`. In Deutschland ist ein Impressum für
eine öffentlich erreichbare Seite Pflicht; im Repository stehen nur Platzhalter.

Beim ersten `wrangler deploy` fragt wrangler nach dem Cloudflare-Login und
legt alles Nötige an; die Abstimmung läuft danach unter der eigenen
workers.dev-Adresse.

Eigene Domain (optional): in `wrangler.jsonc` einen `routes`-Eintrag ergänzen

```jsonc
"workers_dev": false,
"routes": [
  { "pattern": "abstimmung.example.org", "zone_name": "example.org", "custom_domain": true }
]
```

und `workers_dev` abschalten, sonst bleibt die workers.dev-Adresse eine zweite
Tür, an der ein Access-Schutz der eigenen Domain vorbeiginge (workers.dev
lässt sich nicht mit Cloudflare Access absichern, es liegt nicht in der
eigenen Zone).

