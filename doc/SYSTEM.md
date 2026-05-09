# Planet Eating – Systemdokumentation

## Übersicht

Planet Eating ist ein browserbasiertes 2D-Arcade-Spiel. Der Spieler steuert einen Planeten durch ein großes Weltall-Level, frisst Asteroiden und kleinere Gegner, und muss die **Blaue Wolke** am rechten Rand der Welt erreichen, bevor die Giftwolke alles verschlingt.

---

## Spielfeld

- Die Welt ist `3 × Fensterbreite` mal `2 × Fensterhöhe` groß.
- Die Kamera folgt dem Spieler und bleibt innerhalb der Weltgrenzen.
- Hintergrund: Ein Weltraum-Bild (`images/`). Fehlt das Bild, wird ein dunkler Hintergrund (`#000011`) gezeichnet.
- Die Weltgrenzen werden als weißer Rahmen dargestellt.

---

## Spieler

| Eigenschaft | Wert |
|---|---|
| Startposition | 10 % vom linken Rand, vertikal mittig |
| Startmasse | 100 |
| Radius | `sqrt(masse) * 2` |
| Basisgeschwindigkeit | 3 (skaliert mit Masse, s. u.) |
| Darstellung | Cyanfarbener Kreis, Label „Ich" |

### Steuerung

Pfeiltasten (`↑ ↓ ← →`). Bei gleichzeitiger Rechtsbewegung gibt es einen leichten Rechtsbonus (+0,4 auf X).

Die effektive Geschwindigkeit nimmt mit steigender Masse ab:

$$v_{eff} = \max\!\left(0.8,\ \frac{v_{base}}{\left(\frac{m}{100}\right)^{0.7}}\right) \times 1.25$$

---

## Masse & Radius

Jede Spielentität (Spieler, Bot, Asteroid) hat eine Masse. Der Radius ergibt sich immer als:

$$r = \sqrt{m} \times 2$$

---

## Asteroiden

- Werden jede Sekunde in 3er-Schüben zufällig in der Welt gespawnt.
- Maximale Masse eines Asteroiden: `min(asteroidSizeLimit, spielerMasse - 1)`. Das Limit folgt der **niedrigsten** bisher erreichten Spielermasse (damit man nicht von riesigen Asteroiden überwältigt werden kann, wenn man kleiner wird).
- Mindestmasse: 10.
- Asteroiden prallen an Weltgrenzen ab (Velocity-Bounce).
- Visuelle Darstellung: Unregelmäßige Polygone mit Radialverlauf und Kratern.
- Aufsammeln: Spieler bekommt `asteroidMasse × 0,4` Masse. Bots erhalten `asteroidMasse × 0,4 × 0,75`.

---

## Gegner-Bots

- **Anzahl**: 20 beim Start.
- **Startmasse**: 100 (wie Spieler).
- **Namen**: Zufällig aus einem Pool von 50 Weltraum-Namen.
- **Farben**: 10 unterschiedliche Farbschemata, zyklisch vergeben.

### Bot-KI (Prioritätsreihenfolge)

1. **Giftwolke fliehen** – höchste Priorität; wenn der Bot in der Gefahrenzone ist, bewegt er sich in Richtung sicherer Zone.
2. **Laufende Launch-Bewegung beenden** – nach einem gleichmassigen Aufprall wird der Bot zu einem sicheren Ort katapultiert.
3. **Fliehen** – vor dem Spieler oder anderen Bots, die eindeutig schwerer sind (Radius 350 px).
4. **Jagen** – den nächsten kleineren Feind (Spieler oder Bot) mit dem besten Masse/Distanz-Score.
5. **Asteroiden sammeln** – nächsten sammelbaren Asteroiden innerhalb 1200 px ansteuern.
6. **Wandern** – zu einem zufälligen Zielpunkt im sicheren Bereich.

Bewegung wird sanft interpoliert (Steer-Faktor 0,22), um abrupte Richtungswechsel zu vermeiden.

---

## Kampfsystem

Wenn zwei Planeten kollidieren:

