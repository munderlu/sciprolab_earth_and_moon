# Behobene Bugs – Earth and Moon Simulation

---

## Bug 1 – `Body.__init__`: `color`-Attribut wurde nicht gespeichert

**Klasse:** `Body`  
**Methode:** `__init__`

**Problem:**  
Der Parameter `color` wurde im Konstruktor entgegengenommen, aber nie als Instanzattribut gespeichert. Dadurch war die Farbe eines Körpers nach der Initialisierung unwiederbringlich verloren und ein späterer Zugriff auf `body.color` hätte einen `AttributeError` erzeugt.

**Vorher:**
```python
def __init__(self, name:str, position, velocity, radius:float, mass:float, color: str):
    ...
    self.mass = mass
    # self.color wurde nie gesetzt
```

**Nachher:**
```python
self.color = color # Farbe des Körpers für die spätere Visualisierung
```

---

## Bug 2 – `step()`: Kraft wurde nicht in Beschleunigung umgerechnet

**Klasse:** `Simulation`  
**Methode:** `step`

**Problem:**  
`calculate_gravity()` gibt Kraftvektoren (in Newton) zurück. `update_velocity` erwartet aber eine **Beschleunigung** (in m/s²). Laut Newtons zweitem Gesetz gilt `F = m·a`, also `a = F/m`. Ohne die Division durch die Masse wurde jeder Körper so behandelt, als hätte er eine Masse von 1 kg – beide Körper wurden identisch stark beschleunigt. Da die Gravitationskraft zwischen Erde und Mond ~1,98 × 10²⁰ N beträgt, erhielten beide Körper im ersten Zeitschritt eine Geschwindigkeit von ~3,96 × 10²¹ m/s und flogen sofort auseinander.

**Vorher:**
```python
body.update_velocity(forces[body.name], self.time_step)
```

**Nachher:**
```python
# F = m*a umformen zu a = F/m, da update_velocity eine Beschleunigung erwartet
acceleration = forces[body.name] / body.mass
body.update_velocity(acceleration, self.time_step)
```

---

## Bug 3 – `merge_bodies()`: Falscher Nenner bei der Schwerpunktberechnung

**Klasse:** `Simulation`  
**Methode:** `merge_bodies`

**Problem:**  
Der Schwerpunkt zweier Körper ergibt sich aus dem massegewichteten Mittelwert ihrer Positionen: `(m1·r1 + m2·r2) / (m1 + m2)`. Im Code stand stattdessen `body1.mass * body2.mass` im Nenner (das **Produkt** der Massen statt der **Summe**). Das liefert bei realen Planetenmassen (10²² bis 10²⁴ kg) einen um viele Größenordnungen falschen Positionswert und setzt den verschmolzenen Körper an eine völlig falsche Stelle im Raum.

**Vorher:**
```python
newPos = (body1.mass * body1.position + body2.mass * body2.position)/(body1.mass*body2.mass)
```

**Nachher:**
```python
newPos = (body1.mass * body1.position + body2.mass * body2.position) / newMass # Schwerpunkt: Division durch Gesamtmasse
```

---

## Bug 4 – `check_collisions()`: Listenmodifikation während der Iteration

**Klasse:** `Simulation`  
**Methode:** `check_collisions`

**Problem:**  
`merge_bodies` entfernt zwei Körper aus `self.bodies` und fügt einen neuen hinzu, während die äußere `for`-Schleife noch über die ursprünglichen Indizes aus `range(len(self.bodies))` läuft. Da `range()` einmalig bei Schleifenbeginn ausgewertet wird, laufen die Indizes ins Leere oder greifen auf falsche Körper zu – abhängig von der Listenänderung drohen ein `IndexError` oder still übergangene Kollisionspaare.

**Vorher:**
```python
def check_collisions(self):
    collCheck = False
    for i in range(len(self.bodies)):
        for j in range(i+1, len(self.bodies)):
            ...
            if (distance <= (b1.radius + b2.radius)):
                self.merge_bodies(b1,b2)
                collCheck = True
    return collCheck
```

**Nachher:**  
Nach jeder Kollision werden beide Schleifen sofort abgebrochen und die Suche beginnt von vorne, bis keine weiteren Kollisionen mehr vorhanden sind:
```python
def check_collisions(self):
    collCheck = False
    found_collision = True
    while found_collision:
        found_collision = False
        for i in range(len(self.bodies)):
            for j in range(i+1, len(self.bodies)):
                ...
                if (distance <= (b1.radius + b2.radius)):
                    self.merge_bodies(b1, b2)
                    collCheck = True
                    found_collision = True
                    break # Liste hat sich geändert, Suche neu starten
            if found_collision:
                break
    return collCheck
```

---

## Bug 5 – Setup: Simulationszeitraum viel zu kurz

**Datei:** Setup-Zelle  

**Problem:**  
`time_step=20` Sekunden und `total_time=1000` Sekunden ergaben nur **50 Simulationsschritte**, die insgesamt 1000 Sekunden (≈ 17 Minuten) realer Zeit abdecken. Der Mond benötigt für eine vollständige Umlaufbahn jedoch ~27,3 Tage (≈ 2.360.000 Sekunden). In 1000 Sekunden legt er lediglich ~1.022 km zurück – 0,04 % seiner Umlaufbahn – was in der Animation vollständig unsichtbar ist.

**Vorher:**
```python
simulation = Simulation(20)
simulation.run(1000)
```

**Nachher:**
```python
simulation = Simulation(3600)       # Zeitschritt: 1 Stunde
simulation.run(28 * 24 * 3600)     # 28 Tage ≈ ein Mondmonat
```

---

## Bug 6 – Setup: Sekunden als Zeiteinheit erschwert die Lesbarkeit

**Datei:** Setup-Zelle und `Simulation.__init__`

**Problem:**  
Die Verwendung von Sekunden als Zeiteinheit erzwingt große, schwer lesbare Zahlenwerte in den Simulationsparametern (z. B. `28 * 24 * 3600` statt `28`). Da sich der Mondmonat natürlicherweise in **Tagen** beschreiben lässt, ist die Sekunde als Basiseinheit ungeeignet. Außerdem musste die Gravitationskonstante `G` an die neue Zeiteinheit angepasst werden, da sie in m³/(kg·s²) angegeben ist.

**Vorher:**
```python
# Simulation
self.G = 6.67430e-11               # m³/(kg·s²)

# Setup
simulation = Simulation(3600)      # 1 Stunde in Sekunden
mond = Body(..., [0, 1022, 0], ...)  # Geschwindigkeit in m/s
simulation.run(28 * 24 * 3600)     # 28 Tage in Sekunden
```

**Nachher:**  
Zeiteinheit auf **Tage** umgestellt. `G` wird intern umgerechnet, die Mondgeschwindigkeit von m/s auf m/Tag konvertiert:
```python
# Simulation
self.G = 6.67430e-11 * (86400 ** 2)  # Umrechnung von m³/(kg·s²) auf m³/(kg·Tag²)

# Setup
simulation = Simulation(0.1)          # Zeitschritt: 0.1 Tage (≈ 2,4 Stunden)
mond = Body(..., [0, 1022 * 86400, 0], ...)  # 1022 m/s × 86400 s/Tag → m/Tag
simulation.run(28)                     # 28 Tage ≈ ein Mondmonat
```
