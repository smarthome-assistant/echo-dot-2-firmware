# Echo Dot 2 als lokaler Sprachassistent — eigene Bauten

Dieses Repository enthält **Änderungen an der freien Firmware [TECHO5 Dot](https://github.com/HuskerMinion/techo5-dot)** für den Amazon Echo Dot der 2. Generation (Modell RS03QR, Entwicklungsname `biscuit`), dazu einen eigenen Update-Kanal für fertige Bauten.

Das Ziel ist ein **Sprach-Satellit für Home Assistant** auf vorhandener Hardware: Mikrofone, Lautsprecher, Lichtring und Tasten arbeiten weiter, nur ohne Amazons Cloud.

> **Dies ist nicht das Ursprungsprojekt.** Die eigentliche Arbeit steckt in TECHO5 Dot. Hier liegen nur Änderungen obendrauf, jede als eigener Patch und jede mit dem Messbefund, aus dem sie entstanden ist. Wer die Firmware sucht, ist [dort](https://github.com/HuskerMinion/techo5-dot) richtig.

---

## Was hier anders ist

| Patch | Was er ändert | Belegt durch |
|---|---|---|
| `0001` | Das Gerät kündigt sich alle drei Minuten selbst im Netz an | Takt am Gerät gemessen: 182,4 s |
| `0002` | Diese Ankündigung lässt sich abschalten | — |
| `0003` | Umschalter zurück zu Fire OS, über die Einrichtungsseite | am Gerät erprobt |
| `0004` | Die Einhängungen dafür in der Einrichtungsseite | — |
| `0005` | **Der WLAN-Wächter reagiert schneller** | zwei Messreihen à ~9 h |

### Der wichtigste: `0005`

Betroffene Geräte verlieren nach einem Gruppenschlüssel-Wechsel des Routers zeitweise den Empfang für Rundrufe — der Chip verwirft sie, unterhalb des Treibers. Das Gerät bleibt erreichbar, ist aber für mDNS, AirPlay und die Geräteerkennung unsichtbar.

**TECHO5 hat dagegen längst einen Wächter.** Er erkennt den Verlust und repariert ihn zuverlässig — er wartete nur zu lange, bevor er hinsah: zehn Minuten Stille, bei einem Router, der alle **587 Sekunden** neu verschlüsselt. Er lief dem Fehler dauerhaft hinterher.

Geändert wurden **nur Zahlenwerte**:

```
mcQuiet / mcFirstWait   10 min  →  90 s
mcMostWait               6 h    →  20 min
bcMostWait               6 h    →  30 min
```

**Gemessen an einem Gerät, zwei Reihen über je rund neun Stunden:**

| | vorher | nachher |
|---|---|---|
| im Netz auffindbar | 24,2 % der Zeit | **71,2 %** |
| längste blinde Phase | 27 Minuten | **4 Minuten** |
| Minuten ganz ohne Empfang | 174 von 322 | **0 von 148** |

Die letzte Zeile ist die aussagekräftigste: Der Empfang bricht nicht mehr ein. Was bleibt, ist die Reaktionszeit des Wächters — eine Wartezeit, kein Fehler.

**Das gilt für ein Gerät an einem Router.** Wie oft ein anderer Router neu verschlüsselt, ist nicht gemessen; bei einem sehr ruhigen Netz kostet der kürzere Takt eine Protokollzeile hin und wieder, sonst nichts.

---

## Woher die Zahlen kommen

Jede Angabe hier ist an einem Gerät gemessen, nicht geschätzt. Wo etwas nur gelesen und nicht erprobt ist, steht es dabei. Das gilt besonders für eine Sache, bei der die Dokumentation des Ursprungsprojekts an unserem Gerät **nicht zutraf**:

> `docs/hardware.md` nennt „Vol-Up → recovery, Vol-Down → fastboot".
> An unserem Gerät stimmte nur die zweite Hälfte beim ersten Versuch. **Lauter** startete zweimal Fire OS — die Taste wirkt, aber sie muss **rund zehn Sekunden** gehalten werden.

Wer eine Anleitung schreibt, die nicht funktioniert, lässt den Leser glauben, er habe etwas falsch gemacht. Deshalb steht es hier so.

---

## Eigener Update-Kanal

Bauten aus diesem Repository tragen einen **eigenen Signaturschlüssel**. Das Gerät prüft jedes Manifest dagegen, bevor es irgendetwas lädt — dieselbe Mechanik wie im Ursprungsprojekt, nur mit anderem Schlüssel.

**Öffentlicher Schlüssel dieses Kanals:**

```
eJX5OR/aTmvmU15gs0fEn0nbluq7bvAd4REPeJcY1EU=
```

**Was das bedeutet:** Ein Gerät mit diesen Bauten nimmt **keine** Updates des Ursprungsprojekts mehr an, und umgekehrt. Der Wechsel ist in beide Richtungen möglich, aber er braucht jedes Mal eine Installation von Hand.

Updates installieren sich **nicht von selbst**, solange das nicht ausdrücklich eingeschaltet wird.

---

## Installation

Die Patches werden gegen den Quelltext von TECHO5 Dot angewendet und selbst gebaut:

```sh
git clone https://github.com/HuskerMinion/techo5-dot
cd techo5-dot
git am ../patches/00*.patch
cd echod
GOOS=linux GOARCH=arm GOARM=7 CGO_ENABLED=0 go build -tags dot -trimpath
```

Die Reihenfolge und die Besonderheiten stehen in [`patches/LIESMICH.md`](patches/LIESMICH.md).

> **Vor einem eigenen Bau:** erst einen **Kontrollbau aus unverändertem Quelltext** machen und mit dem vergleichen, was auf dem Gerät läuft. Stimmt der nicht, ist auch der geänderte Bau nicht zu beurteilen. Bei uns unterschied sich der Kontrollbau in 60 von 32.243.872 Bytes — ausschließlich in der Go-Build-ID.

> **Versionsmarken mit Unterstrich**, nie mit `+`. Home Assistant kann ein `+` nicht einordnen und bietet dann endlos ein Update an, das nie fertig wird. Richtig: `v1.0.72_mdns2`.

---

## Wichtige Hinweise

**Das Gerät kann dabei unbrauchbar werden.** Das Entsperren des Bootloaders ist der Schritt ohne Rückweg. Wer das macht, sollte vorher ein vollständiges Abbild des Originalzustands ziehen — es enthält Amazons Software und gehört deshalb nirgendwohin außer auf die eigene Festplatte.

**Hier liegt nichts von Amazon.** Keine Firmware, keine Abbilder, keine Teile davon. Die Patches enthalten ausschließlich eigenen Quelltext; die Bauten entstehen aus den Quellen von TECHO5 Dot und freien Go-Bibliotheken.

**Amazon, Echo, Echo Dot, Alexa und Fire OS sind Marken von Amazon.** Sie werden hier genannt, um das Gerät zu bezeichnen, um das es geht. Dieses Projekt steht in keiner Verbindung zu Amazon und wird von dort weder unterstützt noch geprüft.

---

## Lizenz

[MIT](LICENSE), wie das Ursprungsprojekt — der Copyright-Hinweis von *TECHO5 contributors* bleibt erhalten.

Die Bauten enthalten freie Bibliotheken Dritter; ihre Lizenztexte stehen vollständig in [`NOTICE.txt`](NOTICE.txt): 24× MIT, 17× BSD-3-Clause, 7× Apache-2.0, 3× Unlicense, 1× MPL-2.0. Keine mit Copyleft-Pflicht für fremden Code.

---

## Dank

An **[HuskerMinion](https://github.com/HuskerMinion)** und alle, die an TECHO5 mitgearbeitet haben. Dieses Repository wäre ohne diese Arbeit leer.