| Bedingung | Ergebnis |
|---|---|
| Gleiche Masse (Verhältnis = 1) | Beide werden zu sicheren Positionen katapultiert + 3 s Schild |
| Spieler größer als Bot | Bot wird absorbiert; Spieler gewinnt `botMasse × 0,8` |
| Bot größer als Spieler | Spieler stirbt (`gameOver`); Bot gewinnt `spielerMasse × 0,8` |
| Bot größer als anderer Bot | Größerer absorbiert kleineren, gewinnt `massdes Kleineren × 0,8` |

### Schild

Nach einem Gleichstand-Aufprall sind beide Planeten für **3 Sekunden** unverwundbar (kein Schaden durch Kampf oder Giftwolke). Dargestellt als pulsierender gelber Ring.

---

## Giftwolke (Poison Cloud)

Die Giftwolke wächst über **300 Sekunden** von allen vier Rändern nach innen. Die sichere Zone beginnt mit einem Rand-Inset von 38 px und schrumpft bis auf `max(Weltbreite, Welthöhe) / 2`.

- **Schadenstakt**: 1 Sekunde.
- **Schaden pro Tick**: 25 % der Masse beim Eintreten in die Giftwolke.
- Nach 4 Ticks → Tod.
- Visuelle Warnung im HUD: gelb (nahe), orange (WARNUNG), rot (KRITISCH).
- Asteroiden, die in der Giftwolke landen, werden sofort zerstört.
- Bots haben eigene Poison-Timer und werden ebenfalls nach 4 Ticks getötet.
- Bei aktivem Schild wird kein Giftschaden genommen.

---

## Siegbedingung & Spielende

| Ereignis | Ergebnis |
|---|---|
| Spieler berührt die Blaue Wolke | **Sieg** – `endMatch('Ziel erreicht')` |
| Spieler stirbt (Kampf oder Gift) | **Niederlage** – `gameOver` |
| Alle Bots zerstört | **Rundenende** – `endMatch('Alle Bots zerstoert')` |

Nach dem Rundenende wird ein Overlay mit Titel, Ergebnis und dem Massenführer angezeigt.

### Blaue Wolke

- Position: rechter Rand der Welt, vertikal mittig (`world.width - 180`, `world.height / 2`).
- Radius: 130 px.
- Trefferzone: Spielerradius + `endCloud.radius × 0,55`.

---

## HUD (Benutzeroberfläche)

Oben links, halbtransparentes Panel:

| Anzeige | Beschreibung |
|---|---|
| **Masse** | Aktuelle gerundete Spielermasse |
| **Position** | X/Y-Koordinaten in der Welt |
| **Feind-Bots** | Anzahl verbleibender Bots |
| **Warnung** | Hazard-Status (Sicher / Nähe Giftwolke / WARNUNG / KRITISCH / ALARM: Feind-Bot / SCHILD) |
| **Start** | Startet die Runde (verschwindet nach Klick) |
| **Nochmal** | Setzt das Spiel zurück und startet sofort neu |

---

## Explosionen & Effekte

| Typ | Farbe | Auslöser |
|---|---|---|
| `combat` | Rot-Orange | Kampftod |
| `poison` | Grün | Gifttod |
| `poison` (Schaden) | Hellgrün, Partikel | Gifttick-Schaden |
| generisch | Orange | Asteroid aufgesammelt |

---

## Spielzustände

```
Initialisierung
    └─► resetGame() → Startscreen anzeigen
            └─► Start-Klick → gameStarted = true → gameLoop aktiv
                    ├─► gameWon = true  → matchEnded
                    ├─► gameOver = true → matchEnded
                    └─► bots.length = 0 → matchEnded
                              └─► Nochmal-Klick → resetGame() + startGame()
```

---

## Technische Details

- Reine HTML5-Canvas-Anwendung, kein Framework.
- Animationsloop via `requestAnimationFrame`.
- Delta-Zeit begrenzt auf maximal 50 ms pro Frame (Anti-Spiral-of-Death).
- Welt passt sich bei `resize` an die neue Fenstergröße an.
- Asteroiden-Spawn-Intervall: 1000 ms (3 pro Tick via `setInterval`).

---

