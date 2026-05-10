# Git Branch Experimente

---

## Nr. 1 – Änderung nur im master

```bash
git checkout -b end origin/end
git diff master end --name-only
git add hero.md
git commit -m "Änderung im master"
git merge end
```

**Beobachtung:** Der Merge von `end` in `master` verläuft ohne Konflikte.

**Erklärung:** Es entsteht kein Konflikt, da die Datei nur im `master` geändert wurde und im Branch `end` unverändert bleibt. Git kann die Änderungen daher automatisch zusammenführen.

**Konflikte:** Es treten keine Konflikte auf.

---

## Nr. 2 – Gleiche Datei, Änderungen an unterschiedlichen Stellen

```bash
git diff master end --name-only
notepad rucksack.md
git merge end
git add rucksack.md
git commit
```

**Beobachtung:** Wenn unterschiedliche Zeilen bearbeitet wurden, kann Git die Änderungen oft automatisch zusammenführen. Bei überlappenden Änderungen kann jedoch ein Konflikt entstehen.

**Erklärung:** Git kann Änderungen nur dann automatisch zusammenführen, wenn sich die bearbeiteten Bereiche nicht überschneiden. Sobald beide Branches denselben Bereich verändern, ist keine automatische Entscheidung möglich.

**Konfliktlösung:**
Die Datei wird geöffnet, die Konfliktmarkierungen werden entfernt und die gewünschten Inhalte werden manuell kombiniert. Danach wird die Datei mit `git add` markiert und der Merge mit `git commit` abgeschlossen.

---

## Nr. 3 – identische vs. unterschiedliche Änderungen

**Beobachtung:**
Wenn beide Branches exakt dieselben Änderungen enthalten, entsteht kein Konflikt. Wenn die Änderungen jedoch unterschiedlich sind, entsteht ein Merge-Konflikt.

**Erklärung:** Git erkennt nur Unterschiede im Inhalt. Sind die Änderungen identisch, behandelt Git sie als gleich. Bei unterschiedlichen Versionen derselben Stelle muss der Benutzer entscheiden, welche Version gültig ist.

**Konflikte:** Konflikte werden manuell durch Bearbeiten der Datei und anschließendes Committen gelöst.

---

## Nr. 4 – Rebase vor Merge

```bash
git checkout master
git commit -m "Änderung auf master"
git checkout -b end origin/end
git rebase master
git checkout master
git merge end
```

**Beobachtung:** Nach dem Rebase erfolgt der Merge häufig als Fast-Forward-Merge ohne separaten Merge-Commit.

**Erklärung:** Durch den Rebase werden die Commits des Branches `end` so verschoben, dass sie auf dem aktuellen Stand von `master` aufbauen. Dadurch entsteht eine lineare Historie, und Git muss keinen echten Merge-Commit mehr erzeugen.

**Konflikte:** Konflikte treten nur auf, wenn beide Branches dieselben Stellen unterschiedlich geändert haben. Diese müssen dann manuell gelöst werden.

---

## Gesamtfazit

- Kein Konflikt entsteht bei Änderungen in unterschiedlichen Dateien oder nicht überlappenden Bereichen.
- Konflikte entstehen bei widersprüchlichen Änderungen an derselben Stelle.
- Konflikte werden durch manuelles Bearbeiten der Datei und anschließendes `git add` und `git commit` gelöst.
- Rebase sorgt für eine lineare Historie und erleichtert den anschließenden Merge.

## JUnit
**Testfälle**

package catcafe;

import static org.junit.jupiter.api.Assertions.*;

import org.junit.jupiter.api.Test;

class CatCafeTest {

    @Test
    void addCat_shouldIncreaseCatCount() {
        // given
        CatCafe cafe = new CatCafe();
        FelineOverLord cat = new FelineOverLord("Mimi", 5);

        // when
        cafe.addCat(cat);

        // then
        assertEquals(1, cafe.getCatCount());
    }

    @Test
    void addMultipleCats_shouldIncreaseCountCorrectly() {
        // given
        CatCafe cafe = new CatCafe();

        // when
        cafe.addCat(new FelineOverLord("Mimi", 5));
        cafe.addCat(new FelineOverLord("Balu", 7));

        // then
        assertEquals(2, cafe.getCatCount());
    }

