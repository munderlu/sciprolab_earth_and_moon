# Projektstruktur: Gravitations-Simulation

<!--toc:start-->
- [Projektstruktur: Gravitations-Simulation](#projektstruktur-gravitations-simulation)
  - [1. Klasse: Body (Himmelskörper)](#1-klasse-body-himmelskörper)
  - [2. Klasse: Simulation (Die Physik-Engine)](#2-klasse-simulation-die-physik-engine)
  - [3. Klasse: Visualizer (Darstellung)](#3-klasse-visualizer-darstellung)
  - [4. Klasse: ScenarioController (Szenario-Steuerung)](#4-klasse-scenariocontroller-szenario-steuerung)
<!--toc:end-->

Für die N-Körper-Simulation wird das System in vier logische Hauptkomponenten unterteilt: Die physikalischen Objekte, die Simulations-Engine, die visuelle Ausgabe und die Szenario-Steuerung.

- Himmelskörper und Unit Test: Lara
- Visualizer: Bekir
- Core-Simulation: Luke
- Kollision: Darian
- Theia - Physik: Viktoria
- Theia - Testauswertung: Sheila

## 1. Klasse: Body (Himmelskörper)

Repräsentiert einen einzelnen Himmelskörper als Punktmasse bzw. rigide Kugel.

*Attribute:*
- name (String): Name des Körpers.
- mass (Float): Masse in kg.
- radius (Float): Abgeleitet aus dem vorgegebenen Durchmesser in Metern.
- position (NumPy Array 3D): Kartesische Koordinaten in Metern.
- velocity (NumPy Array 3D): Geschwindigkeitsvektor in m/s.
- color (String): Farbcode für die Visualisierung.

*Methoden:*
- __init__(self, ...): Konstruktor zur Initialisierung aller Eigenschaften.
- update_velocity(self, force, dt): Aktualisiert die Geschwindigkeit basierend auf der einwirkenden Kraft und dem Zeitschritt dt.
- update_position(self, dt): Aktualisiert die Position basierend auf der aktuellen Geschwindigkeit.
- get_momentum(self): Gibt den aktuellen Impuls zurück.

---

## 2. Klasse: Simulation (Die Physik-Engine)

Verwaltet die Himmelskörper, berechnet die gravitativen Wechselwirkungen und treibt die Zeit in konfigurierbaren Schritten voran.

*Attribute:*
- bodies (List): Liste aller aktiven Body-Objekte in der Simulation.
- G (Float): Gravitationskonstante.
- time_step (Float): Der konfigurierbare Zeitschritt für den beschleunigten Ablauf.
- history (Dict): Speichert die Positionen aller Körper über die Zeit ab.

*Methoden:*
- add_body(self, body): Fügt einen neuen Körper zur Simulation hinzu.
- calculate_gravity(self): Berechnet die paarweisen Anziehungskräfte aller Körper untereinander.
- check_collisions(self): Erkennt, wenn zwei Körper sich im Raum berühren.
- merge_bodies(self, body1, body2): Führt eine inelastische Kollision durch, bei der die beiden Körper verschmelzen sowie sich Masse und Impuls addieren. Der neue Radius wird volumenbasiert angepasst.
- step(self): Führt einen einzelnen Simulationsschritt durch.
- run(self, total_time): Führt die Simulation in einer Schleife aus.

---

## 3. Klasse: Visualizer (Darstellung)

Kapselt die Logik für die visuelle Darstellung der berechneten Daten.

*Attribute:*
- simulation (Simulation): Referenz auf das Simulationsobjekt.
- distance_scale (Float): Maßstab für die Abstände zwischen den Körpern.
- body_scale (Float): Unabhängiger Maßstab für die Körpergröße.

*Methoden:*
- plot_static_trajectory(self): Zeichnet die Bahnen der Körper als statisches Diagramm im Koordinatensystem.
- animate(self): Erstellt eine interaktive Animation des Systemablaufs über die Zeit.
- set_scales(self, dist_scale, body_scale): Ermöglicht die Anpassung der Visualisierungsmaßstäbe.

---

## 4. Klasse: ScenarioController (Szenario-Steuerung)

Automatisiert das mathematische Aufsetzen und die Parameter-Sweeps für spezielle Testszenarien mit einfallenden Körpern.

*Attribute:*
- initial_distance (Float): Die vorgegebene Startentfernung des einfallenden Körpers.
- speed_magnitude (Float): Die zu testende Anfangsgeschwindigkeit.
- target_type (String): Das gewählte Ziel der Flugbahn.

*Methoden:*
- __init__(self, speed_magnitude, target_type): Konstruktor zur Konfiguration des jeweiligen Testlaufs.
- calculate_geometry(self): Berechnet die exakten initialen 3D-Positionsvektoren für das gewählte geometrische Startlayout.
- calculate_velocity_vector(self, start_pos, target_pos): Bestimmt den präzisen, normierten Richtungsvektor vom Start zum gewählten Ziel und skaliert ihn mit der Geschwindigkeit.
- setup_simulation(self): Erstellt eine spielbereite Instanz der Klasse Simulation und fügt die Himmelskörper hinzu.