## Planeten – Beschreibung, Attacke & Gadget

Jeder Planet hat eine eigene Rolle, eine Attacke und ein Gadget. Das Gadget wird in der Startscreen-UI über dem Planeten-Bild angezeigt und wechselt automatisch bei der Planet-Auswahl.

---

### 🔥 Lava

**Rolle:** Offensiv  
**Beschreibung:** Ein Vulkanplanet mit lodernden Ausbrüchen, der Gegner mit brachialem Druck überrollt.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Magma-Stoss | Trifft Gegner vorne und bremst sie kurz. |
| **Gadget** | Magma-Anker | Setzt ein heisses Gravitationsfeld – nahe Planeten werden langsam zur Mitte gezogen. Rausfliegen kostet mehr Kraft. |

**Gadget-Typ:** Gravity Control / Zone Lock  
**Visual:** Glühender roter Kern, flackernde Hitzewellen, leichte Raumverzerrung.  
**Mechanik:** Reduziert Bewegungsgeschwindigkeit im Radius, verhindert schnelles Verlassen der Zone, erhöht Energieverbrauch beim Bewegen gegen die Gravitation.

---

### ❄️ Eis

**Rolle:** Kontrolle  
**Beschreibung:** Eine gefrorene Kugel mit splitterndem Eis, langsamem Kern und kräftigem Gegendruck.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Frost-Stoss | Schiebt Gegner weg und verlangsamt sie kurz. |
| **Gadget** | Kryo-Spiegel | Spiegelt die Bewegungsrichtung des ersten Objekts um, das hineinläuft. Kein Schaden, nur Umleitung. |

**Gadget-Typ:** Reflection / Movement Manipulation  
**Visual:** Eisblaue reflektierende Oberfläche, Lichtbrechung wie in Kristall, kurze Frost-Ausbreitung beim Trigger.  
**Mechanik:** Erster Bewegungsimpuls wird umgekehrt, keine Schadensanwendung, kann Positionen unvorhersehbar ändern.

---

### ⚡ Sturm

**Rolle:** Tempo  
**Beschreibung:** Ein geladener Sturmplanet, dessen Blitze bei jeder Bewegung am Mantel entlangzucken.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Blitz-Impuls | Schickt einen Blitz nach vorne, der kurz ablenkt. |
| **Gadget** | Circuit-Sprung | Springt entlang der Bewegungslinien anderer Planeten. Extrem schnelle Positionswechsel möglich. |

**Gadget-Typ:** Teleport / Momentum Chain  
**Visual:** Elektrische Linien verbinden Objekte, Blitz-Impulse zwischen Punkten, kurze Afterimages beim Teleport.  
**Mechanik:** Erlaubt Bewegung entlang bestehender Bewegungsstraßen anderer Planeten, kein direkter Schaden, extrem hohe Mobilität.

---

### 🌿 Natur

**Rolle:** Kontrolle  
**Beschreibung:** Ein organischer Planet, der Gegner durch Wurzeln bindet und synchronisiert.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Wurzel-Peitsche | Schlägt mit einer Wurzel zu und hält den Gegner kurz fest. |
| **Gadget** | Wurzelnetz | Verbindet zwei Planeten durch ein biologisches Wurzelsystem – beide teilen dieselbe Bewegungsrichtung. |

**Gadget-Typ:** Binding / Sync Movement  
**Visual:** Grüne leuchtende Wurzeln im Raum, organische Verbindungslinien, leichte Pulsation wie Herzschlag.  
**Mechanik:** Verbundene Planeten teilen Bewegungsrichtung, kann Bewegung einschränken oder synchronisieren, stabilisiert Positionen.

---

### ☢️ Radioaktiv

**Rolle:** Chaos  
**Beschreibung:** Ein instabiler Planet, der physikalische Regeln kurz außer Kraft setzt.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Strahlen-Burst | Feuert einen radioaktiven Strahl, der die Kontrolle kurz stört. |
| **Gadget** | Mutations-Feld | Ändert eine physikalische Regel kurz im Bereich – Gravitation oder Geschwindigkeit können sich zufällig umkehren. |

