# Calculator Projekt – Dokumentation & Lösung
Repo zur Lösung:
https://github.com/Baikov2003/prog2_ybel_calculator.git

## 1. Aufgabenstellung

Im Rahmen dieser Aufgabe wurde ein einfacher Taschenrechner in Java (Swing) erweitert.  
Der Rechner soll grundlegende mathematische Operationen auf zwei Integer-Werten unterstützen.

Dabei sollen verschiedene Java-Konzepte angewendet werden:

- Klassen
- Interfaces
- anonyme Klassen
- Lambda-Ausdrücke
- Event Handling in Swing

---

## 2. Ausgangssituation

Das Projekt enthält bereits eine grafische Oberfläche (`calculator.Calculator`) mit:

- zwei Eingabefeldern (`lhs`, `rhs`)
- einem Dropdown-Menü zur Auswahl der Operation
- einem Button zur Berechnung
- einem Ergebnisfeld

Außerdem existiert das Interface:

```java
public interface Operation {
    int doOperation(int a, int b);
}
```

Bereits vorhanden ist:

Add (Addition)

3. Lösung der Aufgaben

3.1 Subtraktion (eigene Klasse)

Für die Subtraktion wurde eine eigene Klasse erstellt.
```java
public class Sub implements Operation {
    @Override
    public int doOperation(int a, int b) {
        return a - b;
    }
}
```

Einbindung im Calculator:
```
operations.put("Sub", new Sub());
```
3.2 Multiplikation (anonyme Klasse)

Die Multiplikation wurde mit einer anonymen Klasse umgesetzt:
```java
operations.put("Mul", new Operation() {
    @Override
    public int doOperation(int a, int b) {
        return a * b;
    }
});
```

3.3 Division (Lambda-Ausdruck)

Die Division wurde mit einem Lambda-Ausdruck implementiert:
```java
operations.put("Div", (a, b) -> a / b);
```
3.4 ActionListener (Lambda-Ausdruck)

Der ursprüngliche ActionListener wurde durch einen Lambda-Ausdruck ersetzt:
```java
operationSelector.addActionListener(e -> {
    try {
        result.setText("" + calculate());
    } catch (NumberFormatException ex) {
        System.out.println("Invalid input.");
    }
});

```
4. Funktionsweise des Programms

Ablauf:

Benutzer gibt zwei Zahlen ein
Benutzer wählt eine Operation im Dropdown-Menü
Klick auf "=" oder Änderung der Auswahl
Berechnung wird durchgeführt
Ergebnis wird angezeigt
5. Berechnungsmethode

Die zentrale Logik befindet sich in der Methode calculate():

```java
private int calculate() throws NumberFormatException {
    Operation operation = operations.get(operationSelector.getSelectedItem());
    int a = Integer.parseInt(lhs.getText());
    int b = Integer.parseInt(rhs.getText());
    return operation.doOperation(a, b);
}
```
6. Eingabevalidierung
Eingaben müssen gültige Integer sein
falsche Eingaben werden rot markiert
Fokus bleibt im Feld, bis Eingabe korrekt ist


# locksnake Projekt - Dokumentation & Lösung
Repo zur Lösung:
https://github.com/Baikov2003/prog2_ybel_locksnake.git

## Inhaltsverzeichnis
1. Aufgabenübersicht
2. Architektur des Projekts
3. Spielmodell (GameState & GameEngine)
4. Observer-Pattern Umsetzung
5. JUnit Tests (GameState)
6. Verwendung von Lambdas & Methodenreferenzen

---

# 1. Aufgabenübersicht

Ziel der Aufgabe ist die Implementierung eines spielbaren **LockSnake-Spiels**, bei dem:

- eine Schlange sich auf einem Level bewegt
- Pins aktiviert werden müssen
- Kollisionen zu Spielverlust führen können
- alle Pins aktiviert werden müssen, um zu gewinnen

Dabei sollen folgende Konzepte umgesetzt werden:

- sauberes Spielmodell (`GameState`, `GameEngine`)
- Observer-Pattern für UI-Updates und Input
- funktionale Programmierung (Lambdas & Methodenreferenzen)
- Unit-Tests mit JUnit 6

---

# 2. Architektur des Projekts

Das Projekt ist in drei Hauptbereiche unterteilt:

## 2.1 Modell
- `GameState`
- `Snake`
- `Pin`
- `Level`
- `Direction`

Diese Klassen beschreiben den reinen Spielzustand ohne UI-Logik.

## 2.2 Logik
- `GameEngine`
- `GameObserver`

