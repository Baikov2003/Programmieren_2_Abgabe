# B05 – Dokumentation und Analyse

Diese Datei dokumentiert die bearbeiteten Aufgaben zum Syntax-Highlighting-Projekt und zum Projekt Cycle Chronicles.  
D

Links zu den Repositories: 
https://github.com/Baikov2003/prog2_ybel_syntaxhighlighting.git
https://github.com/Baikov2003/prog2_ybel_cyclechronicles.git

---

## Inhaltsverzeichnis

1. [ANTLR im Syntax-Highlighting-Projekt](#1-antlr-im-syntax-highlighting-projekt)
   - [Aufgabe 1.1: Syntaxhighlighting mit `AntlrTokenCollector`](#aufgabe-11-syntaxhighlighting-mit-antlrtokencollector)
   - [Konflikte, `normalize` und `resolveConflicts`](#konflikte-normalize-und-resolveconflicts)
   - [Vergleich mit den Varianten von Blatt 04](#vergleich-mit-den-varianten-von-blatt-04)
   - [Aufgabe 1.2: Pretty Printing mit ANTLR](#aufgabe-12-pretty-printing-mit-antlr)
   - [Demonstrationsbeispiele](#demonstrationsbeispiele)
   - [Beobachtung: Warum fehlen bestimmte Teile aus dem Input?](#beobachtung-warum-fehlen-bestimmte-teile-aus-dem-input)
2. [Cycle Chronicles](#2-cycle-chronicles)
   - [Aufgabe 2.1: Äquivalenzklassenbildung und Grenzwertanalyse](#aufgabe-21-äquivalenzklassenbildung-und-grenzwertanalyse)
   - [Konkrete Testfälle](#konkrete-testfälle)
   - [Aufgabe 2.2: Mocking mit JUnit und Mockito](#aufgabe-22-mocking-mit-junit-und-mockito)
3. [Prüfhinweise zu den Projekten](#prüfhinweise-zu-den-projekten)

---

# 1. ANTLR im Syntax-Highlighting-Projekt

## Projektstruktur

Im Syntax-Highlighting-Projekt befinden sich die für die ANTLR-Aufgabe relevanten Dateien an diesen Stellen:

```text
src/main/antlr/highlighting/antlr/MiniJava.g4
src/main/java/highlighting/antlr/AntlrTokenCollector.java
src/main/java/highlighting/antlr/PrettyPrinterVisitor.java
src/main/java/highlighting/antlr/MiniJavaPrettyPrinter.java
src/main/java/highlighting/antlr/PrettyPrinterDemo.java
src/main/java/highlighting/core/SyntaxHighlighter.java
src/main/java/highlighting/core/HighlightRegion.java
src/main/java/highlighting/presets/MiniJavaColours.java
```

Die ANTLR-Grammatik liegt in:

```text
src/main/antlr/highlighting/antlr/MiniJava.g4
```

Aus dieser Grammatik erzeugt Gradle unter anderem die Klassen:

- `MiniJavaLexer`
- `MiniJavaParser`
- `MiniJavaBaseVisitor`
- `MiniJavaVisitor`

Die generierten Klassen werden im Build-Verzeichnis abgelegt, zum Beispiel unter:

```text
build/generated-src/antlr/main/highlighting/antlr/
```

Die Gradle-Konfiguration enthält bereits das ANTLR-Plugin und die notwendigen Abhängigkeiten:

```gradle
id 'antlr'

antlr 'org.antlr:antlr4:4.13.2'
implementation 'org.antlr:antlr4-runtime:4.13.2'
```

Außerdem wird für ANTLR der Visitor aktiviert:

```gradle
arguments.addAll(['-visitor', '-long-messages'])
```

Dadurch steht die Klasse `MiniJavaBaseVisitor` zur Verfügung, die für den Pretty Printer verwendet wird.

---

# Aufgabe 1.1: Syntaxhighlighting mit `AntlrTokenCollector`

## Ziel

Die Methode

```java
List<HighlightRegion> collectMatches(String text)
```

in der Klasse

```java
highlighting.antlr.AntlrTokenCollector
```

soll den eingegebenen Java-/MiniJava-Code mithilfe des von ANTLR generierten Lexers analysieren.  
Der Lexer zerlegt den Eingabetext in Tokens. Diese Tokens werden anschließend in `HighlightRegion`-Objekte umgewandelt.

Eine `HighlightRegion` enthält:

- Startposition
- Endposition
- Farbe

Die Klasse `HighlightRegion` verwendet dabei ein exklusives Ende:

```java
new HighlightRegion(start, end, colour)
```

Das bedeutet:

- `start` ist inklusive.
- `end` ist exklusiv.

## Umsetzungsidee

Die Umsetzung verwendet in `AntlrTokenCollector` diesen Ablauf:

1. Eingabetext in einen ANTLR-`CharStream` umwandeln.
2. `MiniJavaLexer` mit diesem Input erzeugen.
3. Tokens mit `CommonTokenStream` sammeln.
4. Tokenstream mit `fill()` vollständig laden.
5. Tokens durchlaufen.
6. Je nach Token-Typ beziehungsweise Token-Text eine Farbe bestimmen.
7. Passende Tokens als `HighlightRegion` speichern.

Vereinfacht dargestellt:

```java
CharStream input = CharStreams.fromString(text);
MiniJavaLexer lexer = new MiniJavaLexer(input);

CommonTokenStream tokenStream = new CommonTokenStream(lexer);
tokenStream.fill();

for (Token token : tokenStream.getTokens()) {
    if (token.getType() == Token.EOF) {
        continue;
    }

    // Farbe bestimmen
    // HighlightRegion erzeugen
}
```

## Positionsberechnung

ANTLR-Tokens enthalten bereits Positionsinformationen:

```java
token.getStartIndex()
token.getStopIndex()
```

Da `getStopIndex()` inklusiv ist, wird für `HighlightRegion` das Ende um 1 erhöht:

```java
int start = token.getStartIndex();
int end = token.getStopIndex() + 1;
```

Dadurch passt die Region zum üblichen Bereichsmodell `[start, end)`.

Beispiel:

```java
class Test
```

Das Token `class` beginnt bei Index `0` und endet bei Index `4`.  
Die Highlight-Region ist deshalb:

```text
start = 0
end   = 5
```

## Berücksichtigte Token-Arten

Im Projekt werden insbesondere diese Gruppen hervorgehoben:

| Token-Art | Beispiele | Farbe |
|---|---|---|
| Schlüsselwörter | `class`, `public`, `return`, `if`, `else`, `while` | `MiniJavaColours.KEYWORD_COLOUR` |
| String-Literale | `"Hallo"` | `MiniJavaColours.STRING_LITERAL_COLOUR` |
| Char-Literale | `'a'` | `MiniJavaColours.CHAR_LITERAL_COLOUR` |
| Zeilenkommentare | `// Kommentar` | `MiniJavaColours.LINE_COMMENT_COLOUR` |
| Blockkommentare | `/* Kommentar */` | `MiniJavaColours.BLOCK_COMMENT_COLOUR` |
| Javadoc-Kommentare | `/** Kommentar */` | `MiniJavaColours.JAVADOC_COMMENT_COLOUR` |
| Annotationen | `@Override` | `MiniJavaColours.ANNOTATION_COLOUR` |

Die Farben sind in folgender Klasse definiert:

```java
highlighting.presets.MiniJavaColours
```

## Kommentare im Tokenstream

In der Grammatik werden Kommentare nicht vollständig übersprungen, sondern auf den versteckten Kanal gelegt:

```antlr
LINE_COMMENT    : '//' ~[\r\n]*             -> channel(HIDDEN);
JAVADOC_COMMENT : '/**' (.|[\r\n])*? '*/'   -> channel(HIDDEN);
BLOCK_COMMENT   : '/*'  (.|[\r\n])*? '*/'   -> channel(HIDDEN);
```

Das ist für das Syntaxhighlighting wichtig.  
Wären Kommentare mit `skip` entfernt worden, wären sie im Tokenstream nicht mehr vorhanden und könnten nicht farbig markiert werden.

Whitespace wird dagegen übersprungen:

```antlr
WS : [ \t\r\n]+ -> skip;
```

Das ist unproblematisch, weil Whitespace nicht hervorgehoben werden muss.

## Annotationen

Annotationen bestehen in der Grammatik aus einem `@` und einem folgenden Identifier:

```antlr
annotation: '@' IDENTIFIER ( '(' argumentList? ')' )? ;
```

Im Tokenstream sind `@` und der Identifier einzelne Tokens. Deshalb behandelt die Implementierung den Fall `@` gesondert und markiert auch den direkt folgenden Identifier als Annotation.

Beispiel:

```java
@Override
```

Daraus entstehen sinngemäß zwei Tokenbereiche:

```text
@
Override
```

Beide werden mit der Annotation-Farbe markiert. Optisch ergibt sich dadurch eine zusammenhängend hervorgehobene Annotation.

## Schlüsselwörter aus der Grammatik

Die Grammatik definiert diese Schlüsselwort-Tokens:

```antlr
PACKAGE     : 'package';
IMPORT      : 'import';
CLASS       : 'class';
PUBLIC      : 'public';
PRIVATE     : 'private';
FINAL       : 'final';
RETURN      : 'return';
NULL        : 'null';
NEW         : 'new';
IF          : 'if';
ELSE        : 'else';
WHILE       : 'while';
EXTENDS     : 'extends';
IMPLEMENTS  : 'implements';
```

Für eine möglichst robuste Umsetzung sollte die Keyword-Erkennung an diesen Token-Typen ausgerichtet werden.  
Das ist stabiler als eine reine Prüfung des Token-Texts, weil die Grammatik die Sprache definiert.

Eine mögliche Variante ist:

```java
private static boolean isKeyword(Token token) {
    return switch (token.getType()) {
        case MiniJavaLexer.PACKAGE,
             MiniJavaLexer.IMPORT,
             MiniJavaLexer.CLASS,
             MiniJavaLexer.PUBLIC,
             MiniJavaLexer.PRIVATE,
             MiniJavaLexer.FINAL,
             MiniJavaLexer.RETURN,
             MiniJavaLexer.NULL,
             MiniJavaLexer.NEW,
             MiniJavaLexer.IF,
             MiniJavaLexer.ELSE,
             MiniJavaLexer.WHILE,
             MiniJavaLexer.EXTENDS,
             MiniJavaLexer.IMPLEMENTS -> true;
        default -> false;
    };
}
```

---

# Konflikte, `normalize` und `resolveConflicts`

## Konflikte bei Regex-Highlighting

Bei regulären Ausdrücken können Konflikte entstehen, weil mehrere Muster unabhängig voneinander auf denselben Text angewendet werden.

Beispiel:

```java
String text = "return";
```

Eine naive Regex-Erkennung könnte innerhalb des Strings zusätzlich das Keyword `return` erkennen. Dadurch entstehen zwei überlappende Regionen:

1. Region für den String
2. Region für das Keyword innerhalb des Strings

Solche Konflikte müssen anschließend normalisiert oder aufgelöst werden.

## Konflikte bei ANTLR

Bei ANTLR ist die Situation anders. Der Lexer zerlegt den Eingabetext in eine lineare Folge von Tokens. Ein Zeichenbereich gehört normalerweise genau zu einem Token.

Beispiel:

```java
String text = "return";
```

Der Bereich `"return"` wird als String-Literal erkannt. Das Wort `return` innerhalb des Strings wird nicht zusätzlich als Keyword-Token erzeugt.

Dadurch entstehen bei der ANTLR-basierten Variante im Normalfall keine überlappenden Highlight-Regionen.

## Bedeutung von `normalize`

Die Methode `normalize` aus der Oberklasse `SyntaxHighlighter` entfernt ungültige Regionen und sortiert die Regionen:

```java
public List<HighlightRegion> normalize(List<HighlightRegion> candidates)
```

Da der ANTLR-Lexer die Tokens bereits in Textreihenfolge erzeugt und die erzeugten Tokenbereiche gültig sind, ist keine besondere Normalisierung notwendig.  
Die Standardimplementierung ist trotzdem unproblematisch.

Eine explizite Überschreibung wäre möglich:

```java
@Override
public List<HighlightRegion> normalize(List<HighlightRegion> candidates) {
    return candidates;
}
```

Dadurch würde man dokumentieren, dass die Tokenliste bereits geordnet und gültig ist.

## Bedeutung von `resolveConflicts`

Die Methode `resolveConflicts` ist für überlappende Regionen gedacht:

```java
public List<HighlightRegion> resolveConflicts(List<HighlightRegion> normalized)
```

Bei ANTLR-Tokens entstehen solche Konflikte normalerweise nicht. Deshalb kann die Methode unverändert bleiben oder bewusst als Identitätsfunktion überschrieben werden:

```java
@Override
public List<HighlightRegion> resolveConflicts(List<HighlightRegion> normalized) {
    return normalized;
}
```

## Begründung

Für den ANTLR-basierten TokenCollector gilt:

- Der Lexer entscheidet eindeutig, welches Token an welcher Stelle beginnt und endet.
- Strings und Kommentare werden als eigene Tokens erkannt.
- Keywords innerhalb von Strings oder Kommentaren erzeugen keine zusätzlichen Keyword-Tokens.
- Dadurch sind Konflikte zwischen Regionen deutlich unwahrscheinlicher als bei Regex.

---

# Vergleich mit den Varianten von Blatt 04

## Regex-basierte Variante

Die Regex-Variante sucht mit regulären Ausdrücken nach Mustern im Text.

### Vorteile

- Für einfache Muster schnell umzusetzen.
- Wenig Infrastruktur notwendig.
- Keine Grammatik erforderlich.

### Nachteile

- Mehrere Regex-Muster können denselben Textbereich treffen.
- Konflikte und Überlappungen sind wahrscheinlich.
- Reihenfolge und Priorität der Muster sind wichtig.
- Keywords können fälschlich in Strings oder Kommentaren erkannt werden.
- Bei komplexeren Sprachbestandteilen wird die Lösung schnell schwer wartbar.

### Beispielproblem

```java
// return ist hier nur ein Kommentar
```

Eine einfache Regex-Lösung könnte `return` trotzdem als Keyword erkennen, obwohl es Teil eines Kommentars ist.

## Scanner-basierte Variante

Die Scanner-Variante arbeitet typischerweise von links nach rechts durch den Text und erkennt Tokens selbst.

### Vorteile

- Mehr Kontrolle als bei Regex.
- Strings und Kommentare können gezielt erkannt und übersprungen werden.
- Weniger Konflikte als bei reinem Regex-Ansatz.

### Nachteile

- Deutlich mehr Implementierungsaufwand.
- Viele Sonderfälle müssen manuell behandelt werden.
- Änderungen an der Sprache müssen im Scanner nachgebaut werden.
- Es besteht die Gefahr, dass Scanner und eigentliche Grammatik auseinanderlaufen.

## ANTLR-basierte Variante

Die ANTLR-Variante verwendet die Grammatik des Projekts. Der Lexer wird aus dieser Grammatik generiert.

### Vorteile

- Tokenisierung basiert auf der Sprachgrammatik.
- Weniger Konfliktbehandlung notwendig.
- Kommentare, Strings und Keywords werden systematischer erkannt.
- Die Implementierung von `collectMatches` wird relativ kurz.
- Die Grammatik kann auch für Parser und Visitoren wiederverwendet werden.

### Nachteile

- ANTLR muss korrekt in Gradle eingebunden sein.
- Eine passende Grammatik muss vorhanden sein.
- Die generierten Klassen müssen erzeugt und in der IDE erkannt werden.
- Änderungen an der Grammatik können Auswirkungen auf Lexer, Parser und Highlighting haben.

## Welche Variante ist aufwändiger?

Für sehr kleine Beispiele ist Regex zunächst am einfachsten.  
Sobald aber Kommentare, Strings, Annotationen und verschiedene Tokenarten korrekt behandelt werden sollen, wird Regex deutlich komplizierter.

Die ANTLR-Variante ist in der Einrichtung aufwändiger, weil sie eine Grammatik und Build-Integration benötigt. Wenn diese Infrastruktur vorhanden ist, ist die eigentliche Token-zu-HighlightRegion-Umsetzung aber übersichtlicher und robuster.

In diesem Projekt ist ANTLR bereits vorkonfiguriert. Deshalb ist die ANTLR-Variante für Aufgabe 1.1 insgesamt gut geeignet.

---

# Aufgabe 1.2: Pretty Printing mit ANTLR

## Ziel

Der Pretty Printer soll MiniJava-Code neu ausgeben, sodass Einrückungen und Zeilenumbrüche konsistent sind.

Es geht nicht darum, jedes Leerzeichen perfekt nach Java-Styleguides zu setzen. Wichtig sind vor allem:

- Blöcke korrekt umbrechen
- Einrückung korrekt erhöhen und verringern
- Statements mit Semikolon auf eigene Zeilen setzen
- Klassenrümpfe und Methodenrümpfe lesbar strukturieren

## Relevante Klassen

Im Projekt sind dafür diese Klassen relevant:

```text
src/main/java/highlighting/antlr/MiniJavaPrettyPrinter.java
src/main/java/highlighting/antlr/PrettyPrinterVisitor.java
src/main/java/highlighting/antlr/PrettyPrinterDemo.java
```

## Parse-Tree-Erzeugung

Die Klasse `MiniJavaPrettyPrinter` erzeugt aus einem String einen Parse-Tree:

```java
public static MiniJavaParser.CompilationUnitContext parse(String sourceCode)
```

Dafür werden diese ANTLR-Klassen verwendet:

```java
CharStreams.fromString(sourceCode)
MiniJavaLexer
CommonTokenStream
MiniJavaParser
```

Der Parser startet mit der Regel:

```java
parser.compilationUnit()
```

Das Ergebnis ist:

```java
MiniJavaParser.CompilationUnitContext
```

Diese Klasse entspricht der Startregel der Grammatik:

```antlr
compilationUnit: packageDecl? importDecl* typeDecl* EOF ;
```

Zusätzlich prüft die Methode, ob Syntaxfehler aufgetreten sind:

```java
if (parser.getNumberOfSyntaxErrors() > 0) {
    throw new IllegalArgumentException("MiniJava-Code enthält Syntaxfehler.");
}
```

## Pretty-Print-Methode

Die Methode

```java
public static String prettyPrint(String sourceCode, int indentWidth)
```

führt beide Schritte zusammen:

1. Parse-Tree erzeugen
2. Visitor auf dem Parse-Tree ausführen

Konzeptionell:

```java
MiniJavaParser.CompilationUnitContext tree = parse(sourceCode);

PrettyPrinterVisitor visitor = new PrettyPrinterVisitor(indentWidth);
visitor.visit(tree);

return visitor.result();
```

## Visitor-Prinzip

`PrettyPrinterVisitor` erweitert:

```java
MiniJavaBaseVisitor<Void>
```

Der Visitor traversiert den Parse-Tree. Währenddessen schreibt er Text in einen internen `StringBuilder`.

Der Visitor ist zustandsbehaftet und speichert unter anderem:

```java
private final StringBuilder out = new StringBuilder();
private final int indentWidth;
private int currentIndent = 0;
private boolean atLineStart = true;
private Token lastToken = null;
```

Diese Variablen haben folgende Bedeutung:

| Variable | Bedeutung |
|---|---|
| `out` | enthält die erzeugte Ausgabe |
| `indentWidth` | Anzahl Leerzeichen pro Einrückstufe |
| `currentIndent` | aktuelle Einrücktiefe |
| `atLineStart` | merkt, ob gerade am Anfang einer neuen Zeile geschrieben wird |
| `lastToken` | wird für einfache Leerzeichen-Heuristik zwischen Tokens verwendet |

## Hilfsmethoden zur Ausgabe

Die Ausgabe wird über Hilfsmethoden gesteuert:

| Methode | Aufgabe |
|---|---|
| `write(String s)` | schreibt Text und fügt bei Bedarf vorher Einrückung ein |
| `nl()` | schreibt einen Zeilenumbruch |
| `writeln(String s)` | schreibt Text und danach einen Zeilenumbruch |
| `indent()` | fügt Einrückungsleerzeichen am Zeilenanfang ein |
| `result()` | liefert die fertige Ausgabe als String |

Die Einrückung erfolgt mit Leerzeichen. Tabs werden nicht verwendet.

## Implementierte Visitor-Methoden

Die Aufgabe verlangt die Implementierung dieser vier Methoden:

```java
visitCompilationUnit
visitClassBody
visitBlock
visitStatement
```

Diese Methoden behandeln die wichtigsten strukturellen Bestandteile des Programms.

---

## `visitCompilationUnit`

Die Methode `visitCompilationUnit` behandelt das gesamte Programm.

Sie berücksichtigt:

- optionale Package-Deklaration
- Import-Deklarationen
- Typdeklarationen, insbesondere Klassen

Die Methode sorgt außerdem für Leerzeilen zwischen größeren Abschnitten, zum Beispiel zwischen Package, Imports und Klassen.

Konzeptionell:

```java
@Override
public Void visitCompilationUnit(MiniJavaParser.CompilationUnitContext ctx) {
    // package besuchen
    // imports besuchen
    // type declarations besuchen
    return null;
}
```

Dadurch wird nicht einfach blind `visitChildren(ctx)` verwendet, sondern die oberste Programmstruktur etwas kontrollierter formatiert.

---

## `visitClassBody`

Die Methode `visitClassBody` behandelt den Inhalt einer Klasse.

Grammatikregel:

```antlr
classBody: '{' classBodyDeclaration* '}' ;
```

Formatierungsregeln:

- Vor beziehungsweise bei der öffnenden Klammer wird `{` ausgegeben.
- Nach `{` folgt ein Zeilenumbruch.
- Der Klasseninhalt wird eine Stufe eingerückt.
- Jede Deklaration wird besucht.
- Die Einrückung wird vor der schließenden Klammer wieder reduziert.
- Die schließende Klammer `}` steht auf eigener Zeile.

Beispiel:

```java
class Test {
    private String name;
    public String getName() {
        return name;
    }
}
```

---

## `visitBlock`

Die Methode `visitBlock` behandelt Blöcke.

Grammatikregel:

```antlr
block: '{' blockStatement* '}' ;
```

Blöcke kommen unter anderem vor bei:

- Methodenrümpfen
- `if`
- `else`
- `while`
- verschachtelten Blöcken

Formatierungsregeln:

- `{` wird ausgegeben.
- Danach folgt ein Zeilenumbruch.
- Der Blockinhalt wird eine Stufe eingerückt.
- Alle `blockStatement`-Kinder werden besucht.
- Danach wird die Einrückung wieder reduziert.
- `}` steht auf eigener Zeile.

Beispiel:

```java
while (condition) {
    return "loop";
}
```

---

## `visitStatement`

Die Methode `visitStatement` behandelt verschiedene Arten von Statements.

Grammatikregel:

```antlr
statement: block
         | 'return' expression? ';'
         | 'if' '(' expression ')' statement ('else' statement)?
         | 'while' '(' expression ')' statement
         | expression ';'
         ;
```

Die Methode unterscheidet unter anderem:

- Block-Statements
- Return-Statements
- If-/Else-Statements
- While-Statements
- Ausdrucks-Statements

### Return-Statements

Return-Statements werden in einer Zeile ausgegeben:

```java
return name;
```

Falls ein Ausdruck vorhanden ist, wird dieser nach `return` besucht.

### If-/Else-Statements

Bei `if` wird zuerst die Bedingung ausgegeben. Danach wird das zugehörige Statement besucht.

Wenn das Statement ein Block ist, ergibt sich zum Beispiel:

```java
if (condition) {
    return "yes";
} else {
    return "no";
}
```

### While-Statements

Bei `while` wird die Bedingung ausgegeben und danach der Körper formatiert:

```java
while (condition) {
    return "loop";
}
```

### Ausdrucks-Statements

Ausdrucks-Statements werden besucht und mit `;` abgeschlossen:

```java
x = y;
```

---

## Token-Ausgabe und Leerzeichen-Heuristik

Nicht jede Parser-Regel besitzt eine eigene `visit...`-Methode. Für viele Bestandteile reicht die Standard-Traversierung.

Terminals werden über `visitTerminal` ausgegeben. Dabei wird eine einfache Leerzeichen-Heuristik verwendet:

```java
private boolean needsSpaceBetween(int prevType, int curType) {
    return isWordLike(prevType) && isWordLike(curType);
}
```

Zwischen zwei „wortartigen“ Tokens wird ein Leerzeichen eingefügt. Dadurch entstehen zum Beispiel sinnvolle Ausgaben wie:

```java
public String getName
```

Statt:

```java
publicStringgetName
```

Die Heuristik ist bewusst einfach. Eine perfekte Formatierung von Ausdrücken ist nicht Ziel der Aufgabe.

---

## Konsolenbenutzung

Die Klasse

```java
highlighting.antlr.PrettyPrinterDemo
```

fragt auf der Konsole nach der gewünschten Anzahl von Leerzeichen pro Einrückstufe:

```java
System.out.print("Anzahl Leerzeichen pro Einrückstufe eingeben, z.B. 2, 4 oder 8: ");
int indentWidth = scanner.nextInt();
```

Danach wird diese Zahl an den Pretty Printer übergeben:

```java
String prettyPrinted = MiniJavaPrettyPrinter.prettyPrint(sourceCode, indentWidth);
```

Die Demo enthält mehrere Beispiele unterschiedlicher Komplexität:

1. einfache Klasse mit Feld und Methode
2. Kontrollstrukturen mit `if`/`else` und `while`
3. verschachtelte Blöcke

---

# Demonstrationsbeispiele

Die folgenden Beispiele zeigen, welche Art von Ausgabe der Pretty Printer erzeugen soll.

## Beispiel 1: Einfache Klasse mit Feld und Methode

Eingabe:

```java
class Test{private String name;public String getName(){return name;}}
```

Mögliche Ausgabe bei Einrückbreite 4:

```java
class Test {
    private String name;
    public String getName() {
        return name;
    }
}
```

## Beispiel 2: `if`/`else`

Eingabe:

```java
class Control{public String run(){if(condition){return "yes";}else{return "no";}}}
```

Mögliche Ausgabe:

```java
class Control {
    public String run() {
        if (condition) {
            return "yes";
        } else {
            return "no";
        }
    }
}
```

## Beispiel 3: `while`

Eingabe:

```java
class Control{public String loop(){while(condition){return "loop";}}}
```

Mögliche Ausgabe:

```java
class Control {
    public String loop() {
        while (condition) {
            return "loop";
        }
    }
}
```

## Beispiel 4: Verschachtelte Blöcke

Eingabe:

```java
class Nested{public String test(){{{return "deep";}}}}
```

Mögliche Ausgabe:

```java
class Nested {
    public String test() {
        {
            {
                return "deep";
            }
        }
    }
}
```

---

# Beobachtung: Warum fehlen bestimmte Teile aus dem Input?

Bei der Ausgabe des Pretty Printers können bestimmte Bestandteile des ursprünglichen Eingabetextes fehlen oder anders aussehen.

Typische Beispiele:

- ursprüngliche Leerzeichen
- ursprüngliche Zeilenumbrüche
- Kommentare
- exakte manuelle Formatierung aus der Eingabe

Der Grund ist, dass der Pretty Printer nicht den ursprünglichen Text kopiert. Stattdessen traversiert er den Parse-Tree. Der Parse-Tree enthält vor allem syntaktisch relevante Struktur.

Whitespace wird in der Grammatik übersprungen:

```antlr
WS : [ \t\r\n]+ -> skip;
```

Dadurch taucht Whitespace nicht mehr als normaler Knoten im Parse-Tree auf.

Kommentare liegen in der Grammatik auf dem versteckten Kanal:

```antlr
LINE_COMMENT    : '//' ~[\r\n]*             -> channel(HIDDEN);
JAVADOC_COMMENT : '/**' (.|[\r\n])*? '*/'   -> channel(HIDDEN);
BLOCK_COMMENT   : '/*'  (.|[\r\n])*? '*/'   -> channel(HIDDEN);
```

Diese Tokens sind zwar für das Syntaxhighlighting noch im Tokenstream verfügbar, werden aber vom Parser beziehungsweise Visitor nicht wie normale Programmknoten besucht. Deshalb erscheinen Kommentare im Pretty-Printer-Ergebnis nicht automatisch.

Die Ausgabe ist also keine exakte Rekonstruktion der Eingabe, sondern eine neu generierte, konsistent formatierte Darstellung des syntaktischen Programms.

---

# 2. Cycle Chronicles

## Projektstruktur

Im Cycle-Chronicles-Projekt befinden sich die relevanten Dateien an diesen Stellen:

```text
src/main/java/cyclechronicles/Shop.java
src/main/java/cyclechronicles/Order.java
src/main/java/cyclechronicles/Type.java
src/test/java/ShopAcceptTest.java
```

Die Testklasse liegt direkt unter:

```text
src/test/java/ShopAcceptTest.java
```

Sie befindet sich deshalb im Default Package und importiert die Produktivklassen explizit:

```java
import cyclechronicles.Order;
import cyclechronicles.Shop;
import cyclechronicles.Type;
```

Das Projekt verwendet JUnit Jupiter und Mockito. In `build.gradle` sind unter anderem diese Test-Abhängigkeiten vorhanden:

```gradle
testImplementation platform('org.junit:junit-bom:6.0.3')
testImplementation 'org.junit.jupiter:junit-jupiter'
testImplementation 'org.mockito:mockito-core:5.23.0'
```

Mockito wird zusätzlich als Java-Agent eingebunden:

```gradle
jvmArgs "-javaagent:${configurations.mockitoAgent.asPath}"
```

---

# Aufgabe 2.1: Äquivalenzklassenbildung und Grenzwertanalyse

## Zu testende Methode

Getestet wird die Methode:

```java
public boolean accept(Order o)
```

aus der Klasse:

```java
cyclechronicles.Shop
```

Die Methode entscheidet, ob ein neuer Reparaturauftrag angenommen wird.

Die Implementierung prüft:

```java
if (o.getBicycleType() == Type.GRAVEL) return false;
if (o.getBicycleType() == Type.EBIKE) return false;
if (pendingOrders.stream().anyMatch(x -> x.getCustomer().equals(o.getCustomer()))) return false;
if (pendingOrders.size() > 4) return false;

return pendingOrders.add(o);
```

Ein Auftrag wird also nur angenommen, wenn alle Bedingungen erfüllt sind:

1. Das Fahrrad ist kein Gravel-Bike.
2. Das Fahrrad ist kein E-Bike.
3. Der Kunde hat noch keinen offenen Auftrag.
4. Es sind aktuell höchstens vier andere offene Aufträge vorhanden.

Wird der Auftrag angenommen, wird er hinten in die Warteschlange `pendingOrders` eingefügt.

## Eingabefaktoren

Für die Testanalyse sind diese Faktoren relevant:

| Faktor | Beschreibung |
|---|---|
| Fahrradtyp | Wert von `o.getBicycleType()` |
| Kunde | Wert von `o.getCustomer()` |
| vorhandene offene Aufträge desselben Kunden | Prüfung über `pendingOrders.stream().anyMatch(...)` |
| Anzahl offener Aufträge | Größe der Queue `pendingOrders` vor dem neuen Auftrag |

---

## Äquivalenzklassen für den Fahrradtyp

Der Fahrradtyp ist ein zentraler Eingabefaktor. Die Enum `Type` enthält:

```java
RACE,
SINGLE_SPEED,
FIXIE,
GRAVEL,
EBIKE
```

Daraus ergeben sich folgende Äquivalenzklassen:

| Äquivalenzklasse | Werte | Erwartung |
|---|---|---|
| erlaubter Fahrradtyp | `RACE`, `SINGLE_SPEED`, `FIXIE` | Auftrag kann angenommen werden, wenn auch die anderen Bedingungen erfüllt sind |
| verbotener Fahrradtyp: Gravel | `GRAVEL` | Auftrag wird abgelehnt |
| verbotener Fahrradtyp: E-Bike | `EBIKE` | Auftrag wird abgelehnt |

`GRAVEL` und `EBIKE` werden getrennt betrachtet, weil sie in der Implementierung durch zwei separate Bedingungen geprüft werden.

---

## Äquivalenzklassen für den Kunden

Die Methode erlaubt maximal einen offenen Auftrag pro Kunde.

| Äquivalenzklasse | Beschreibung | Erwartung |
|---|---|---|
| Kunde hat keinen offenen Auftrag | In `pendingOrders` befindet sich kein Auftrag mit gleichem Kundennamen | Auftrag kann angenommen werden |
| Kunde hat bereits einen offenen Auftrag | In `pendingOrders` befindet sich bereits ein Auftrag mit gleichem Kundennamen | Auftrag wird abgelehnt |

Wichtig ist: Nicht die Objektidentität des `Order`-Objekts ist entscheidend, sondern der Rückgabewert von `getCustomer()`.

Zwei verschiedene `Order`-Objekte mit demselben Kundennamen gelten also als Aufträge desselben Kunden.

---

## Äquivalenzklassen für die Anzahl offener Aufträge

Die Warteschlange darf zu jeder Zeit maximal fünf offene Aufträge enthalten.  
Deshalb darf ein weiterer Auftrag nur angenommen werden, wenn vor dem Aufruf höchstens vier Aufträge vorhanden sind.

| Äquivalenzklasse | Anzahl offener Aufträge vor `accept` | Erwartung |
|---|---:|---|
| freie Kapazität | 0 bis 4 | Annahme möglich |
| Warteschlange voll | 5 oder mehr | Annahme nicht möglich |

---

## Grenzwertanalyse

Der relevante Grenzwert liegt bei der Kapazität der Warteschlange.

Maximal erlaubt sind fünf offene Aufträge.  
Da der neue Auftrag erst noch hinzugefügt werden soll, darf die Warteschlange vor dem Aufruf höchstens vier Aufträge enthalten.

| Anzahl offener Aufträge vor dem Aufruf | Bedeutung | Erwartung |
|---:|---|---|
| 0 | leere Warteschlange | Auftrag kann angenommen werden |
| 4 | letzter freier Platz | Auftrag kann angenommen werden |
| 5 | Warteschlange bereits voll | Auftrag wird abgelehnt |

Besonders wichtig sind also die Fälle `4` und `5`, da sie direkt an der Grenze liegen.

---

# Konkrete Testfälle

Aus den Äquivalenzklassen und Grenzwerten ergeben sich folgende Testfälle:

| Nr. | Testfall | Fahrradtyp | Kunde bereits offen? | Offene Aufträge vorher | Erwartetes Ergebnis |
|---:|---|---|---|---:|---|
| 1 | Rennrad-Auftrag bei leerer Warteschlange | `RACE` | nein | 0 | `true` |
| 2 | Singlespeed-Auftrag bei leerer Warteschlange | `SINGLE_SPEED` | nein | 0 | `true` |
| 3 | Fixie-Auftrag bei leerer Warteschlange | `FIXIE` | nein | 0 | `true` |
| 4 | Gravel-Bike-Auftrag | `GRAVEL` | nein | 0 | `false` |
| 5 | E-Bike-Auftrag | `EBIKE` | nein | 0 | `false` |
| 6 | Zweiter Auftrag desselben Kunden | gültiger Typ | ja | 1 | `false` |
| 7 | Neuer Auftrag bei vier offenen Aufträgen | gültiger Typ | nein | 4 | `true` |
| 8 | Neuer Auftrag bei fünf offenen Aufträgen | gültiger Typ | nein | 5 | `false` |

Diese Testfälle decken die relevanten gültigen und ungültigen Äquivalenzklassen sowie die wichtigsten Grenzwerte ab.

---

# Aufgabe 2.2: Mocking mit JUnit und Mockito

## Problem

Die Klasse `Order` ist in der Vorgabe nicht vollständig implementiert:

```java
public Type getBicycleType() {
    throw new UnsupportedOperationException();
}

public String getCustomer() {
    throw new UnsupportedOperationException();
}
```

Die Methode `Shop#accept` ruft aber genau diese Methoden auf:

```java
o.getBicycleType()
o.getCustomer()
```

Wenn echte `Order`-Objekte verwendet würden, würden die Tests mit einer `UnsupportedOperationException` abbrechen.

## Lösung

Die Tests verwenden Mockito, um `Order`-Objekte zu mocken.  
Dadurch können für jeden Test kontrollierte Rückgabewerte festgelegt werden.

Die Testklasse enthält dafür eine Hilfsmethode:

```java
private static Order order(Type type, String customer) {
    Order order = mock(Order.class);

    when(order.getBicycleType()).thenReturn(type);
    when(order.getCustomer()).thenReturn(customer);

    return order;
}
```

Damit kann ein Testauftrag kompakt erzeugt werden:

```java
Order order = order(Type.RACE, "Alice");
```

## Warum wird `Shop` nicht gemockt?

Die Aufgabe verlangt ausdrücklich, dass die zu testende Methode `Shop#accept` nicht weg-gemockt wird.

Deshalb wird im Test immer ein echtes `Shop`-Objekt erzeugt:

```java
Shop shop = new Shop();
```

Auf diesem echten Objekt wird die echte Methode getestet:

```java
shop.accept(order)
```

Nur die abhängige Klasse `Order` wird gemockt, weil sie in der Vorgabe noch keine echte Implementierung besitzt.

## Begründung für Mockito

Mockito ist hier sinnvoll, weil:

- `Order` noch unvollständig ist.
- die Tests trotzdem `Shop#accept` isoliert prüfen sollen.
- die Rückgabewerte von `getBicycleType()` und `getCustomer()` gezielt kontrolliert werden müssen.
- keine eigene Fake-Unterklasse oder Hilfsimplementierung von `Order` geschrieben werden muss.

Die Tests prüfen also nicht die Klasse `Order`, sondern ausschließlich das Verhalten von `Shop#accept`.

---

## Implementierte Tests

Die Datei

```text
src/test/java/ShopAcceptTest.java
```

enthält diese Testmethoden:

| Testmethode | Geprüfter Fall |
|---|---|
| `acceptsRaceBikeWhenQueueIsEmpty()` | `RACE` wird bei leerer Queue angenommen |
| `acceptsSingleSpeedBikeWhenQueueIsEmpty()` | `SINGLE_SPEED` wird bei leerer Queue angenommen |
| `acceptsFixieBikeWhenQueueIsEmpty()` | `FIXIE` wird bei leerer Queue angenommen |
| `rejectsGravelBike()` | `GRAVEL` wird abgelehnt |
| `rejectsEBike()` | `EBIKE` wird abgelehnt |
| `rejectsOrderWhenCustomerAlreadyHasPendingOrder()` | zweiter Auftrag desselben Kunden wird abgelehnt |
| `acceptsOrderWhenFourOrdersAreAlreadyPending()` | fünfter Auftrag wird noch angenommen |
| `rejectsOrderWhenFiveOrdersAreAlreadyPending()` | sechster Auftrag wird abgelehnt |

Damit entsprechen die implementierten Tests den in Aufgabe 2.1 hergeleiteten Testfällen.

---

## Beispiel: Test für bereits vorhandenen Kunden

Der Test für doppelte Kunden funktioniert so:

1. Es wird ein echter Shop erzeugt.
2. Ein erster Auftrag für `"Alice"` wird angenommen.
3. Ein zweiter Auftrag für `"Alice"` wird erstellt.
4. Der zweite Auftrag wird abgelehnt.

Sinngemäß:

```java
Shop shop = new Shop();

Order firstOrder = order(Type.RACE, "Alice");
Order secondOrder = order(Type.FIXIE, "Alice");

assertTrue(shop.accept(firstOrder));
assertFalse(shop.accept(secondOrder));
```

Der Fahrradtyp des zweiten Auftrags ist gültig.  
Die Ablehnung erfolgt also nicht wegen des Fahrradtyps, sondern wegen des bereits offenen Auftrags desselben Kunden.

---

## Beispiel: Grenzwert fünf offene Aufträge

Für den Grenzwert werden zunächst fünf gültige Aufträge mit unterschiedlichen Kunden angenommen:

```java
assertTrue(shop.accept(order(Type.RACE, "Customer 1")));
assertTrue(shop.accept(order(Type.SINGLE_SPEED, "Customer 2")));
assertTrue(shop.accept(order(Type.FIXIE, "Customer 3")));
assertTrue(shop.accept(order(Type.RACE, "Customer 4")));
assertTrue(shop.accept(order(Type.SINGLE_SPEED, "Customer 5")));
```

Danach wird ein sechster Auftrag versucht:

```java
Order sixthOrder = order(Type.FIXIE, "Customer 6");

assertFalse(shop.accept(sixthOrder));
```

Dieser Auftrag hat einen erlaubten Fahrradtyp und einen neuen Kunden.  
Er wird also nur wegen der vollen Warteschlange abgelehnt.

---

## Testausführung

Die Tests können mit Gradle ausgeführt werden:

```bash
./gradlew test
```

Unter Windows je nach Shell:

```bash
gradlew test
```

oder:

```bash
.\gradlew test
```

Ein erfolgreicher Testlauf endet mit:

```text
BUILD SUCCESSFUL
```

Im hochgeladenen Projekt befindet sich bereits ein Gradle-Testbericht zu `ShopAcceptTest`.  
Die Datei

```text
build/test-results/test/TEST-ShopAcceptTest.xml
```

enthält:

```xml
<testsuite name="ShopAcceptTest" tests="8" skipped="0" failures="0" errors="0">
```

Das zeigt, dass die acht Testfälle in diesem gespeicherten Testlauf erfolgreich waren.


# Fazit

Die ANTLR-Aufgabe zeigt, dass Syntaxhighlighting zuverlässiger umgesetzt werden kann, wenn die Tokenisierung nicht über lose reguläre Ausdrücke, sondern über eine Grammatik erfolgt. Der ANTLR-Lexer erkennt Strings, Kommentare, Keywords und andere Tokens systematisch. Dadurch entstehen deutlich weniger Konflikte zwischen Highlight-Regionen.

Beim Pretty Printing wird der Parse-Tree verwendet, um das Programm strukturiert neu auszugeben. Dabei gehen ursprüngliche Leerzeichen, Zeilenumbrüche und Kommentare teilweise verloren, weil der Pretty Printer nicht den Originaltext kopiert, sondern aus der syntaktischen Struktur eine neue Ausgabe erzeugt.

Die Cycle-Chronicles-Aufgabe zeigt, wie aus Anforderungen systematisch Äquivalenzklassen, Grenzwerte und konkrete Testfälle abgeleitet werden. Mockito wird eingesetzt, weil `Order` noch unvollständig implementiert ist. Die Methode `Shop#accept` selbst wird dabei nicht gemockt, sondern echt getestet.