    @Test
    void getCatCount_emptyCafe_shouldBeZero() {
        // given
        CatCafe cafe = new CatCafe();

        // when
        long count = cafe.getCatCount();

        // then
        assertEquals(0, count);
    }

    @Test
    void getCatByName_shouldReturnCorrectCat() {
        // given
        CatCafe cafe = new CatCafe();
        FelineOverLord cat = new FelineOverLord("Mimi", 5);
        cafe.addCat(cat);

        // when
        FelineOverLord result = cafe.getCatByName("Mimi");

        // then
        assertEquals(cat, result);
    }

    @Test
    void getCatByName_unknownName_shouldReturnNull() {
        // given
        CatCafe cafe = new CatCafe();

        // when
        FelineOverLord result = cafe.getCatByName("Ghost");

        // then
        assertNull(result);
    }

    @Test
    void getCatByName_null_shouldReturnNull() {
        // given
        CatCafe cafe = new CatCafe();

        // when
        FelineOverLord result = cafe.getCatByName(null);

        // then
        assertNull(result);
    }

    @Test
    void getCatByWeight_shouldReturnCatInRange() {
        // given
        CatCafe cafe = new CatCafe();
        FelineOverLord cat = new FelineOverLord("Mimi", 5);
        cafe.addCat(cat);

        // when
        FelineOverLord result = cafe.getCatByWeight(4, 6);

        // then
        assertEquals(cat, result);
    }

    @Test
    void getCatByWeight_outOfRange_shouldReturnNull() {
        // given
        CatCafe cafe = new CatCafe();
        cafe.addCat(new FelineOverLord("Mimi", 5));

        // when
        FelineOverLord result = cafe.getCatByWeight(6, 10);

        // then
        assertNull(result);
    }

    @Test
    void getCatByWeight_invalidRange_shouldReturnNull() {
        // given
        CatCafe cafe = new CatCafe();

        // when
        FelineOverLord result = cafe.getCatByWeight(10, 5);

        // then
        assertNull(result);
    }

    @Test
    void addCat_null_shouldThrowException() {
        // given
        CatCafe cafe = new CatCafe();

        // when + then
        assertThrows(NullPointerException.class, () -> cafe.addCat(null));
    }
}

**Warum sind die von Ihnen formulierten Testfälle relevant?**
Die Testfälle sind relevant, weil sie das Verhalten der Klasse CatCafe vollständig und systematisch überprüfen. Sie testen zentrale Funktionen wie das Hinzufügen von Katzen, das Zählen der Katzen sowie das Suchen nach Katzen anhand von Name und Gewicht. Zusätzlich enthalten sie auch Rand- und Fehlerfälle, zum Beispiel leere Zustände, ungültige Eingaben oder nicht vorhandene Suchergebnisse. Dadurch wird sichergestellt, dass die Klasse in normalen, kritischen und fehlerhaften Situationen korrekt funktioniert.

**Warum halten Sie die formulierten Testfälle für unterschiedlich?**
Die Testfälle sind unterschiedlich, weil sie verschiedene Aspekte und Szenarien der Klasse abdecken. Einige Tests prüfen Standardverhalten (z. B. erfolgreiche Suche oder korrektes Hinzufügen), andere testen Grenzfälle (z. B. leeres Café oder ungültige Gewichtsspannen) und wieder andere behandeln Fehlerfälle (z. B. null-Eingaben). Außerdem testen die Fälle unterschiedliche Methoden der Klasse, sodass jede Funktion isoliert und unabhängig von den anderen überprüft wird.

## Remotes und CI-Pipeline
Für die Aufgabe wurde eine CI-Pipeline in GitHub Actions eingerichtet.

Die Pipeline wird bei jedem Push auf den Branch „master“ automatisch ausgeführt und besteht aus drei Schritten:

1. Projekt bauen (kompilieren)
   ./gradlew build

2. JUnit-Tests aus der Aufgabe „Katzen-Café“ ausführen
   ./gradlew test

3. Code-Formatierung überprüfen (Spotless)
   ./gradlew spotlessCheck
