# Änderungen seit dem letzten Pull

Vergleich des aktuellen Stands von `Earth_and_moon_simulation.ipynb` gegen den letzten
Commit (`7426c3a – Merge pull request #3 ... theia_physik`). Verglichen wurde nur der
**Quellcode** der Zellen (die riesigen Bild-Outputs wurden ausgeklammert).

Struktur des Notebooks: **12 → 15 Zellen** (3 neue Zellen).

---

## 1. Überblick

| Bereich | Art der Änderung |
|---|---|
| Klasse `Simulation` | **Physik/Logik geändert** – synchrone History mit NaN-Auffüllung, Farb-/Radius-Tracking, Mischfarbe bei Verschmelzung |
| Klasse `Visualizer` | **Stark erweitert** – Performance-Mode, dynamischer Zoom, proportionale Punktgrößen, Geschwindigkeitsregler, robustere Skalierung |
| Klasse `Body` | **unverändert** |
| Klasse `ScenarioController` | **unverändert** |
| Test-/Auswertungszellen | Alte Einzel-Testzelle ersetzt durch **systematische 3×3-Szenario-Matrix** + Erklärungstexte + 2 Animations-Zellen |

Die in der Aufgabenstellung als „Visualisierung" gedachten Änderungen haben tatsächlich
**kleine, aber notwendige Eingriffe in die `Simulation`-Klasse** erfordert (siehe unten),
weil die Visualisierung saubere, gleich lange Trajektorien und Metadaten (Farbe, Radius)
braucht, die nach einer Verschmelzung sonst verloren gingen.

---

## 2. Änderungen an der Klasse `Simulation` (nicht-Visualisierung)

### 2.1 Neue Zustandsfelder im Konstruktor
```python
self.colors = {}          # Farbe jedes je hinzugefügten Körpers
self.radii  = {}          # Radius jedes je hinzugefügten Körpers
self.recorded_steps = 0   # Anzahl bereits gespeicherter Zeitschritte
```
**Warum:** Nach einer Verschmelzung verschwindet der ursprüngliche `Body` aus
`self.bodies`. Farbe und Radius wären dann für die Visualisierung nicht mehr abrufbar.
Jetzt werden sie dauerhaft pro Name gespeichert.

