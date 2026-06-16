# Dokumentation: Filter-DSL und AST-Builder

repolink: https://github.com/Baikov2003/prog2_ybel_filterdsl.git
## Aufgabe 1: JUnit-Testfälle

In Aufgabe 1 wurden JUnit-Testfälle für die AST-Erzeugung geschrieben. Ziel war es, wichtige Query-Formen zu überprüfen und sicherzustellen, dass der erzeugte AST der erwarteten Struktur entspricht.

Getestet wurden unter anderem:

- einfache String-Vergleiche
- einfache Zahlenvergleiche
- alle Vergleichsoperatoren
- `in`-Ausdrücke
- `and`-Ausdrücke
- `or`-Ausdrücke
- `not`-Ausdrücke
- Klammerung
- Operatorpräzedenz
- verkettete `and`- und `or`-Ausdrücke
- komplexe Beispiele aus der Aufgabenstellung

Ein Beispiel für einen Testfall:

```java
@Test
void parsesAndExpression() {
    Expr expected = new Expr.And(
            new Expr.Comparison("artist", CompOp.EQ, new Value.Str("Beatles")),
            new Expr.Comparison("year", CompOp.EQ, new Value.Num(1965))
    );

    assertEquals(expected, ast("artist == \"Beatles\" and year == 1965"));
}
```

Da die AST-Klassen als Records modelliert sind, kann `assertEquals` direkt genutzt werden. Die Records vergleichen ihre Inhalte automatisch.

---

## Aufgabe 2: AST-Builder mit dem Visitor-Pattern

In Aufgabe 2 wurde der AST-Builder `AstBuilderVisitor` implementiert. Dieser Builder nutzt das von ANTLR bereitgestellte Visitor-Pattern.

Die Klasse erbt von:

```java
FilterBaseVisitor<Void>
```

Da der Visitor `Void` zurückgibt, wird der AST zustandsbehaftet aufgebaut. Dafür werden mehrere Stacks verwendet:

```java
private final Deque<Expr> exprs = new ArrayDeque<>();
private final Deque<Value> values = new ArrayDeque<>();
private final Deque<List<Value>> valueLists = new ArrayDeque<>();
```

Die Startmethode ist:

```java
public Expr translate(FilterParser.QueryContext ctx)
```

Sie startet die rekursive Verarbeitung des Parse-Trees mit `visit(ctx)` und nimmt am Ende den fertigen AST vom Expression-Stack.

Die wichtigsten Methoden sind:

- `visitQuery`
- `visitExpr`
- `visitOrExpr`
- `visitAndExpr`
- `visitNotExpr`
- `visitPrimary`
- `visitComparison`
- `visitLiteralList`
- `visitLiteral`

Beispielhaft wird bei einem `and`-Ausdruck zuerst der linke Ausdruck besucht, dann der rechte Ausdruck. Danach werden beide Ausdrücke vom Stack genommen und zu einem neuen `Expr.And` verbunden:

```java
Expr right = exprs.pop();
Expr left = exprs.pop();

exprs.push(new Expr.And(left, right));
```

Die Tests wurden anschließend mit dem Visitor-Builder ausgeführt:

```java
private Expr ast(String query) {
    return AstBuilders.fromQuery(query, new AstBuilderVisitor()::translate);
}
```

Der Testlauf war erfolgreich:

```text
BUILD SUCCESSFUL
```

Damit ist der Visitor-basierte AST-Builder funktionsfähig.

---

## Aufgabe 3: AST-Builder mit manueller Traversierung und Pattern Matching

In Aufgabe 3 wurde der zweite AST-Builder `AstBuilderPattern` implementiert. Im Gegensatz zum Visitor arbeitet diese Variante ohne internen Zustand.

Die Startmethode ist ebenfalls:

```java
public Expr translate(FilterParser.QueryContext ctx)
```

Sie ruft rekursiv eigene Hilfsmethoden auf:

```java
return buildExpr(ctx.expr());
```

Die Methoden geben direkt AST-Knoten zurück, zum Beispiel:

```java
private Expr buildAndExpr(FilterParser.AndExprContext ctx)
```

oder:

```java
private Value buildLiteral(FilterParser.LiteralContext ctx)
```

Der AST wird also direkt über Rückgabewerte aufgebaut. Für Entscheidungen über verschiedene Parse-Tree-Strukturen wird Pattern Matching beziehungsweise `switch` verwendet.

Beispiel:

```java
private Expr buildPrimary(FilterParser.PrimaryContext ctx) {
    ParseTree firstChild = ctx.getChild(0);

    return switch (firstChild) {
        case FilterParser.ComparisonContext comparison -> buildComparison(comparison);
        default -> buildExpr(ctx.expr());
    };
}
```

