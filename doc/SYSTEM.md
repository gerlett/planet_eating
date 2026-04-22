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