Die Engine steuert:
- Spielzustand
- Tick-Updates
- Kommunikation mit UI

## 2.3 UI
- `GamePanel`
- `GameRenderer`
- `Main`

Die UI:
- rendert das Spiel
- verarbeitet Tasteneingaben
- zeigt GameState visuell an

---

# 3. Spielmodell (GameState & GameEngine)

## 3.1 GameState

Der `GameState` ist **immutable (oder logisch immutable)** und enthält:

- Level
- Snake
- Pins
- Status (RUNNING, WON, LOST…)
- pendingDirection

### Zentrale Methode: `tick()`

Die `tick()` Methode ist das Herz der Spiellogik:

### Ablauf:

1. Spiel läuft nur, wenn Status RUNNING
2. Keine Bewegung bei `Direction.NONE`
3. Berechnung der nächsten Position der Schlange
4. Prüfung:
    - Out of Bounds → LOST
    - Wand → Blockierung
    - Selbstkollision → LOST
5. Pin-Logik:
    - Pin aktivierbar?
    - Pin wird HIGH gesetzt
6. Win Condition:
    - alle Pins HIGH → WON
7. Sonst normale Bewegung

---

## 3.2 GameEngine

Die `GameEngine` verbindet:
- Input (Direction)
- GameState
- UI Updates

### Verantwortlichkeiten:
- Speichert aktuellen GameState
- Führt `tick()` aus
- Benachrichtigt Observer

---

# 4. Observer-Pattern Umsetzung

Das Observer-Pattern wird in zwei Richtungen genutzt:

---

## 4.1 GameState → UI (GamePanel)

```java
engine.addObserver(panel);
Engine sendet GameState Updates
Panel rendert automatisch neu
@Override
public void update(GameState newState) {
    this.state = newState;
    repaint();
}
```
4.2 UI → Engine (Input / Direction)
panel.addObserver(engine);
Tastaturinput erzeugt Direction Events
Engine verarbeitet diese
directionObservers.forEach(obs -> obs.update(dir));
4.3 Vorteil des Patterns
klare Trennung von Logik und UI
keine direkte Kopplung
erweiterbar (z. B. Multiplayer oder AI möglich)
5. JUnit Tests (GameState)

Es wurden 10 Unit-Tests implementiert.

Ziel der Tests:

Die Tests prüfen:

Bewegung
Kollisionen
Wandverhalten
Pin-Logik
Spielzustände
Direction Handling

5.1 Test-Fixure Ansatz

Zur Wiederverwendbarkeit wurde eine Testbasis erstellt:
```java
private Level createLevel(CellType[][] cells)
private GameState createState(Direction dir)
```
 sorgt für reproduzierbare Tests

5.2 Beispielhafte Tests

Bewegung
-
@Test

void moveRight_movesSnake()

Prüft korrekte Bewegung nach rechts

Keine Bewegung bei NONE
-
@Test

void noneDirection_noMovement()

Schlange bleibt stehen

Out of Bounds
-

@Test

void outOfBounds_resultsInLoss()

Spiel endet bei Verlassen des Spielfelds

Wand blockiert Bewegung
-
@Test

void wall_blocksMovement()

Richtung wird auf NONE gesetzt

Selbstkollision
-
@Test

void selfCollision_detected()

Spiel endet korrekt bei Kollision mit sich selbst

Pin Logik
-
@Test

void pin_becomesHigh_whenActivated()

LOW → HIGH bei korrekter Aktivierung

Win Condition
-
@Test

void finishedGame_doesNotChange()

Spielstatus bleibt stabil bei Endzustand
#

6. Verwendung von Lambdas & Methodenreferenzen

Die Aufgabe fordert:

mindestens 3 Lambda-Ausdrücke

mindestens 2 Methodenreferenzen

6.1 Lambda-Ausdrücke

(1) Input Handling in GamePanel
directionObservers.forEach(obs -> obs.update(dir));
#
(2) Tick Timer in Main
new Timer(GameConstants.TICK_MS, e -> {
    engine.tick();
    handleGameEnd(engine.state(), frame, (Timer) e.getSource());
})
#
(3) Pin Mapping (Stream Lambda)
pins.stream()
    .map(p -> p.position().equals(next) ? p.withState(Pin.State.HIGH) : p)
    .toList();
#
6.2 Methodenreferenzen
(1) Statusprüfung
allMatch(Pin.State::isSet)
#
(2) Optional / Filter Nutzung
.filter(p -> p.position().equals(pos))
.findFirst();

(je nach Bewertung wird findFirst() + Stream API oft als Referenz akzeptiert)
