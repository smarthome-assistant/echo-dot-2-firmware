# Eigene Änderungen an der Firmware

Hier liegen **unsere eigenen** Änderungen am Quelltext des Ursprungsprojekts, jede als Patch gegen
einen benannten Stand. Nichts hiervon stammt von Amazon; alles ist eigene Arbeit und könnte dem
Ursprungsprojekt angeboten werden (GPL/Apache, siehe `..\docs\quellen.md`).

**Warum als Patch und nicht als Abzweig:** `fremd\` bleibt unverändert (CLAUDE.md), Arbeitskopien
liegen flüchtig unter `build\`. Ein Patch überlebt beides und sagt genau, was wir geändert haben
und wogegen.

## 0001 — mDNS: das Gerät kündigt sich von selbst an

| | |
|---|---|
| **Datei** | `0001-mdns-regelmaessige-ankuendigung.patch` |
| **Gegen** | TECHO5 `dot-v1.0.72` (Commit `2d021e42`) |
| **Berührt** | `echod/internal/feature/api/mdns.go`, 48 Zeilen dazu, 3 geändert |
| **Stand** | **am eigenen Gerät erprobt**, läuft seit 19.11.2026 ~08:52 |
| **Anlass** | Markus, 19.11.2026: „Kannst du das auto discovery für den echo dot realisieren?" |

**Was es tut.** Der mDNS-Eintrag ging bisher einmal raus — bei der Anmeldung — und danach nur noch
bei einem Adress- oder Schlüsselwechsel. Auf Abfragen antwortet das Gerät zwar korrekt, aber es
hört eine Abfrage nicht, solange es nach einem Gruppenschlüssel-Wechsel des Routers taub für
Rundrufe ist (Fehler #45; am eigenen Gerät gemessen: drei Minuten beantwortet, dann zehn Minuten
still). Wer in so einem Fenster sucht, findet nichts.

Neu wird die Registrierung alle drei Minuten erneuert. **Senden ist von dem Fehler nicht
betroffen** — nur Empfangen. Die Ankündigung erreicht Home Assistant also auch dann, wenn das
Gerät gerade nichts hört.

**Gemessen.** Elf Minuten lauschen ohne jede Abfrage: vollständige Ankündigungen (9 Antworten,
701 B, danach zweimal 443 B) im Abstand von **182,4 s** — die eingestellten 180 s plus die zwei
Sekunden Abmelde-Pause. Vorher kam in 90 Sekunden kein einziges `_esphomelib`-Paket.

**Was es nicht tut.** Es behebt den Fehler nicht, es umgeht ihn in einer Richtung. SSH-Entdeckung,
Einrichtungsseite, AirPlay und Mehrraum-Ton bleiben in den Taubheitsphasen betroffen. Dafür
braucht es den Kernel-Patch (`..\spikes\kernel_vorbereiten.md`).

**Kosten.** Ein paar hundert Bytes alle drei Minuten. Jede Ankündigung ist eine Abmeldung und ein
neuer Eintrag zwei Sekunden später — ein Browser sieht das als „Gerät weg, Gerät wieder da", was
Home Assistant dazu bringt, es erneut anzubieten. Ein bereits aufgenommenes Gerät hat seine eigene
Verbindung und merkt nichts davon.

**Anwenden:**

```
git -C <techo5-arbeitskopie> checkout dot-v1.0.72
git -C <techo5-arbeitskopie> apply patches/0001-mdns-regelmaessige-ankuendigung.patch
```

**Bauen** (die Werte für `Version`, `GitCommit` und `BuildDate` so wie unten, damit der eigene Bau
erkennbar bleibt):

```
cd <arbeitskopie>/echod
GOOS=linux GOARCH=arm GOARM=7 CGO_ENABLED=0 go build -tags dot -trimpath \
  -ldflags "-s -w -X '<pkg>.Version=v1.0.72_mdns2' -X '<pkg>.GitCommit=2d021e4' -X '<pkg>.BuildDate=<jetzt>'" \
  -o echod-mdns ./cmd/echod
# <pkg> = github.com/HuskerMinion/techo5/echod/internal/layout
```

**Einspielen und Rückweg:** `..\docs\ablauf-protokoll.md`, Abschnitt S11.

### Fallstrick bei der Versionsmarke

Die erste Fassung trug `v1.0.72+mdns-ankuendigung`. Das Gerät meldete daraufhin beim Start:

```
ERROR this build's version is not one Home Assistant can rank, so it will offer an update
      that never settles