Für Vergleichsausdrücke wird unterschieden, ob ein normaler Vergleich oder ein `in`-Ausdruck vorliegt:

```java
return switch (ctx.getChild(1).getText()) {
    case "in" -> new Expr.InList(field, buildLiteralList(ctx.literalList()));
    default -> new Expr.Comparison(field, CompOp.fromSymbol(ctx.op.getText()), buildLiteral(ctx.value));
};
```

Auch dieser Builder wurde mit den JUnit-Tests überprüft:

```java
private Expr ast(String query) {
    return AstBuilders.fromQuery(query, new AstBuilderPattern()::translate);
}
```

---

## Aufgabe 4: Vergleich Visitor-Pattern vs. Pattern Matching

Beide AST-Builder erzeugen aus dem ANTLR-Parse-Tree denselben AST, unterscheiden sich aber in der Art der Traversierung. Der Visitor-Builder nutzt das von ANTLR bereitgestellte Visitor-Pattern. Das ist gut strukturiert, weil es für jede Grammatikregel eine eigene `visit`-Methode gibt. Außerdem passt dieser Ansatz gut zu ANTLR, da die Basisklassen automatisch generiert werden.

Ein Nachteil des Visitor-Builders ist, dass er zustandsbehaftet arbeitet. Durch die verwendeten Stacks für `Expr`- und `Value`-Objekte ist der Datenfluss weniger direkt sichtbar. Man muss genau auf die Reihenfolge von `push` und `pop` achten, sonst entstehen leicht Fehler.

Der Pattern-Matching-Builder arbeitet ohne internen Zustand. Die Methoden geben direkt `Expr`- oder `Value`-Objekte zurück. Dadurch ist der Code aus meiner Sicht leichter zu lesen und nachzuvollziehen. Besonders bei einfachen Grammatiken ist diese rekursive Lösung sehr übersichtlich.

Der Nachteil dieser Variante ist, dass man die Traversierung des Parse-Trees selbst schreiben muss. Man ist stärker von der konkreten Struktur der Grammatik abhängig und muss bei Änderungen an der Grammatik eventuell mehrere Stellen im Code anpassen.

Insgesamt finde ich für dieses Projekt den Pattern-Matching-Ansatz übersichtlicher, weil der AST direkt über Rückgabewerte aufgebaut wird. Der Visitor-Ansatz ist dafür näher an ANTLR und bei größeren Grammatiken oft systematischer.

---

## Aufgabe 5: AST-Normalisierung mit Pattern Matching

In Aufgabe 5 wurde die Methode

```java
Expr simplify(Expr e)
```

in der Klasse `AstBuilders` vervollständigt. Ziel war es, eine Abbildung von AST nach AST zu schreiben. Dabei wird ein bestehender AST analysiert und gegebenenfalls vereinfacht.

Ein wichtiges Beispiel ist die Eliminierung von doppeltem `not`:

```text
not not artist == "Beatles"
```

wird vereinfacht zu:

```text
artist == "Beatles"
```

Die Implementierung nutzt Pattern Matching und Record-Dekonstruktion:

```java
public static Expr simplify(Expr e) {
    return switch (e) {
        case Expr.Not(Expr.Not(var inner)) -> simplify(inner);

        case Expr.Not(var inner) -> new Expr.Not(simplify(inner));

        case Expr.And(var left, var right) -> {
            Expr simplifiedLeft = simplify(left);
            Expr simplifiedRight = simplify(right);
            yield new Expr.And(simplifiedLeft, simplifiedRight);
        }

        case Expr.Or(var left, var right) -> {
            Expr simplifiedLeft = simplify(left);
            Expr simplifiedRight = simplify(right);
            yield new Expr.Or(simplifiedLeft, simplifiedRight);
        }

        case Expr.Comparison(var field, var op, var value) ->
            new Expr.Comparison(field, op, value);

        case Expr.InList(var field, var values) ->
            new Expr.InList(field, values);
    };
}
```

Durch die sealed AST-Hierarchie kann der `switch` alle möglichen `Expr`-Varianten vollständig behandeln.

---

## Aufgabe 6: Approval Testing

In Aufgabe 6 wurde Approval Testing eingesetzt, um ASTs besser prüfen zu können. Das Problem bei klassischen Unit-Tests ist, dass erwartete ASTs oft sehr verschachtelt von Hand aufgebaut werden müssen.

Beispiel:

```java
var expr =
    new Expr.And(
        new Expr.Comparison("artist", CompOp.EQ, new Value.Str("Beatles")),
        new Expr.Comparison("year", CompOp.EQ, new Value.Num(1965)));
```

Das ist bei größeren Ausdrücken schnell unübersichtlich.

Deshalb wird der AST mit dem vorgegebenen Pretty-Printer in einen String umgewandelt:

```java
AstPrinter.toString(expr)
```

