[B04_Lösung.md](https://github.com/user-attachments/files/28447969/B04_Losung.md)
# Dokumentation – Programmieren 2: Syntaxhighlighting
https://github.com/Baikov2003/prog2_ybel_syntaxhighlighting.git

## 1. Projektübersicht

Dieses Projekt implementiert ein vereinfachtes Syntaxhighlighting für MiniJava-Code. Ziel ist es, bestimmte Bestandteile eines Java-ähnlichen Quelltexts mithilfe von regulären Ausdrücken zu erkennen und farblich hervorzuheben.

Im Projekt wurden insbesondere folgende Komponenten bearbeitet:

- `MiniJavaTokens`
- `RegexHighlighter`
- `ScanningHighlighter`
- JUnit-Tests für Token, RegexHighlighter und ScanningHighlighter
- GitHub-Actions-CI-Pipeline
- Feature-Branches und Pull Requests

Die Klasse `SyntaxHighlighter` gibt dabei den allgemeinen Ablauf vor:

```java
collectMatches(text)
normalize(candidates)
resolveConflicts(normalized)
```

Die konkreten Highlighter unterscheiden sich darin, wie sie Treffer sammeln und wie sie mit überlappenden Regionen umgehen.

---

## 2. MiniJavaTokens

### 2.1 Ziel

Die Klasse `highlighting.presets.MiniJavaTokens` definiert die Tokens, die von den Highlightern erkannt werden sollen. Jedes Token besteht aus einem regulären Ausdruck und einer Farbe aus `MiniJavaColours`.

Die Methode

```java
public static List<Token> defaultTokens()
```

liefert die verwendete Token-Liste zurück.

### 2.2 Implementierte Tokens

Es wurden Tokens für folgende Sprachelemente definiert:

| Token-Art | Beschreibung |
|---|---|
| Javadoc-Kommentare | Kommentare der Form `/** ... */` |
| Blockkommentare | Kommentare der Form `/* ... */` |
| Zeilenkommentare | Kommentare von `//` bis zum Zeilenende |
| String-Literale | Text zwischen doppelten Anführungszeichen |
| Character-Literale | Ein Zeichen oder eine Escape-Sequenz zwischen einfachen Anführungszeichen |
| Annotationen | Annotationen wie `@Override` oder `@my-annotation` |
| Keywords | MiniJava-Schlüsselwörter wie `class`, `public`, `return`, `new` usw. |

Die Reihenfolge der Tokens ist wichtig. Javadoc-Kommentare werden vor normalen Blockkommentaren geprüft, damit ein Kommentar wie `/** ... */` nicht als normaler Blockkommentar behandelt wird. Kommentare stehen außerdem vor Keywords, damit Schlüsselwörter innerhalb von Kommentaren nicht einzeln hervorgehoben werden, wenn die spätere Konfliktauflösung greift.

### 2.3 Wichtige reguläre Ausdrücke

#### Javadoc-Kommentar

```java
Pattern.compile("/\\*\\*[\\s\\S]*?\\*/")
```

Dieser Ausdruck erkennt mehrzeilige Javadoc-Kommentare. Durch `[\s\S]` werden sowohl normale Zeichen als auch Zeilenumbrüche abgedeckt.

#### Blockkommentar

```java
Pattern.compile("/\\*(?!\\*)[\\s\\S]*?\\*/")
```

Dieser Ausdruck erkennt normale Blockkommentare. Durch `(?!\*)` wird verhindert, dass Javadoc-Kommentare als normale Blockkommentare erkannt werden.

#### Zeilenkommentar

```java
Pattern.compile("//[^\\r\\n]*")
```

Dieser Ausdruck erkennt einen Kommentar von `//` bis vor den nächsten Zeilenumbruch.

#### String-Literal

```java
Pattern.compile("\"([^\"\\\\]|\\\\.)*\"")
```

Dieser Ausdruck erkennt Strings zwischen doppelten Anführungszeichen und erlaubt einfache Escape-Sequenzen.

#### Character-Literal

```java
Pattern.compile("'([^'\\\\]|\\\\.)'")
```

Dieser Ausdruck erkennt ein einzelnes Zeichen oder eine Escape-Sequenz zwischen einfachen Anführungszeichen.

#### Annotation

```java
Pattern.compile("@[A-Za-z][A-Za-z-]*")
```

Dieser Ausdruck erkennt Annotationen, die mit `@` beginnen und anschließend Buchstaben oder Minuszeichen enthalten.

#### Keywords

```java
Pattern.compile(
    "(?<![A-Za-z0-9_$])(?:package|import|class|public|private|final|return|null|new)(?![A-Za-z0-9_$])")
```

Dieser Ausdruck erkennt die geforderten Keywords nur dann, wenn sie nicht Teil eines größeren Bezeichners sind.

---

## 3. JUnit-Tests für MiniJavaTokens

### 3.1 Ziel

Für die Tokens wurden JUnit-Tests erstellt, um sicherzustellen, dass die regulären Ausdrücke die gewünschten Texte erkennen und unerwünschte Treffer vermeiden.

### 3.2 Getestete Fälle

Die Tests prüfen unter anderem:

- Treffer am Anfang eines Textes
- Treffer in der Mitte eines Textes
- Treffer am Ende eines Textes
- mehrere Treffer im selben Text
- Fälle ohne Treffer
- Strings mit kommentarähnlichem Inhalt
- Kommentare mit keywordähnlichem Inhalt
- Annotationen am Zeilenanfang und nach Leerzeichen
- Javadoc-Kommentare und normale Blockkommentare
- Keywords als ganze Wörter und nicht als Teil von Bezeichnern

### 3.3 Beispielhafte Testidee

Ein String wie

```java
"this is not // a comment"
```

soll vollständig als String erkannt werden. Das enthaltene `//` soll in diesem Test nicht verhindern, dass der String als String-Literal erkannt wird.

Ein Bezeichner wie

```java
className
```

darf nicht teilweise als Keyword `class` erkannt werden.

---

## 4. RegexHighlighter

### 4.1 Ziel

Der `RegexHighlighter` implementiert eine einfache, naive Highlighting-Strategie. Dabei werden alle Tokens unabhängig voneinander auf den gesamten Eingabetext angewendet. Dadurch können zunächst überlappende Treffer entstehen.

Die Klasse implementiert insbesondere:

```java
public List<HighlightRegion> collectMatches(String text)
```

und

```java
public List<HighlightRegion> resolveConflicts(List<HighlightRegion> regions)
```

### 4.2 collectMatches

In `collectMatches` werden alle Tokens aus `MiniJavaTokens.defaultTokens()` auf den Eingabetext angewendet. Alle gefundenen `HighlightRegion`s werden in einer gemeinsamen Liste gesammelt und zurückgegeben.

Vereinfachtes Prinzip:

```java
List<HighlightRegion> result = new ArrayList<>();

for (Token token : MiniJavaTokens.defaultTokens()) {
  result.addAll(token.test(text));
}

return result;
```

Die Sortierung übernimmt anschließend die vorgegebene `normalize`-Methode der Basisklasse.

### 4.3 resolveConflicts

Da der RegexHighlighter zunächst alle Tokens unabhängig voneinander sammelt, können überlappende Regionen entstehen. Diese werden in `resolveConflicts` entfernt.

Die Regel lautet:

- Die normalisierte Liste wird von vorne nach hinten durchlaufen.
- Eine Region wird nur übernommen, wenn sie mit keiner bereits übernommenen Region überlappt.
- Wenn eine Überlappung existiert, wird die spätere Region verworfen.
- Halb-offene Intervalle werden verwendet: `[start, end)`.
- Zwei Regionen `[0,5)` und `[5,10)` überlappen nicht.

Die Überlappungsbedingung lautet:

```java
a.start() < b.end() && b.start() < a.end()
```

---

## 5. JUnit-Tests für RegexHighlighter

### 5.1 Ziel

Die Tests für den RegexHighlighter prüfen sowohl das naive Sammeln der Treffer als auch die anschließende Konfliktauflösung.

### 5.2 Getestete Fälle

Die Tests prüfen unter anderem:

- einfache Fälle ohne Überlappung
- mehrere Treffer in einem Text
- Keywords innerhalb eines Kommentars
- Javadoc-Kommentar als längerer Treffer
- angrenzende Regionen wie `[0,5)` und `[5,10)`
- Leerstring
- Text ohne Matches
- direkte Tests der Methode `resolveConflicts`

### 5.3 Beispiel: Keyword innerhalb eines Kommentars

Ein Text wie

```java
// return new class
```

enthält zwar die Keywords `return`, `new` und `class`, soll nach der Konfliktauflösung aber als ein einziger Zeilenkommentar hervorgehoben werden.

---

## 6. ScanningHighlighter

### 6.1 Ziel

Der `ScanningHighlighter` arbeitet anders als der RegexHighlighter. Er liest den Text von links nach rechts und entscheidet an jeder Position, welches Token dort am besten passt.

Dabei gilt:

- Es wird nur ein Treffer an der aktuellen Position ausgewählt.
- Es wird der längste passende Treffer gewählt.
- Bei gleich langen Treffern gewinnt das Token, das früher in der Token-Liste steht.
- Nach einem Treffer springt der Scanner direkt an das Ende der Region.
- Dadurch entstehen bereits sortierte und nicht überlappende Regionen.

### 6.2 collectMatches

Die Methode `collectMatches` durchläuft den Text mit einer Positionsvariable. Für jede Position werden die Tokens geprüft. Wenn ein Token genau an dieser Position passt, wird der längste Treffer gespeichert.

Vereinfachtes Prinzip:

```java
while (position < text.length()) {
  HighlightRegion bestMatch = null;

  for (Token token : tokens) {
    for (HighlightRegion region : token.test(text)) {
      if (region.start() == position && region.end() > region.start()) {
        if (bestMatch == null || length(region) > length(bestMatch)) {
          bestMatch = region;
        }
      }
    }
  }

  if (bestMatch != null) {
    result.add(bestMatch);
    position = bestMatch.end();
  } else {
    position++;
  }
}
```

### 6.3 normalize

Da der ScanningHighlighter die Regionen bereits sortiert und konfliktfrei erzeugt, wird `normalize` als Identitätsfunktion implementiert:

```java
return candidates;
```

---

## 7. JUnit-Tests für ScanningHighlighter

### 7.1 Ziel

Die Tests für den ScanningHighlighter prüfen, ob das links-nach-rechts-Verfahren korrekt funktioniert und die längste passende Region auswählt.

### 7.2 Getestete Fälle

Die Tests prüfen unter anderem:

- einfache Treffer in korrekter Reihenfolge
- Leerstring
- Text ohne Matches
- angrenzende Regionen
- Auswahl des längsten Treffers
- keine Hervorhebung von Keywords innerhalb von Kommentaren
- keine Hervorhebung von kommentarähnlichem Text innerhalb von Strings
- sortierte und nicht überlappende Ergebnisregionen
- `normalize` als Identitätsfunktion

### 7.3 Beispiel: Kommentar mit Keywords

Bei folgendem Text:

```java
// return new class
```

wird der gesamte Kommentar als eine Region erkannt. Die enthaltenen Keywords werden nicht zusätzlich hervorgehoben, weil der Scanner nach dem Kommentar direkt an dessen Ende springt.

---

## 8. CI-Pipeline mit GitHub Actions

### 8.1 Ziel

Für das Projekt wurde eine einfache CI-Pipeline mit GitHub Actions eingerichtet.

Die Workflow-Datei liegt unter:

```text
.github/workflows/ci.yml
```

### 8.2 Trigger

Die Pipeline wird ausgelöst bei:

- Pull Requests
- manueller Auslösung über `workflow_dispatch`

Die Pipeline wird nicht automatisch bei Pushes auf `master` oder `main` ausgeführt.

### 8.3 Jobs

Die CI-Pipeline besteht aus drei Jobs:

| Job | Aufgabe |
|---|---|
| `build` | Kompiliert das Projekt mit `./gradlew classes --no-daemon` |
| `test` | Führt die JUnit-Tests mit `./gradlew test --no-daemon` aus |
| `format` | Prüft die Formatierung mit `./gradlew spotlessCheck --no-daemon` |

### 8.4 Ergebnis

Die CI wurde in den Pull Requests ausgeführt. Die Jobs `Build`, `Test` und `Format` liefen erfolgreich, bevor die jeweiligen Änderungen gemergt wurden.

---

## 9. Branches und Pull Requests

### 9.1 Ziel

Die Implementierungsaufgaben wurden in separaten Feature-Branches bearbeitet. Dadurch sind die Änderungen besser nachvollziehbar und können einzeln geprüft werden.

### 9.2 Verwendete Branches

| Aufgabe | Branch | Pull Request |
|---|---|---|
| CI-Pipeline | `ci-pipeline` | `Add GitHub Actions CI pipeline` |
| MiniJavaTokens | `feature/minijava-tokens` | `Implement MiniJava tokens` |
| RegexHighlighter | `feature/regex-highlighter` | `Implement RegexHighlighter` |
| ScanningHighlighter | `feature/scanning-highlighter` | `Implement ScanningHighlighter` |

### 9.3 Merge-Reihenfolge

Die Pull Requests wurden in sinnvoller Reihenfolge erstellt und gemergt:

1. CI-Pipeline
2. MiniJavaTokens
3. RegexHighlighter
4. ScanningHighlighter

Diese Reihenfolge ist sinnvoll, weil der RegexHighlighter und der ScanningHighlighter auf den Tokens aus `MiniJavaTokens` aufbauen.

---

## 10. Reviews

### 10.1 Status

Die Aufgabe sieht gegenseitige Reviews der Pull Requests vor. In meinem aktuellen Bearbeitungsstand wurden die Pull Requests zwar erstellt, geprüft und über die CI validiert, jedoch wurden keine Reviews durch Kommiliton:innen durchgeführt.

Dieser Punkt ist daher noch offen beziehungsweise nicht vollständig erfüllt.

### 10.2 Geplantes Vorgehen für Reviews

Für eine vollständige Bearbeitung müssten folgende Schritte nachgeholt werden:

1. Kommiliton:innen als Reviewer zu den Pull Requests hinzufügen.
2. Review-Kommentare an konkreten Codestellen einholen.
3. Auf Kommentare antworten.
4. Kommentare bewusst schließen.
5. Falls sinnvoll, Änderungen auf Basis des Feedbacks umsetzen und erneut pushen.
6. Im Gegenzug selbst Pull Requests anderer Kommiliton:innen reviewen.
7. Die GitHub-Funktion „Code vorschlagen“ für konkrete Änderungsvorschläge verwenden.

### 10.3 Beispielhafte Review-Kommentare

Mögliche sinnvolle Review-Kommentare wären:

```text
Der Regex für Annotationen erlaubt aktuell nur Buchstaben und Minuszeichen. Sollen Ziffern nach dem ersten Zeichen ebenfalls erlaubt sein?
```

```text
Die Konfliktauflösung ist korrekt. Ein kurzer Kommentar zur halb-offenen Intervalllogik könnte die Lesbarkeit verbessern.
```

```text
Beim ScanningHighlighter wird token.test(text) an jeder Position erneut ausgeführt. Für die Aufgabenstellung ist das ausreichend, könnte aber bei langen Texten ineffizient sein.
```

---

## 11. Ausführen des Projekts und der Tests

### 11.1 Kompilieren

```powershell
.\gradlew.bat classes
```

### 11.2 Tests ausführen

```powershell
.\gradlew.bat test
```

### 11.3 Formatierung prüfen

```powershell
.\gradlew.bat spotlessCheck
```

### 11.4 Formatierung automatisch anwenden

```powershell
.\gradlew.bat spotlessApply
```

---

## 12. Zusammenfassung

Im Projekt wurde ein vereinfachtes Syntaxhighlighting für MiniJava umgesetzt. Die Token-Erkennung erfolgt über reguläre Ausdrücke. Darauf aufbauend wurden zwei unterschiedliche Highlighting-Strategien implementiert:

- Der `RegexHighlighter` sammelt zunächst alle Treffer und löst Konflikte danach auf.
- Der `ScanningHighlighter` scannt den Text von links nach rechts und wählt an jeder Position den längsten passenden Treffer.

Die Implementierung wurde durch JUnit-Tests abgesichert. Zusätzlich wurde eine GitHub-Actions-CI-Pipeline eingerichtet, die Build, Tests und Formatierung in Pull Requests automatisch prüft.

Die Branch- und Pull-Request-Struktur wurde umgesetzt. Der Review-Teil ist im aktuellen Stand noch offen, da keine gegenseitigen Reviews mit Kommiliton:innen durchgeführt wurden.