### 2.2 `add_body()` – History wird zeitlich synchronisiert
```python
self.colors[body.name] = body.color
self.radii[body.name]  = body.radius
self.history[body.name] = [np.full(3, np.nan) for _ in range(self.recorded_steps)]
```
**Warum:** Ein Körper, der mitten im Lauf entsteht (z. B. „Erde+Theia"), bekommt für alle
bereits vergangenen Zeitschritte `NaN`-Positionen. Dadurch sind **alle** Trajektorien
exakt gleich lang → die Animation kann nicht mehr über das Ende eines Arrays hinaus
indexieren.

### 2.3 Neue Methode `_blend_colors()`
Mischt zwei Farben 50/50 zu einer neuen RGB-Farbe (z. B. blau + rot → violett), damit der
verschmolzene Körper optisch von beiden Ausgangskörpern unterscheidbar ist.

### 2.4 `merge_bodies()` – neuer Körper bekommt Mischfarbe
Vorher wurde der verschmolzene Körper mit leerer Farbe `""` erzeugt:
```python
# vorher
newBody = Body(..., "")
# nachher
newColor = self._blend_colors(body1.color, body2.color)
newBody  = Body(..., newColor)
```

### 2.5 `step()` – History für ALLE registrierten Körper, NaN für verschmolzene
```python
alive = {body.name: body for body in self.bodies}
for name in self.history:
    if name in alive:
        self.history[name].append(alive[name].position.copy())
    else:
        self.history[name].append(np.full(3, np.nan))
self.recorded_steps += 1
```
**Vorher** wurde die Position nur innerhalb der `bodies`-Schleife gespeichert. Nach einer
Verschmelzung wuchsen die Trajektorien dadurch **unterschiedlich lang** – das war die in
Commit `896dc6c` notierte „Fehler in der Merge-Simulation".
**→ Dieser Bug ist mit der Änderung behoben** (per Smoke-Test verifiziert: alle
Trajektorien gleich lang, NaN-Auffüllung korrekt).

---

## 3. Änderungen an der Klasse `Visualizer`

`animate_3d()` hat 5 neue Parameter:
`performance_mode`, `max_frames`, `dynamic_scaling`, `figsize`, `speed`.

- **`performance_mode` / `max_frames`:** Rendert bei sehr vielen Schritten (kleiner
  Zeitschritt → tausende Frames) nur jeden `stride`-ten Frame (max. ~150). Die Physik
  bleibt vollständig, da `update()` per Index die komplette Historie bis zum Frame zeichnet.
- **`dynamic_scaling`:** Mitlaufender, würfelförmiger Zoom auf die aktuell lebenden Körper.
  Wichtig, weil Theia aus 4 Mio. km anfliegt, Erde/Mond aber nur ~380.000 km auseinander
  liegen – statisch würde das Erde-Mond-System zu einem Punkt schrumpfen.
- **`speed`:** Abspielgeschwindigkeit über `interval = 50 ms / speed` (Zeitlupe möglich).
- **`figsize` / `dpi`:** Standardmäßig größere Figur (10×10); im Performance-Mode nur
  leicht reduzierte dpi statt verkleinerter Figur.

Weitere Verbesserungen:
- Trajektorien werden **einmalig** in NumPy-Arrays umgewandelt (vorher in jedem Frame neu).
- Es wird über **alle je registrierten Körper** (`history`-Keys) geplottet statt nur über
  die aktuell lebenden → kein `KeyError` mehr nach einer Verschmelzung.
- **Proportionale Punktgrößen** über `marker_size_for()` (relativ zum größten Körper).
- **NaN-robuste Achsenskalierung** (`np.nanmax`, Fallback `max_val = 1`).
- `update()` nutzt `idx = min(frame, len-1)` als Sicherheitsnetz gegen Index-Überlauf.

---

## 4. Neue / ersetzte Inhaltszellen

- **Alte Zelle 10** (manueller Einzeltest „TEST-SETUP FÜR DEN THEIA-CONTROLLER") wurde
  **entfernt** und ersetzt durch:
  - **Markdown (Aufgabe 2.2):** Erläuterung der Aufgabe + Begründung des kleinen
    Zeitschritts (`0.002` Tage) gegen „Tunneln" der Kollisionsprüfung.
  - **Code – 3×3-Szenario-Matrix:** `run_theia_scenario()` läuft alle 9 Kombinationen
    (3/8/12 km/s × Mitte/Mond/Erde), erzeugt eine Ergebnistabelle mit Kollisionspartner
    und minimalen Abständen und beantwortet die Leitfrage (kollisionsfreie Konfiguration?).
  - **Markdown – Beobachtungen.**
  - **2 Animations-Zellen:** je ein repräsentativer Kollisions- und Vorbeiflug-Fall.
- In der Basis-Zelle (Erde/Mond) ist die stationäre Erde-Variante als Kommentar ergänzt.

---

## 5. Bug-Analyse

### ✅ Behoben / korrekt
- **History-Synchronisation nach Verschmelzung** – verifiziert: alle Trajektorien gleich
  lang, NaN-Auffüllung vorne (neuer Körper) und hinten (aufgelöste Körper) korrekt.
  Kein „dictionary changed size during iteration" (Key-Anlage passiert in
  `check_collisions` vor der Aufzeichnungs-Schleife).
- **Physik der Verschmelzung:** Impuls- und Schwerpunktberechnung korrekt
  (`newVeloc = (m1·v1 + m2·v2)/M`, `newPos` = massengewichteter Schwerpunkt).
- **Integrationsverfahren:** semi-implizites Euler (erst `update_velocity`, dann
  `update_position`) – stabil für Orbits.
- **Impulsbilanz im Theia-Setup:** In `baue_erde_mond_sim` heben sich Erd- und
  Mondimpuls auf (5,972e24·12,573 ≈ 7,348e22·1022) → Schwerpunkt ruht. Korrekt.

### ⚠️ Offene Befunde

1. **Radius-Formel bei Verschmelzung widerspricht der Doku.** *(offen)*
   Code: `newRadius = sqrt(r1² + r2²)`. Der Markdown-Text (Zelle 4) behauptet einen
   *„volumenbasierten Radius"*. Volumenerhaltung wäre `(r1³ + r2³)^(1/3)`.
   Beispiel (r1=r2=1e6 m): Code liefert **1,41e6 m**, volumenbasiert wären **1,26e6 m**.
   → Entweder Code auf `(r1**3 + r2**3)**(1/3)` ändern oder die Doku korrigieren.
   *(Hinweis: bereits im alten Stand vorhanden, nicht durch deine Änderung entstanden.
   Bewusst noch offen gelassen, da nicht beauftragt.)*

### ✅ In diesem Durchgang behoben

Siehe Abschnitt **6** für Details. Kurz:

2. **`distance_scale` (toter Parameter)** – entfernt aus Signatur, Docstring und Zuweisung
   des `Visualizer`. Keine Aufrufstelle übergab den Parameter, daher gefahrlos.
3. **Kommentar-/Wert-Widerspruch in der Basis-Zelle** – Zeitschritt-Kommentar auf „1 Tag"
   korrigiert; Ansicht-Kommentare an die tatsächlich aktive Standardansicht angeglichen.
4. **Label bei Mehrfach-Verschmelzung** – `replace(...)` durch eine `split("+")`-basierte
   Auswertung ersetzt, die alle echten Kollisionspartner korrekt auflistet.

### ℹ️ Hinweis statt Bug

5. **Speicher-/Laufzeit der 3×3-Matrix (kein Bug):**
   100 Tage / 0,002 = **50.000 Schritte pro Szenario × 9 = 450.000 Schritte**; alle 9
   Simulationen werden in `sims` vollständig im Speicher gehalten (jeweils komplette
   Trajektorien). Das erklärt teils die Notebook-Größe (~97 MB) und kann mehrere Minuten
   Laufzeit + viel RAM kosten. Bei Bedarf `total_time` senken oder nur benötigte Sims
   behalten. **Status:** Die Matrix wurde einmalig zum Testen eingebaut und wird
   weiterhin von **Sheila** bearbeitet (entsprechender Hinweis steht jetzt im Notebook).

### Kein Crash gefunden
Die geänderten Klassen wurden isoliert ausgeführt (Konstruktor, `add_body`, `step`,
`check_collisions`, `merge_bodies`) inkl. erzwungener Verschmelzung – **kein Laufzeitfehler**.
Imports sind vollständig (`io`, `contextlib`, `math`, `matplotlib.colors`).

---

## 6. Durchgeführte Korrekturen (Stand 2026-06-16)

Folgende Punkte wurden direkt im Notebook umgesetzt (alle Outputs blieben erhalten,
chirurgische Bearbeitung der betroffenen Zellen):

| # | Zelle | Korrektur |
|---|---|---|
| 1 | 6 (`Visualizer`) | Toten Parameter `distance_scale` entfernt (Signatur `__init__(self, simulation, body_scale=1.0)`, Docstring-Zeile, Zuweisung). |
| 2 | 7 (Basis-Setup) | Kommentar `# Zeitschritt: 0.1 Tage …` → `# Zeitschritt: 1 Tag`. |
| 3 | 7 (Basis-Setup) | Ansicht-Kommentare neu sortiert: aktive Zeile `anim = vis.animate_3d()` steht nun korrekt unter „Standardansicht"; Alternativen sauber als `view='top'` / `view='side'` auskommentiert. |
| 4 | 10 (Markdown 2.2) | Status-Hinweis ergänzt: 3×3-Matrix einmalig zum Testen eingebaut, **wird weiterhin von Sheila bearbeitet**. |
| 5 | 11 (Matrix-Code) | Label-Bug behoben – statt `b.name.replace("+Theia","")` wird der Name über `split("+")` zerlegt und alle Partner außer Theia werden zusammengesetzt. |

**Verifikation der Label-Korrektur:**

| Body-Name | `kollision_mit` (neu) |
|---|---|
| `Erde+Theia` | `Erde` |
| `Mond+Theia` | `Mond` |
| `Theia+Erde` | `Erde` |
| `Erde+Theia+Mond` | `Erde+Mond` |
| `Theia` / `Erde` (keine Verschmelzung) | `None` |

Nach den Änderungen: gültiges JSON (15 Zellen), keine verbleibenden `distance_scale`-Vorkommen.

---

## 7. Könnte man kritisieren? (Stilfrage, kein Bug)

### `calculate_gravity()` ist nicht vektorisiert

Die Methode iteriert mit einer doppelten Python-`for`-Schleife über alle Körperpaare:
```python
for i in range(len(self.bodies)):
    for j in range(i + 1, len(self.bodies)):
        ...
```
Innerhalb der Schleife werden zwar NumPy-Vektoren genutzt (`np.linalg.norm`, Vektor-
subtraktion), aber die paarweise Kraftberechnung selbst läuft als reine Python-Schleife
statt über NumPy-Broadcasting (z. B. via `np.newaxis`, eine Differenzmatrix aller
Positionen, vektorisierte Distanz- und Kraftberechnung in wenigen Zeilen ohne Schleife).

**Warum das in einem „Scientific Programming Lab" auffallen könnte:** Solche Kurse legen
häufig Wert darauf, NumPy nicht nur als Container für Vektoren zu nutzen, sondern für
vektorisierte Berechnungen ("NumPy als Rechenwerkzeug statt nur als Datenstruktur"). Eine
doppelte Python-Schleife über Body-Paare bedeutet bei N Körpern O(N²) reinen
Python-Overhead statt vektorisierter NumPy-Operationen.

**Relevanz hier:** Bei 2–3 Körpern (Erde, Mond, Theia) ist der Performance-Unterschied
vernachlässigbar – der Punkt ist rein stilistisch/didaktisch, kein funktionaler Fehler.
Ob der Professor das als Abzugskriterium wertet, lässt sich nicht pauschal sagen; eine
vektorisierte Variante ließe sich aber leicht als Alternative neben der bestehenden
Schleifen-Implementierung ergänzen, falls ihr auf der sicheren Seite sein wollt.

---

---

## 8. Zweiter Review-Durchgang (Stand 2026-06-16) – neue Befunde

Erneute Durchsicht des **aktuellen** Notebook-Stands. Die folgenden Punkte sind **noch nicht
umgesetzt** – es sind Vorschläge zur Diskussion. Reihenfolge nach Wichtigkeit.

### 🐞 8.1 Startzustand (t=0) wird nicht in `history` gespeichert  *(verifiziert)*

`Simulation.step()` ruft erst `check_collisions()` / `calculate_gravity()` auf, **bewegt** dann
alle Körper um einen vollen Zeitschritt und **hängt erst danach** die Position an `history` an.
Folge: Der **Anfangszustand bei t=0 fehlt** komplett – der erste gespeicherte Eintrag ist
bereits die Position nach einem Zeitschritt.

**Verifikation** (bewegter Einzelkörper, `v=[10,0,0]`, 3 Schritte):
```
Gespeicherte X-Positionen: [10. 20. 30.]   # erwartet wäre [0. 10. 20.] oder [0. 10. 20. 30.]
-> Startposition x=0 fehlt? True
```

**Auswirkung:** gering, aber real – Animationen starten eine Zeitschritt-Breite „zu spät", und
`_min_distance` kann einen exakt bei t=0 liegenden kleinsten Abstand verpassen (bei Theia aus
4 Mio. km unkritisch, prinzipiell aber falsch).

**Fix-Vorschlag** – Initialposition einmalig vor dem Lauf aufzeichnen, z. B. am Anfang von `run()`:
```python
def run(self, total_time):
    if self.recorded_steps == 0:          # Startzustand t=0 festhalten
        for name in self.history:
            alive = next((b for b in self.bodies if b.name == name), None)
            self.history[name].append(alive.position.copy() if alive else np.full(3, np.nan))
        self.recorded_steps += 1
    steps = int(total_time / self.time_step)
    for _ in range(steps):
        self.step()
```
(Alternativ direkt in `step()` **vor** dem Bewegen speichern statt danach.)

### 🐞 8.2 `input()` blockiert „Run All" / macht das Notebook nicht reproduzierbar

Zelle mit dem `Visualizer` (Erde-Mond-Basis) fragt die Ansicht interaktiv ab:
```python
auswahl = input("Ansicht wählen (1=Standard, 2=Oben, 3=Seite): ").strip()
```
Bei „Alle Zellen ausführen" **hängt** das Notebook an dieser Stelle und wartet auf Eingabe;
beim automatischen Bewerten/Re-Run ist das Verhalten nicht reproduzierbar.

**Fix-Vorschlag:** feste Default-Ansicht als Variable, Alternativen als Kommentar – analog zur
Basis-Zelle:
```python
vis = Visualizer(simulation)
view = 'standard'        # Alternativen: 'top' (Draufsicht) / 'side' (Seitenansicht)
anim = vis.animate_3d(view=view)
anim
```

### 📄 8.3 Abschnitt 3 dieser Datei ist gegenüber dem Code veraltet

Abschnitt 3 beschreibt für `animate_3d()` die Parameter **`speed`** (`interval = 50 ms / speed`)
und **`dpi`** als eigenen Parameter. Im aktuellen Code existieren beide **nicht**:
- tatsächliche Signatur: `animate_3d(self, view='standard', performance_mode=False, max_frames=150, dynamic_scaling=False, figsize=None)`
- `interval = 50` ist **fest**; die Geschwindigkeit wird laut Docstring über die +/- Buttons des
  jshtml-Players geregelt.

**Fix-Vorschlag:** Abschnitt 3 entsprechend korrigieren (kein `speed`-Parameter, `dpi` wird intern
abhängig von `performance_mode` gesetzt, nicht übergeben). Reiner Doku-Abgleich, kein Code-Bug.

### ⚠️ 8.4 Rückgabewert von `check_collisions()` wird nie genutzt

`check_collisions()` baut `collCheck` auf und gibt es zurück, `step()` ignoriert den Wert aber.
Entweder verwenden (z. B. um nach einer Verschmelzung etwas zu loggen) oder den Rückgabewert
entfernen. Kein Bug, nur toter Code.

### ⚠️ 8.5 Robustheit: Division durch Null in `calculate_geometry()`

```python
senkrecht_vektor = np.array([-r_erde_mond[1], r_erde_mond[0], 0.0])
senkrecht_normiert = senkrecht_vektor / np.linalg.norm(senkrecht_vektor)
```
Liegt die Erde-Mond-Linie rein entlang der Z-Achse (x=y=0), ist `senkrecht_vektor = [0,0,0]` →
Division durch Null (NaN). In den vorhandenen Setups (Erde/Mond in der X/Y-Ebene) tritt das nicht
auf, daher nur ein Robustheits-Hinweis – ggf. Norm prüfen und einen Fallback-Senkrechtenvektor
wählen.

### 🧹 8.6 Mehrfache / lokale Imports (Cleanup)

- `import numpy as np` steht in der `Body`-Zelle **und** in der `ScenarioController`-Zelle;
  `import math` taucht mehrfach auf.
- `import matplotlib.colors as mcolors` steht **innerhalb** von `_blend_colors()` (wird bei jeder
  Verschmelzung neu ausgeführt).

Kein funktionaler Fehler, aber sauberer wäre, die Imports gebündelt einmal oben zu halten.

### ℹ️ 8.7 Markdown-Formel des Gravitationsgesetzes (Didaktik)

Die Markdown-Zelle schreibt die Kraft als Vektorgleichung
$\vec{F}_{ji} = -G\,\frac{m_i m_j}{\lVert \vec r_{ji}\rVert^{2}}$, hat auf der rechten Seite aber
**keine Richtungsangabe** (Einheitsvektor). Konsistent vektoriell wäre
$\vec{F}_{ji} = -G\,\frac{m_i m_j}{\lVert \vec r_{ji}\rVert^{2}}\,\hat{r}_{ji}$ bzw.
$-G\,\frac{m_i m_j}{\lVert \vec r_{ji}\rVert^{3}}\,\vec r_{ji}$. Der **Code** ist korrekt (er nutzt
`direction = r_vector / distance`); nur der Formeltext ist verkürzt. Reine Darstellungs-/Doku-Frage.

### Status-Übersicht Abschnitt 8

| # | Befund | Schweregrad | Status |
|---|---|---|---|
| 8.1 | t=0 fehlt in `history` | Bug (gering) | offen – Fix vorgeschlagen |
| 8.2 | `input()` blockiert Run-All | Usability-Bug | offen – Fix vorgeschlagen |
| 8.3 | Abschnitt 3 veraltet (`speed`/`dpi`) | Doku | offen |
| 8.4 | ungenutzter Rückgabewert | toter Code | offen |
| 8.5 | Division durch Null (Edge Case) | Robustheit | offen |
| 8.6 | Mehrfach-/lokale Imports | Stil | offen |
| 8.7 | Formeltext ohne Richtungsvektor | Doku/Didaktik | offen |

---

*Erstellt am 2026-06-16 durch automatischen Vergleich gegen Commit `7426c3a`;
Abschnitt 6 ergänzt nach Umsetzung der Korrekturen, Abschnitt 7 ergänzt nach Diskussion
über NumPy-Vektorisierung; Abschnitt 8 ergänzt nach zweitem Review-Durchgang (neue,
noch offene Befunde).*