**Gadget-Typ:** Random Physics Modifier  
**Visual:** Flackernde grüne/lila Partikel, glitchartige Verzerrung, instabile Raumanimation.  
**Mechanik:** Gravitation kann invertiert werden, Geschwindigkeit zufällig verändert, Effekte können sich selbst überschreiben.

---

### 🌑 Schatten

**Rolle:** Täuschung  
**Beschreibung:** Ein dunkler Planet, der Positionen tauscht und Gegner desorientiert.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Schatten-Schlag | Greift aus dem Nichts an und bricht die Zielausrichtung. |
| **Gadget** | Phasentausch | Tauscht die Position mit einem anderen Objekt aus – ohne Bewegung, nur Teleport. |

**Gadget-Typ:** Position Swap / Desync  
**Visual:** Dunkle Rauchspur zwischen Punkten, kurze Glitch-Teleport-Animation, Schatten-Nachbilder.  
**Mechanik:** Kein Bewegen nötig, nur Positionsaustausch, kann Gegner desorientieren, bricht Zielausrichtung.

---

### 🪨 Stein

**Rolle:** Defense  
**Beschreibung:** Ein massiver Planet, der sich durch rotierende Felsbrocken schützt.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Fels-Ramme | Rammt mit voller Wucht und schiebt alles weg. |
| **Gadget** | Orbit-Schild | Lässt Steinfragmente um den Planeten kreisen – schützt vor Push, Pull und Positionsveränderungen. |

**Gadget-Typ:** Defense / Barrier Physics  
**Visual:** Rotierende Felsbrocken, stabile kreisförmige Struktur, schwere langsame Rotation.  
**Mechanik:** Blockiert Bewegungen und externe Kräfte, schützt vor Push/Pull-Effekten, stabilisiert Position im Raum.

---

### 🌪️ Wind

**Rolle:** Positionierung  
**Beschreibung:** Ein Planet, der gerichtete Luftströme erzeugt und Gegner zieht oder stößt.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Windstoss | Pusht Gegner weg mit einem gezielten Luftstoss. |
| **Gadget** | Wind-Griff | Erzeugt Luftströme, die Planeten ziehen oder schieben können – flexibel für Angriff und Positionierung. |

**Gadget-Typ:** Force Push/Pull Control  
**Visual:** Sichtbare Luftlinien, swirlende Windwirbel, leichte Objektverzerrung.  
**Mechanik:** Kann Planeten bewegen ohne Schaden, kontrollierte Push/Pull-Kräfte, sehr flexibel im Positioning.

---

### 🌊 Wasser

**Rolle:** Schutz & Mobilität  
**Beschreibung:** Ein Planet, der sich in einer schützenden Blase fortbewegt.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Wellen-Schlag | Trifft mit einer Welle und verlangsamt kurz. |
| **Gadget** | Strömungsblase | Hüllt den Planeten in eine Wasserblase – schützt vor äusseren Kräften und trägt ihn glatt in eine Richtung. |

**Gadget-Typ:** Auto-Movement / Protection  
**Visual:** Transparente Wasserblase, sanfte Wellenbewegung, Lichtbrechung im Inneren.  
**Mechanik:** Schützt vor externen Kräften, bewegt Objekt automatisch, glatte kontrollierte Bewegung.

---

### 👽 Alien

**Rolle:** Kontrolle  
**Beschreibung:** Ein fremder Planet, der kurzzeitig die Steuerung anderer Planeten übernimmt.

| | Name | Beschreibung |
|---|---|---|
| **Attacke** | Psi-Impuls | Sendet einen mentalen Impuls, der die Steuerung des Gegners kurz stört. |
| **Gadget** | Geistfeld | Übernimmt kurzzeitig die Bewegungssteuerung eines anderen Planeten – kein Schaden, maximaler taktischer Wert. |

**Gadget-Typ:** Temporary Control / Override  
**Visual:** Glitchender HUD-Effekt, mentale Wellen im Raum, fremdartige Symbolik.  
**Mechanik:** Kontrolliert Bewegung eines Zielobjekts, keine direkten Schäden, extrem hoher taktischer Wert.