Dieser String wird dann mit ApprovalTests überprüft:

```java
Approvals.verify(renderBuilderOutput("Visitor Builder", this::visitorAst));
```

Es wurden Approval-Tests für beide Builder geschrieben:

- Visitor-Builder
- Pattern-Matching-Builder

Außerdem wurde verglichen, ob beide Builder für dieselben Queries denselben AST beziehungsweise dieselbe Pretty-Print-Ausgabe erzeugen.

Beim ersten Testlauf erzeugt ApprovalTests `.received.txt`-Dateien. Wenn deren Inhalt korrekt ist, werden diese in `.approved.txt` umbenannt. Danach laufen die Tests erfolgreich durch.

---

## Aufgabe 7: Property-based Testing

In Aufgabe 7 wurden Property-based Tests mit jqwik implementiert. Dabei werden nicht nur einzelne feste Beispiele getestet, sondern allgemeine Eigenschaften formuliert, die für viele generierte Eingaben gelten sollen.

Eine zentrale Eigenschaft ist der Roundtrip:

```text
Query
  -> Parsing
  -> AST
  -> Pretty-Print
  -> Parsing
  -> AST
```

Der AST nach dem erneuten Parsen soll dem ursprünglichen AST entsprechen.

Es wurden unter anderem folgende Eigenschaften getestet:

### Roundtrip mit Visitor-Builder

```java
@Property
boolean visitorBuilderSupportsRoundtrip(@ForAll("representativeQueries") String query) {
    Expr firstAst = visitorAst(query);
    String prettyPrinted = AstPrinter.toString(firstAst);
    Expr secondAst = visitorAst(prettyPrinted);

    return firstAst.equals(secondAst);
}
```

### Roundtrip mit Pattern-Builder

```java
@Property
boolean patternBuilderSupportsRoundtrip(@ForAll("representativeQueries") String query) {
    Expr firstAst = patternAst(query);
    String prettyPrinted = AstPrinter.toString(firstAst);
    Expr secondAst = patternAst(prettyPrinted);

    return firstAst.equals(secondAst);
}
```

### Vergleich beider Builder

```java
@Property
boolean visitorAndPatternBuilderProduceSameAst(@ForAll("representativeQueries") String query) {
    Expr visitorAst = visitorAst(query);
    Expr patternAst = patternAst(query);

    return visitorAst.equals(patternAst);
}
```

Zusätzlich wurden logische Eigenschaften von `and` getestet:

- `and` entspricht der Java-Logik `&&`
- `and` ist für die Auswertung kommutativ
- `and` ist für die Auswertung idempotent
- `and` ist für die Auswertung assoziativ

Beispiel:

```java
@Property
boolean andIsCommutativeForEvaluation(
    @ForAll("representativeQueries") String leftQuery,
    @ForAll("representativeQueries") String rightQuery) {

    Expr left = patternAst(leftQuery);
    Expr right = patternAst(rightQuery);

    Expr leftAndRight = new Expr.And(left, right);
    Expr rightAndLeft = new Expr.And(right, left);

    return Evaluator.matches(SAMPLE_ITEM, leftAndRight)
        == Evaluator.matches(SAMPLE_ITEM, rightAndLeft);
}
```

Damit wird nicht nur geprüft, ob einzelne Beispiele funktionieren, sondern ob grundlegende Eigenschaften der AST-Struktur und der Auswertung über viele automatisch generierte Fälle hinweg gelten.

---

## Testausführung

Die Tests können einzeln oder gemeinsam mit Gradle ausgeführt werden.

Alle Tests:

```powershell
./gradlew clean test
```

Nur die klassischen AST-Tests:

```powershell
./gradlew clean test --tests filter.ast.AstTest
```

Nur die Approval-Tests:

```powershell
./gradlew clean test --tests filter.ast.ApprovalTest
```

Nur die Property-based Tests:

```powershell
./gradlew clean test --tests filter.ast.RoundtripPropertiesTest
```

Ein erfolgreicher Lauf endet mit:

```text
BUILD SUCCESSFUL
```

---

## Fazit

Das Projekt zeigt mehrere Möglichkeiten, einen von ANTLR erzeugten Parse-Tree in einen eigenen AST zu überführen. Der Visitor-Builder ist stark an den von ANTLR generierten Klassen orientiert und arbeitet mit internem Zustand. Der Pattern-Matching-Builder ist direkter und baut den AST über Rückgabewerte auf.

Durch klassische Unit-Tests, Approval Testing und Property-based Testing werden unterschiedliche Teststrategien kombiniert:

- Unit-Tests prüfen konkrete erwartete AST-Strukturen.
- Approval Tests erleichtern den Vergleich komplexer AST-Ausgaben.
- Property-based Tests prüfen allgemeine Eigenschaften über viele generierte Eingaben hinweg.