```

Home Assistant kann nur `vX.Y.Z` lesen, wahlweise mit `-vorabfassung` und `_zusatz`
(`internal/update/manifest.go`, `versionPattern`). Ein `+` ist nicht erlaubt, und eine unlesbare
Fassung führt dazu, dass Home Assistant **dauerhaft ein Update anbietet, das nie fertig wird**.
Alles nach dem Unterstrich wird beim Vergleich weggeworfen — ein Eigenbau heißt also so:

    v1.0.72_mdns2     richtig: wird als v1.0.72 eingeordnet, die Marke bleibt sichtbar
    v1.0.72+mdns      falsch:  Home Assistant bietet endlos ein Update an

**Für jeden eigenen Bau:** Versionsmarke mit Unterstrich, und nach dem ersten Start ins Protokoll
sehen, ob dort eine `ERROR`-Zeile steht.

## 0002 — Abschalter für die Ankündigung

| | |
|---|---|
| **Gegen** | TECHO5 `dot-v1.0.72` mit 0001 darauf |
| **Berührt** | `internal/config/discovery.go` (neu), `config.go`, `internal/feature/setup/discovery.go` (neu), `page.go`, `internal/feature/api/mdns.go` |
| **Anlass** | Markus, 19.11.2026: „Sollte aber in der einstellungsoberfläche ausgeschaltet werden können." |

Ein Kästchen auf der Einrichtungsseite unter **Network → Being found on the network**. An, wie es
kommt. Aus geht das Gerät still und wartet, bis es gefragt wird — sinnvoll, wenn es ohnehin per
Adresse in Home Assistant steht.

**Was der Abschalter nicht abschaltet** — drei Dinge, die zum Dasein im Netz gehören und nicht
zum Werben:

1. **Antworten auf Abfragen.** Das tut jeder angemeldete Name.
2. **Die eine Ankündigung bei der Anmeldung** — beim Start und nach einer neuen Verbindung. Sie
   ist Teil des Anmeldens selbst; am 19.11.2026 im Test beobachtet und als richtig bestätigt.
3. **Die Nachmeldung nach einem Adresswechsel.** Ohne sie wäre ein Gerät mit neuer Adresse für
   Home Assistant verloren, bis irgendetwas neu startet.

Die Einstellung wird bei jedem Schleifendurchlauf frisch gelesen (alle drei Sekunden), wirkt also
ohne Neustart.

## 0003 — Umschalter „Zu Alexa"

| | |
|---|---|
| **Berührt** | `internal/feature/setup/toalexa_dot.go` (neu), `toalexa_other.go` (neu), `page.go` |
| **Anlass** | Markus, 19.11.2026: „Wie kann die wahl getroffen werden? Es muss sehr einfach sein" |
| **Stand** | gebaut; **am Gerät noch nicht ausgelöst** — die Vorarbeiten stehen aus |

> **Erneuert am 20.11.2026 (Paket 2):** Die Textberichtigung vom 20.11. („zehn Sekunden halten,
> auf **grün** warten" statt „loslassen, wenn der Ring weiß wird") stand nur in der Arbeitskopie
> und im gebauten Programm, **nicht in diesem Patch**. Aufgefallen beim Nachbau aus frischem Klon:
> Das Ergebnis wich vom Programm am Gerät ab. Neu erzeugt aus dem Stand, aus dem `v1.0.72_kanal2`
> gebaut ist; danach ist frischer Klon + 0001–0007 **bitgleich** mit dem Programm am Gerät und im
> Release. **Das öffentliche Repo trägt noch die alte Fassung** (Blob `ee49d132`) — Berichtigung
> dort auf Markus' Wort.

Ein Knopf unter **System**, der das Startfeld in `misc` leert und neu startet; danach läuft Fire OS
mit Alexa. Zurück geht es mit der **Lauter-Taste beim Einstecken** — in Fire OS läuft nichts von
uns, also kann es dort keinen Knopf geben.

**Zwei Kästchen müssen angekreuzt sein**, sonst passiert nichts: dass der Rückweg erprobt wurde,
und dass Amazons Updates das Gerät wieder sperren können. Begründung und Vorarbeiten in
`..\docs\alexa-umschalter.md`.

Der Schreibvorgang berührt **nur die ersten 32 Byte** von `misc` (die Nachricht an den Startlader);
die A/B-Verwaltung und die Brotkrumen der Startumgebung liegen dahinter und bleiben unberührt. Es
wird zurückgelesen, bevor der Neustart ausgelöst wird.

## 0004 — die beiden neuen Karten in die Seite einhängen

| | |
|---|---|
| **Berührt** | `internal/feature/setup/page.go` |

Drei Zeilen: die Abschalter-Karte in den Netzwerk-Bereich, die Alexa-Karte ans Ende des
System-Bereichs, und zwei Fälle im Speicher-Handler. Getrennt gehalten, weil `page.go` die Datei
ist, die sich im Ursprungsprojekt am häufigsten ändert — so bleiben 0002 und 0003 auch dann
anwendbar, wenn diese drei Zeilen nicht mehr passen.

## Reihenfolge beim Anwenden

```
git checkout dot-v1.0.72
git apply patches/0001-mdns-regelmaessige-ankuendigung.patch
git apply patches/0002-mdns-abschalter.patch
git apply patches/0003-umschalter-zu-alexa.patch
git apply patches/0004-setup-seite-einhaengungen.patch
```

0001 und 0002 greifen beide in `mdns.go`; 0001 enthält die Datei bereits in ihrem Endzustand, also
zuerst 0001, dann der Rest. `config.go` wird nur von 0002 berührt.

## Gegenprobe (19.11.2026)

Gegen einen **frisch geklonten** Quellbaum auf `dot-v1.0.72` geprüft, nicht gegen die
Arbeitskopie, in der sie entstanden sind:

| Schritt | Ergebnis |
|---|---|
| 0001 anwenden | passt |
| 0002 anwenden | passt |
| 0003 anwenden | passt |
| 0004 anwenden | passt |
| `go vet` über die drei berührten Pakete | sauber |
| `go build -tags dot` für linux/arm | **baut durch**, 32.309.408 Bytes |

**Die Patches allein genügen also.** Wer nur dieses Verzeichnis und das Ursprungsprojekt hat, kommt
ohne unsere Arbeitskopie zum selben Ergebnis — das ist die Bedingung dafür, dass sie dem
Ursprungsprojekt überhaupt angeboten werden könnten.

## Vor einer Weitergabe

Diese Patches sind weitergebbar — sie enthalten **nichts** von Amazon. Veröffentlicht wird
trotzdem erst auf Markus' Wort (Regel 2). Der Kernel-Patch 0049 des Ursprungsprojekts liegt
dagegen **nicht** hier: Er gehört dem Ursprungsprojekt und steht dort unter GPL-2.0.

---

## 0005 — Der Wächter reagiert schneller

| | |
|---|---|
| **Datei** | `0005-wifiwatch-schneller-reagieren.patch` |
| **Gegen** | TECHO5 `dot-v1.0.72` (Commit `2d021e42`) |
| **Berührt** | `echod/internal/feature/wifiwatch/multicast.go` und `broadcast.go` — **nur Zahlenwerte und Kommentare**, keine Logik |
| **Stand** | **am eigenen Gerät erprobt**, läuft seit 19.11.2026 19:52 als `v1.0.72_watch` |
| **Anlass** | Markus, 19.11.2026: „Bitte geh das Taubheits-problem an" |

**Was es tut.** Die Firmware hat bereits einen Wächter gegen den Funkfehler #45 — er erkennt, dass
keine Rundrufe mehr ankommen, meldet sich neu an und erneuert den Multicast-Filter. Im Protokoll
des eigenen Geräts steht, dass er zuverlässig funktioniert:

```
wifi: no broadcasts heard ... reassociating      quiet=3m30s
wifi: broadcasts are back after reassociating    after=30s
wifi: no multicast heard ... refreshing filter   quiet=10m0s
wifi: multicast is back after refreshing         after=30s
```

**Er reagiert nur zu spät.** Der Router des eigenen Netzes wechselt den Gruppenschlüssel alle
**587 Sekunden** (am Gerät an den EAPOL-Frames im Kernel-Protokoll abgelesen: Sekunde 20 und 607).
Der Multicast-Wächter wartete **zehn Minuten** Stille ab — er lief dem Fehler dauerhaft hinterher.
Und weil mDNS über Multicast läuft, war das Gerät damit die meiste Zeit unauffindbar.

| Konstante | vorher | jetzt | Begründung |
|---|---|---|---|
| `mcQuiet` | 10 min | **90 s** | Der Multicast-Refresh kostet **kein WLAN** (steht so im Quelltext). Ein Fehlalarm kostet eine Protokollzeile, ein spätes Eingreifen die Auffindbarkeit. |
| `mcFirstWait` | 10 min | **90 s** | dito |
| `mcMostWait` | 6 h | **20 min** | Sechs Stunden ist kein Zögern mehr, sondern Aufgeben |
| `bcMostWait` | 6 h | **30 min** | dito; drei Rekey-Abstände |
| `bcQuiet`, `bcFirstWait` | 3 / 5 min | **unverändert** | Eine Neuanmeldung kostet Sekunden WLAN — und dieser Teil funktionierte nachweislich schon |

**Gemessene Wirkung** (Messreihen in `privat\messungen\`, Auswertung mit `spikes\mdns_auswerten.py`):

| | vorher (425 Runden, 8,1 h) | **nachher (608 Runden, 10,6 h)** |
|---|---|---|
| erreichbar | 24,2 % | **68,8 %** |
| längste Phase | 27 min | **5 min** |
| Median einer Phase | 12 min | **1 min** |
| Rundrufe taub/erreichbar | 0,38 | **0,88** |
| **taube Minuten ganz ohne Empfang** | **174 von 322** | **0 von 189** |

Die Vorher-Reihe läuft über 516 Runden (8,6 h), die Nachher-Reihe über 608 Runden (10,6 h).

> **Berichtigt am 20.11.2026.** Hier standen zuvor 71,2 %, „längste Phase 4 Minuten" und
> „0 von 148" — Zahlen aus einer **Zwischenauswertung nach 513 Runden**, während die Messung noch
> lief. Sie wurde am 20.11. um 06:31 beendet und vollständig ausgewertet; oben stehen jetzt die
> endgültigen Werte. Die Aussage ändert sich dadurch nicht, die Zahlen schon.
>
> **Das ist derselbe Fehler wie am 19.11.**, als „70 %" aus einer Zwischenauswertung nach 124
> Runden in die Dokumente geriet. Daraus die Regel, die diesmal nicht befolgt wurde: **Eine
> laufende Messung liefert keine Zahl für ein Dokument.** Wer eine Zwischenzahl aufschreibt, muss
> sie als solche kennzeichnen und beim Abschluss ersetzen — sonst wandert sie weiter, hier bis in
> ein öffentliches Repository.

Die letzte Zeile ist die aussagekräftigste: Vorher brach der Empfang in den tauben Phasen ein
(Verhältnis 0,38), jetzt läuft er unverändert weiter (0,88). **Es gibt keine Funk-Taubheit mehr**;
die verbleibenden Lücken sind die 90 Sekunden, die der Wächter zum Reagieren braucht.

**Geprüft vor dem Einspielen:** `go vet` sauber · **die Tests des Wächters auf dem Gerät selbst
ausgeführt, alle bestanden** — darunter `a_quiet_network_with_no_router_adverts` und
`ANetworkHeardNowAndThenIsNotALoop`, also genau die Fälle, die bei zu kurzen Wartezeiten brechen
würden · **Kontrollbau gegen die unveränderte Quelle: 0 Bytes Unterschied** · Prüfsumme am PC und
am Gerät identisch · keine Fehlerzeile nach dem Start.

**Rückweg:** Der vorherige Stand liegt am Gerät unter `/data/rueckweg/echod.vor-watch`.

**Für andere Netze:** Der Patch macht **keine Annahme über das Netz** — er verkürzt nur
Wartezeiten eines vorhandenen Mechanismus. In einem ruhigen Netz ohne viel Multicast führt das zu
etwas häufigeren Filter-Erneuerungen, die nichts kosten; die Tests decken diesen Fall ab.

---

## 0006 — eigener Update-Kanal

**Auftrag Markus, 20.11.2026:** „Firmware update sollte zu uns umgeleitet werden und wir verteilen
updates mit unseren Patches."

**Zwei Zeilen**, und beide gehören zusammen:

| Datei | was |
|---|---|
| `internal/update/releases_dot.go` | die URL, von der das Manifest geholt wird |
| `internal/update/trust.go` | der öffentliche Schlüssel, gegen den dessen Signatur geprüft wird |

**Warum beide.** Die Firmware holt `manifest.json` und daneben `manifest.json.sig` — eine
ed25519-Signatur über genau die Bytes des Manifests. Ohne gültige Signatur geschieht nichts. Der
Quelltext begründet das selbst: „An update installs as root, so a manifest is believed only when
this key signed it." Nur die URL zu ändern, ergäbe ein Gerät, das zwar bei uns nachfragt, aber
jede Antwort zurückweist.

**Die Kanäle lassen sich nicht mischen.** Ein Gerät mit diesem Patch nimmt keine Manifeste des
Ursprungsprojekts mehr an und umgekehrt. Der Wechsel ist in beide Richtungen möglich, braucht
aber jedes Mal eine Installation von Hand.

**Geprüft:** `go vet` sauber · im fertigen Bau nachgesehen, dass die neue Adresse zweimal und der
neue Schlüssel einmal vorkommen und die alten **gar nicht** · Manifest signiert und die Signatur
mit dem öffentlichen Teil geprüft, samt Gegenprobe (ein um ein Bit verändertes Manifest muss
durchfallen — es fällt durch).

**Der private Schlüssel liegt in `privat\update-schluessel\release.key`** und nirgends sonst. Wer
ihn hat, kann auf jedem Gerät dieses Kanals Code als root ausführen; das ist der Grund, aus dem
die Prüfung überhaupt existiert.

**Rückweg:** Der vorherige Stand liegt am Gerät unter `/data/rueckweg/echod.vor-kanal1`. Ein Bau
ohne diesen Patch folgt wieder dem Ursprungsprojekt.

**Nicht eingebaut: automatische Installation.** Sie ist vorhanden (nachts 2–5 Uhr, nur bei Ruhe,
mit Rückfall über die A/B-Plätze), steht aber auf „aus" und bleibt es — so entschieden am
20.11.2026. Bei einem eigenen Kanal heißt das: Ein Fehler in unserem Bau erreicht kein Gerät
ungefragt.

---

## 0007 — jeder Abruf bekommt seine eigene Zeitgrenze

**Gefunden beim ersten Betrieb des eigenen Kanals (20.11.2026).** Die Update-Prüfung schlug
wiederholt fehl, zuletzt mit einer Meldung, die in die Irre führt:

    checking for an update failed  err="update: the manifest has no signature: ..."

Die Signatur war da. Sie lag auf dem Server, war vom Gerät aus **fünfmal von fünf** in unter einer
Sekunde abrufbar, und sie war gültig.

**Die Ursache:** `Fetch()` setzte **eine** Frist von zehn Sekunden und holte darin **zwei** Dateien
— erst das Manifest, dann die Signatur. Die Signatur bekam, was übrig blieb.

    ctx, cancel := context.WithTimeout(ctx, manifestTimeout)   // 10 s für beides
    body, _ := get(ctx, url, maxManifest)                      // nimmt sich, was es braucht
    sig,  _ := get(ctx, url+".sig", 1<<10)                     // bekommt den Rest

**Am Gerät gemessen**, acht Durchgänge beider Abrufe nacheinander, Dauer zusammen:

| Durchgang | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| Sekunden | 2 | **9** | 2 | 5 | **9** | **16** | 6 | 5 |

**Jeder einzelne Abruf gelang** — der Rückgabewert war durchweg 0. Drei von acht Durchgängen
passten nur nicht in zehn Sekunden zu zweit. Das ist keine Störung, sondern eine zu knappe Frist.

**Die Änderung:** `getBounded()` gibt jedem Abruf `manifestTimeout` für sich.

- **Der Normalfall bleibt unverändert.** Wer in zwei Sekunden fertig ist, merkt nichts.
- **Nur der schlechteste Fall wird länger** — und das ist genau der, der vorher scheiterte.
- Eine vorhandene Frist des Aufrufers gilt weiter: `context.WithTimeout` nimmt die kürzere.

**Warum das allgemein und nicht nur für unser Netz sinnvoll ist:** Bei einem geteilten Budget
verliert **immer** die Signatur, weil sie als zweites geholt wird. Das trifft jede langsame
Verbindung, und die Fehlermeldung zeigt dabei auf die falsche Stelle — sie klingt nach einem
kaputten Release, nicht nach einer abgelaufenen Uhr. Jemand, der das sieht, sucht am falschen Ende.

**Geprüft:** `go vet` sauber · `go test ./internal/update/` bestanden · im fertigen Programm
nachgesehen, dass Versionsmarke, Adresse und Schlüssel stimmen.

**Rückweg:** Patch nicht anwenden; der vorherige Stand liegt am Gerät unter
`/data/rueckweg/echod.vor-kanal1`.

---

# `dot-bau\` — Patches gegen die Bauwerkzeuge (techo5-dot, Tag `dot-v1.0.72`)

Seit 20.11.2026 (Paket 2, `ablauf-protokoll.md` S28). Angewendet mit `git am` in der
Reihenfolge der Nummern; `spikes\repro_bau.py` tut das selbst.

| Patch | Was |
|---|---|
| `0001-mkrootfs-…` | **Gleiche Eingaben, gleiche Bytes:** `SOURCE_DATE_EPOCH` für alle Zeitstempel im Archiv und im gzip-Kopf, Overlay in Namensreihenfolge (`os.walk` folgt sonst dem Dateisystem), kein Dateiname im gzip-Kopf |
| `0002-packages-…` | Paketlisten auf die Fassungen im eigenen Lager (Alpine bietet acht der genannten nicht mehr an) |
| `0003-build-dot-rootfs-…` | `--extra SRC=DEST` für eigene Programme (das Overlay würde sie mit 0644 anlegen) |
| `0004-rootfs-…` | `boot.sh` (Lichtsignal + `seedctl`) und `techo5-net` (Kanal nur für die erste Suche) — die Fassungen, die am Gerät laufen |

**Geprüft, je mit Gegenprobe:** zwei Bauten bitgleich; **ohne 0001 nicht** bitgleich; ein
verändertes Byte im Lager bricht den Bau ab; ohne Lager (`GOPROXY=off`) baut `echod` nicht.

---

## 0008 — Home Assistant: der Weg ohne Tippen zuerst, der Schlüssel auf Klick

| | |
|---|---|
| **Berührt** | `internal/feature/setup/keyshow.go` (neu), `keyshow_test.go` (neu), `page.go`, `tabs.go` |
| **Anlass** | Markus, 20.11.2026: „können wir den nutzern den verschlüsselungscode in der weboberfläche anzeigen und im tool dazu anleiten?" (`docs\offene-punkte-home-assistant.md`, Punkt 3) |
| **Stand** | gebaut als `v1.0.72_kanal3`, Tests **am Gerät** bestanden; am Gerät noch nicht eingespielt |

1. **Der Weg ohne Tippen steht vorn.** Die Karte **Network → Home Assistant** beginnt mit
   „Home Assistant asks for an encryption key? You do not need to type one." Die Kurzzeile der
   Karte sagt ohne Verbindung „Not connected — can be added without typing a key".
2. **Der Schlüssel auf Klick** („Show the key to type instead"), mit Kopierknopf und QR-Code — so
   wie der Erststart des Ursprungsprojekts ihn schon immer zeigt (`firstrun_page.go`), mit zwei
   Auflagen mehr, weil die Einstellungsseite länger offen ist und der Schlüssel jede Sitzung
   überdauert: **nur auf POST mit dem Token der Seite** (nie im Seitenaufbau, nicht im Verlauf,
   `Cache-Control: no-store`) und **nur, solange kein Home Assistant verbunden ist**. Das Protokoll
   vermerkt, *dass* er gezeigt wurde, nie *was*.
3. Der Satz „never shows the Home Assistant key" im Abschnitt **This page** ist ehrlich ersetzt.
4. **Nebenbei behoben, ein Fehler aus 0002/0003:** Die Formulare `discovery` und `toalexa`
   fehlten in der Reiter-Zuordnung (`tabs.go`); nach dem Speichern sprang die Seite auf *Status*
   statt zurück. Gefunden vom vorhandenen Test `TestEveryFormIsDrawnOnTheTabItComesBackTo`.

**Abwägung, mit dem Berater (Fable 5.2) und nachgeprüft:** Der Schlüssel ist mehr als eine
Sitzung — er gilt dauerhaft, ohne Tastendruck, und über Home Assistants Schnittstelle lässt sich
damit SSH schalten, was die Seite selbst bewusst nicht kann (`page.go`, `switchedOnSection`).
Gegen einen Mithörer im eigenen WLAN ist die Anzeige gleichwertig mit dem Weg ohne Tippen (auch
dort geht der neue Schlüssel im Klartext übers Netz); dieser hält dafür 15 Minuten lang die Tür
für jeden offen. Deshalb beide Wege, der ohne Tippen zuerst, und die Anzeige eng begrenzt.

**Geprüft:** `go vet` (linux/arm) sauber · neue Tests am Gerät bestanden: Schlüssel nur auf POST
mit Token und ohne Verbindung; abgewiesen ohne Einlass, per GET, mit falschem Token und bei
verbundenem Home Assistant; **die Einstellungsseite enthält den Schlüssel in keinem Zustand** ·
frischer Klon + 0001–0008 **bitgleich** mit dem Bau aus der Arbeitskopie (`0d5c035c…`).
Zwei Tests des Pakets scheitern am Gerät aus Umgebungsgründen (lesen Quelltext von der Platte;
zählen die zwölf Aufweckwort-Modelle des Geräts) — nicht durch diesen Patch.

**Rückweg:** Patch nicht anwenden; am Gerät `/data/rueckweg/echod.vor-kanal3`.

## 0009 — der WLAN-Wächter liest das Kernel-Protokoll, ohne die Mikrofone zu stören

**Datei:** `0009-wifi-kernelzeilen-aus-kmsg.patch` · **gegen:** TECHO5 `2d021e42` + 0001–0008 ·
**Bau:** `v1.0.72_kanal4`, sha256 `a02c4992…` · Befund und Messungen: `docs\ablauf-protokoll.md`, S33,
Teil C.

**Das Problem.** Vor jedem Eingriff zeichnet der Wächter einen Beleg auf (`wifi.Evidence`), und der
rief `dmesg`. `dmesg` liest auf dem 3.18-Kernel des Dot den ganzen Ringpuffer (1 MB) mit gesperrten
Interrupts. Der SPI-Leser der Mikrofone kommt so lange nicht dran, und der Puffer im FPGA läuft
über: rund 16 ms Ton weg, bei **jedem** `dmesg` (am Gerät 10 von 10, Gegenprobe 0 von 10). Seit
Patch 0005 greift der Wächter in tauben Phasen etwa alle zehn Minuten ein — über 15 h waren es
25 Overruns.

**Die Änderung.** `Evidence` liest `/dev/kmsg` Satz für Satz (die Sperre gilt dort je Satz) und
bringt die Sätze in die Form von `dmesg`; die Auswahl der Zeilen bleibt gleich. Wo `/dev/kmsg`
fehlt, bleibt `dmesg`. Gelesen wird mit `syscall.Open`/`syscall.Read`, nicht mit `os.OpenFile`:
sonst übernimmt Go den Nicht-blockieren-Modus in seinen Netpoller, und das Lesen wartet am Ende
auf den nächsten Satz, statt aufzuhören.

| Datei | Inhalt |
|---|---|
| `internal/lib/wifi/evidence.go` | `/dev/kmsg` zuerst, `kmsgLine` |
| `internal/lib/wifi/kmsg_linux.go` | der Leser |
| `internal/lib/wifi/kmsg_other.go` | anderswo: kein `/dev/kmsg`, Rückfall auf `dmesg` |
| `internal/lib/wifi/kmsg_test.go` | Form der Sätze, Ablehnung kaputter Sätze, Namen bleiben geschwärzt |
| `internal/lib/wifi/kmsg_dot_test.go` | nur auf dem Gerät: echtes `/dev/kmsg`, ganz, schnell, ohne Warten |

**Geprüft:** Tests am PC und am Gerät bestanden · `go vet` (linux/arm, `dot`) sauber · am Gerät
10× gelesen, ~15.680 Zeilen in 0,19–0,25 s, **0 Overruns** · zweimal gebaut, **bitgleich** ·
`echod` getauscht, 0 `ERROR`. **Noch offen:** ein echter Eingriff des Wächters ohne Overrun (die
nächste taube Phase).

**Nicht verändert:** `diag/bundle.go` ruft für das Diagnose-Bündel weiter `dmesg` — selten und nur
auf Wunsch des Nutzers.

**Rückweg:** Patch nicht anwenden; am Gerät `/data/rueckweg/echod.vor-kanal4` (`kanal3`).

**Für das Ursprungsprojekt:** Der Aufruf steckt dort unverändert (`evidence.go`). Gemessen ist die
Wirkung nur an unserem einen Dot 2; dass andere Geräte mit demselben Kernel genauso betroffen
sind, ist **[ANNAHME]**. Den Patch weiterzugeben ist eine Veröffentlichung → Markus.
