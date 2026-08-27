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

- **Zugangscodes**: 8 Stellen aus einem Alphabet ohne verwechselbare Zeichen
  (O/0, I/1, B/8 fehlen), über eine Billion Möglichkeiten. Bindestrich und
  Groß/Klein sind beim Eintippen egal.
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
- **Nur eine Adresse**: die automatische workers.dev-Adresse ist abgeschaltet.

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
- Jede Demo-Stimme zählt zusätzlich in eine anonyme Strichliste über alle
  Runden. `/demo-stand` liefert sie als JSON und liegt hinter Cloudflare Access:
  Entscheidungshilfe für den Don, ob das Repository für Vereinsmitglieder
  freigegeben wird.
- Layout für Tablet und Laptop: breitere Karte, größere Schrift, Antwortknöpfe
  nebeneinander.

## Technik

- Ein Durable Object je Abstimmung. Stimmen werden dadurch nacheinander
  verarbeitet und können sich nicht überschreiben. Die Vorgängerfassung nutzte
  KV ohne Sperre, dort konnten gleichzeitige Stimmen verlorengehen.
- Keine Abhängigkeiten, eine Datei: `src/worker.js`.

## Zugriffsschutz (Cloudflare Access, 27.08.2026)

Zwei Zero-Trust-Anwendungen auf konsensomat.happyharry.art:

- **Konsensomat Teilnahme** (Bypass, alle): Pfade `/a/*` und `/info`. Teilnehmer
  brauchen keinen Login, ihre Legitimation ist die Wahlkarte.
- **Konsensomat Verwaltung** (Allow, nur die eigene E-Mail-Adresse): alles andere,
  insbesondere die Anlege-Seite `/`. Login per E-Mail-Einmalcode, Sitzung 24 h.
  Weitere Wahlleiter = weitere E-Mail-Adressen in der Policy „Verwaltung nur
  Volker".

Damit kann kein Fremder mehr Abstimmungen unter dieser Domain anlegen, die wie
offizielle Vereins-Abstimmungen aussehen.

## Erreichbarkeit

Öffentlich ist **nur** konsensomat.happyharry.art. Die automatische
workers.dev-Adresse ist abgeschaltet (`"workers_dev": false`): sie wäre eine
zweite Tür zur selben Anwendung, an der jeder Schutz auf der eigenen Domain
vorbeiginge, und sie lässt sich nicht mit Cloudflare Access absichern, weil sie
nicht in einer eigenen Zone liegt.

Entwickelt wird lokal mit `wrangler dev`. Wird später eine Testumgebung im Netz
gebraucht, gehört sie auf eine eigene Subdomain der eigenen Zone (z. B.
dev.konsensomat.happyharry.art) und dort hinter eine Access-Policy.

## Entwickeln und Deployen

```
nvm use 20
npx wrangler dev --port 8797 --local
npx wrangler deploy
```

