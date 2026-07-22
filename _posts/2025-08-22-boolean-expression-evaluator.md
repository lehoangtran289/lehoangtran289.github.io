---
title: "Build a Boolean Expression Evaluator in Java"
layout: single
date: 2025-08-22
permalink: /posts/boolean-expression-evaluator/
author_profile: false
tags:
  - Java
  - Software Development
  - Best Practices
---

Business rules often start as simple `if` statements. That is usually fine, because hardcoded logic is fast, easy to debug, and checked by the compiler. The problem starts when the rule changes more often than the codebase, or when different customers, cohorts, regions, or experiments need different versions of the same decision logic.

At that point, every rule change becomes a deployment. A product manager asks to add a region, marketing wants to test a different eligibility path, sales needs an enterprise override, and engineering ends up threading feature flags through a method that was supposed to be a clean predicate. The code still runs, but it no longer represents a stable algorithm. It represents mutable business policy trapped inside compiled Java.

A Boolean expression evaluator moves that policy into data. The application still controls which variables are available and how those variables are computed, but the rule itself can be stored, validated, versioned, cached, and evaluated at runtime.

This post builds a small Boolean expression evaluator in Java. It supports variables, `AND`, `OR`, `NOT`, and parentheses. The implementation uses a tokenizer, a recursive descent parser, an abstract syntax tree, and an evaluator with real short-circuit behavior.

## 1. The Problem with Hardcoded Rules

A simple eligibility check is a good fit for normal Java code.

```java
public boolean checkEligibility(User user) {
    return user.isActive() && user.getAge() >= 18;
}
```

This is readable, fast, and hard to misuse. The rule is close to the domain model, the compiler checks the method, and tests can cover the few branches that exist.

Now consider a more realistic rule.

```java
public boolean checkEligibility(User user) {
    return user.isActive()
            && ((user.getAge() >= 18 && user.hasVerifiedEmail())
                || (user.getAge() >= 13 && user.hasParentConsent()))
            && ("US".equals(user.getRegion())
                || "CA".equals(user.getRegion())
                || "UK".equals(user.getRegion()))
            && !user.isBlacklisted()
            && (user.isPremium()
                || (user.isTrial() && user.getDaysActive() <= 30));
}
```

The method is still correct, but it now combines several concerns:

* extracting facts from `User`
* encoding business policy
* controlling Boolean operator precedence
* handling future rule variants through code changes

The immediate cost is readability. The larger cost is operational. Adding a new region, changing the trial window, removing email verification for an experiment, or introducing a tenant-specific override requires a code change, a review, a test cycle, and a deployment. If the system serves multiple tenants, this pattern usually turns into duplicated methods, nested feature flags, or rule-specific branches scattered across services.

A runtime evaluator separates the stable part from the volatile part. Java code computes named facts, while a stored expression decides how those facts are combined.

## 2. Rules as Runtime Data

Instead of compiling the whole rule into Java, store the Boolean expression as a string.

```java
String eligibilityRule =
        "isActive AND "
      + "((age_gte_18 AND hasVerifiedEmail) OR (age_gte_13 AND hasParentConsent)) AND "
      + "(region_US OR region_CA OR region_UK) AND "
      + "NOT isBlacklisted AND "
      + "(isPremium OR (isTrial AND daysActive_lte_30))";
```

The application builds a context map from the domain object.

```java
public Map<String, Boolean> buildContext(User user) {
    Map<String, Boolean> context = new HashMap<>();

    context.put("isActive", user.isActive());
    context.put("age_gte_18", user.getAge() >= 18);
    context.put("age_gte_13", user.getAge() >= 13);
    context.put("hasVerifiedEmail", user.hasVerifiedEmail());
    context.put("hasParentConsent", user.hasParentConsent());
    context.put("region_US", "US".equals(user.getRegion()));
    context.put("region_CA", "CA".equals(user.getRegion()));
    context.put("region_UK", "UK".equals(user.getRegion()));
    context.put("isBlacklisted", user.isBlacklisted());
    context.put("isPremium", user.isPremium());
    context.put("isTrial", user.isTrial());
    context.put("daysActive_lte_30", user.getDaysActive() <= 30);

    return context;
}
```

The service method becomes small again.

```java
public boolean checkEligibility(User user, String expression) {
    Map<String, Boolean> context = buildContext(user);
    return BooleanEvaluator.evaluate(expression, context);
}
```

This design makes one boundary explicit: the evaluator does not know about `User`, regions, subscriptions, trials, or blacklists. It only knows how to evaluate a Boolean expression against named Boolean values. That constraint keeps the evaluator reusable and reduces the risk of turning it into an unbounded scripting layer.

The trade-off is also clear. You lose compile-time validation for the expression string, so you must replace it with save-time validation, good error messages, safe rollout, and monitoring. Runtime rules are useful when policy changes frequently, but they need guardrails because bad rules fail at runtime unless you catch them earlier.

## 3. Use Cases

Boolean expression evaluators are useful when a system needs configurable decision logic but does not need a full rule engine.

Feature flags are a common example.

```java
String rule = "isEmployee AND country_US AND NOT blockedUser";

if (BooleanEvaluator.evaluate(rule, context)) {
    showFeature();
}
```

Authorization checks are another natural fit, especially when policies differ between tenants or environments.

```java
String rule = "hasRole_admin OR (hasRole_manager AND department_engineering)";

if (BooleanEvaluator.evaluate(rule, context)) {
    allowAccess();
}
```

Notification systems can use the same pattern to decide whether an event should trigger a message.

```java
String rule = "order_total_gt_1000 AND (vipCustomer OR firstPurchase)";

if (BooleanEvaluator.evaluate(rule, context)) {
    sendOffer();
}
```

The evaluator should remain narrow. If the business needs numeric comparison, string matching, list membership, date arithmetic, or custom functions, those features can be added, but each one expands the language surface and increases the cost of validation, testing, and abuse prevention.

## 4. Evaluator Architecture

The evaluator has three stages.

```text
Input:
  "NOT (A AND B) OR C"

Tokenizer:
  [NOT, LPAREN, A, AND, B, RPAREN, OR, C, EOF]

Parser:
  builds an abstract syntax tree

AST:
        OR
       /  \
     NOT   C
      |
     AND
    /   \
   A     B

Evaluator:
  walks the tree and reads values from the context map
```

The tokenizer turns raw characters into tokens. The parser turns tokens into an abstract syntax tree, usually called an AST. The evaluator walks that tree and computes the final Boolean result.

This architecture gives each component one job. Tokenization errors are about invalid characters. Parser errors are about invalid grammar. Evaluation errors are about missing variables or unsupported operators. That separation matters in production because different failures need different responses: invalid syntax should block rule activation, while missing variables usually indicate a mismatch between the rule catalog and the application’s context builder.

The grammar is small.

```text
expression    := orExpression
orExpression  := andExpression ("OR" andExpression)*
andExpression := notExpression ("AND" notExpression)*
notExpression := "NOT" notExpression | primary
primary       := VARIABLE | "(" expression ")"
```

This gives `NOT` the highest precedence, followed by `AND`, then `OR`. Parentheses override that order.

## 5. Implementation

The implementation below uses a hand-written recursive descent parser. For this grammar, that is simpler than introducing a parser generator, and it makes precedence easy to inspect in code.

### 5.1. Token Model

Each token has a type, value, and source position. Position is not needed for evaluation, but it is useful for error messages.

```java
enum TokenType {
    VARIABLE,
    AND,
    OR,
    NOT,
    LPAREN,
    RPAREN,
    EOF
}

final class Token {
    final TokenType type;
    final String value;
    final int position;

    Token(TokenType type, String value, int position) {
        this.type = type;
        this.value = value;
        this.position = position;
    }

    @Override
    public String toString() {
        return value == null ? type.name() : type + "(" + value + ")";
    }
}
```

The language reserves `AND`, `OR`, and `NOT`. Variable names are case-sensitive in this implementation, while keywords are case-insensitive.

### 5.2. Parse Exception

A runtime expression language needs precise errors. Returning only `Invalid expression` is cheap for the implementation, but expensive for whoever has to debug a rejected rule.

```java
final class ExpressionParseException extends RuntimeException {
    ExpressionParseException(String message, int position, String expression) {
        super(format(message, position, expression));
    }

    private static String format(String message, int position, String expression) {
        return message + " at position " + position
                + System.lineSeparator()
                + expression
                + System.lineSeparator()
                + " ".repeat(Math.max(0, position)) + "^";
    }
}
```

For example, `A AND (B OR)` can point directly to the token position where another operand was expected.

### 5.3. Tokenizer

The tokenizer scans the input once. It skips whitespace, emits parentheses, reads identifiers, and maps reserved keywords to operator tokens.

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Locale;
import java.util.Map;

final class Tokenizer {
    private static final Map<String, TokenType> KEYWORDS = Map.of(
            "AND", TokenType.AND,
            "OR", TokenType.OR,
            "NOT", TokenType.NOT
    );

    private Tokenizer() {
    }

    static List<Token> tokenize(String input) {
        if (input == null) {
            throw new IllegalArgumentException("Expression must not be null");
        }

        List<Token> tokens = new ArrayList<>();
        int pos = 0;

        while (pos < input.length()) {
            char ch = input.charAt(pos);

            if (Character.isWhitespace(ch)) {
                pos++;
                continue;
            }

            if (ch == '(') {
                tokens.add(new Token(TokenType.LPAREN, "(", pos));
                pos++;
                continue;
            }

            if (ch == ')') {
                tokens.add(new Token(TokenType.RPAREN, ")", pos));
                pos++;
                continue;
            }

            if (Character.isLetter(ch)) {
                int start = pos;
                StringBuilder builder = new StringBuilder();

                while (pos < input.length()) {
                    char current = input.charAt(pos);

                    if (Character.isLetterOrDigit(current) || current == '_') {
                        builder.append(current);
                        pos++;
                    } else {
                        break;
                    }
                }

                String rawValue = builder.toString();
                String upperValue = rawValue.toUpperCase(Locale.ROOT);
                TokenType type = KEYWORDS.getOrDefault(upperValue, TokenType.VARIABLE);
                String tokenValue = type == TokenType.VARIABLE ? rawValue : upperValue;

                tokens.add(new Token(type, tokenValue, start));
                continue;
            }

            throw new ExpressionParseException(
                    "Unexpected character '" + ch + "'",
                    pos,
                    input
            );
        }

        tokens.add(new Token(TokenType.EOF, null, input.length()));
        return tokens;
    }
}
```

This version intentionally rejects Java-style operators such as `&&`, `||`, and `!`. Supporting aliases is easy, but doing so often makes rule authoring less consistent. A small language is easier to validate, document, and expose through a UI.

### 5.4. AST Nodes

The AST has three node types.

```java
abstract class Node {
}

final class VariableNode extends Node {
    final String name;

    VariableNode(String name) {
        this.name = name;
    }
}

final class UnaryOpNode extends Node {
    final String operator;
    final Node operand;

    UnaryOpNode(String operator, Node operand) {
        this.operator = operator;
        this.operand = operand;
    }
}

final class BinaryOpNode extends Node {
    final String operator;
    final Node left;
    final Node right;

    BinaryOpNode(String operator, Node left, Node right) {
        this.operator = operator;
        this.left = left;
        this.right = right;
    }
}
```

The nodes are immutable, which makes them safe to cache and share across threads. That property becomes important once parsed expressions are reused across requests.

### 5.5. Parser

The parser implements operator precedence by splitting the grammar into methods. `parseOr` handles the lowest-precedence operator, `parseAnd` handles the next level, `parseNot` handles unary negation, and `parsePrimary` handles variables and parentheses.

```java
import java.util.List;
import java.util.function.Supplier;

final class Parser {
    private final List<Token> tokens;
    private final String expression;
    private int pos = 0;

    Parser(List<Token> tokens, String expression) {
        this.tokens = tokens;
        this.expression = expression;
    }

    Node parse() {
        Node ast = parseOr();
        expect(TokenType.EOF);
        return ast;
    }

    private Node parseOr() {
        return parseBinary(this::parseAnd, TokenType.OR);
    }

    private Node parseAnd() {
        return parseBinary(this::parseNot, TokenType.AND);
    }

    private Node parseNot() {
        if (match(TokenType.NOT)) {
            Token token = current();
            advance();
            return new UnaryOpNode(token.value, parseNot());
        }

        return parsePrimary();
    }

    private Node parsePrimary() {
        if (match(TokenType.VARIABLE)) {
            String name = current().value;
            advance();
            return new VariableNode(name);
        }

        if (match(TokenType.LPAREN)) {
            advance();
            Node node = parseOr();
            expect(TokenType.RPAREN);
            return node;
        }

        Token token = current();
        throw new ExpressionParseException(
                "Expected variable or '(' but found " + token.type,
                token.position,
                expression
        );
    }

    private Node parseBinary(Supplier<Node> nextPrecedence, TokenType operatorType) {
        Node left = nextPrecedence.get();

        while (match(operatorType)) {
            String operator = current().value;
            advance();

            Node right = nextPrecedence.get();
            left = new BinaryOpNode(operator, left, right);
        }

        return left;
    }

    private Token current() {
        return tokens.get(pos);
    }

    private boolean match(TokenType type) {
        return current().type == type;
    }

    private void advance() {
        pos++;
    }

    private void expect(TokenType type) {
        if (!match(type)) {
            Token token = current();

            throw new ExpressionParseException(
                    "Expected " + type + " but found " + token.type,
                    token.position,
                    expression
            );
        }

        advance();
    }
}
```

The final `expect(TokenType.EOF)` is important. Without it, the parser might accept a valid prefix and ignore trailing junk. For example, `A AND B C` should fail, not silently evaluate as `A AND B`.

### 5.6. Evaluator

The evaluator walks the AST recursively and reads variable values from the provided map.

```java
import java.util.Map;

final class Evaluator {
    private Evaluator() {
    }

    static boolean evaluateAST(Node node, Map<String, Boolean> values) {
        if (node instanceof VariableNode variable) {
            Boolean value = values.get(variable.name);

            if (value == null) {
                throw new IllegalArgumentException("Undefined variable: " + variable.name);
            }

            return value;
        }

        if (node instanceof UnaryOpNode unary) {
            if ("NOT".equals(unary.operator)) {
                return !evaluateAST(unary.operand, values);
            }

            throw new IllegalArgumentException("Unknown unary operator: " + unary.operator);
        }

        if (node instanceof BinaryOpNode binary) {
            return switch (binary.operator) {
                case "AND" -> evaluateAST(binary.left, values)
                        && evaluateAST(binary.right, values);

                case "OR" -> evaluateAST(binary.left, values)
                        || evaluateAST(binary.right, values);

                default -> throw new IllegalArgumentException(
                        "Unknown binary operator: " + binary.operator
                );
            };
        }

        throw new IllegalArgumentException("Unknown AST node: " + node.getClass().getName());
    }
}
```

The placement of the recursive calls matters. This implementation short-circuits correctly because the right-hand side is evaluated inside Java’s `&&` or `||` expression.

This means:

```text
false AND missingVariable -> false
true OR missingVariable   -> true
```

The evaluator does not read `missingVariable` in either case. If the code first evaluated both sides into local variables, short-circuiting would be lost, and these expressions would throw.

Whether this behavior is desirable depends on the system. Java developers usually expect short-circuit semantics, but a rule-management platform should still validate all referenced variables when a rule is created. That catches bad expressions before they enter production traffic.

### 5.7. Public API

The public API combines tokenization, parsing, and evaluation.

```java
import java.util.List;
import java.util.Map;

public final class BooleanEvaluator {
    private BooleanEvaluator() {
    }

    public static boolean evaluate(String expression, Map<String, Boolean> values) {
        if (values == null) {
            throw new IllegalArgumentException("Values map must not be null");
        }

        List<Token> tokens = Tokenizer.tokenize(expression);
        Node ast = new Parser(tokens, expression).parse();

        return Evaluator.evaluateAST(ast, values);
    }

    public static void main(String[] args) {
        String expression = "NOT (A AND B) OR C";

        Map<String, Boolean> values = Map.of(
                "A", true,
                "B", false,
                "C", false
        );

        boolean result = BooleanEvaluator.evaluate(expression, values);
        System.out.println(result);
    }
}
```

For `NOT (A AND B) OR C`, the result is `true` when `A = true`, `B = false`, and `C = false`.

## 6. Testing

The tests should cover operator behavior, precedence, parentheses, malformed expressions, undefined variables, and short-circuit behavior. The short-circuit tests are worth keeping because they prevent a common refactor bug.

```java
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class BooleanEvaluatorTest {

    @Test
    void evaluatesAnd() {
        assertTrue(BooleanEvaluator.evaluate(
                "A AND B",
                Map.of("A", true, "B", true)
        ));

        assertFalse(BooleanEvaluator.evaluate(
                "A AND B",
                Map.of("A", true, "B", false)
        ));
    }

    @Test
    void evaluatesOr() {
        assertTrue(BooleanEvaluator.evaluate(
                "A OR B",
                Map.of("A", false, "B", true)
        ));

        assertFalse(BooleanEvaluator.evaluate(
                "A OR B",
                Map.of("A", false, "B", false)
        ));
    }

    @Test
    void evaluatesNot() {
        assertTrue(BooleanEvaluator.evaluate(
                "NOT A",
                Map.of("A", false)
        ));

        assertFalse(BooleanEvaluator.evaluate(
                "NOT A",
                Map.of("A", true)
        ));
    }

    @Test
    void respectsPrecedence() {
        assertTrue(BooleanEvaluator.evaluate(
                "A OR B AND C",
                Map.of("A", true, "B", false, "C", false)
        ));

        assertFalse(BooleanEvaluator.evaluate(
                "A AND B OR C",
                Map.of("A", true, "B", false, "C", false)
        ));
    }

    @Test
    void respectsParentheses() {
        assertFalse(BooleanEvaluator.evaluate(
                "A AND (B OR C)",
                Map.of("A", true, "B", false, "C", false)
        ));

        assertTrue(BooleanEvaluator.evaluate(
                "(A AND B) OR C",
                Map.of("A", true, "B", false, "C", true)
        ));
    }

    @Test
    void evaluatesNestedExpression() {
        String expression = "(A AND (NOT B)) OR (C AND (NOT (D OR E)))";

        assertTrue(BooleanEvaluator.evaluate(
                expression,
                Map.of(
                        "A", true,
                        "B", false,
                        "C", false,
                        "D", false,
                        "E", false
                )
        ));

        assertFalse(BooleanEvaluator.evaluate(
                expression,
                Map.of(
                        "A", false,
                        "B", false,
                        "C", true,
                        "D", true,
                        "E", false
                )
        ));
    }

    @Test
    void shortCircuitsAnd() {
        assertFalse(BooleanEvaluator.evaluate(
                "A AND missing",
                Map.of("A", false)
        ));
    }

    @Test
    void shortCircuitsOr() {
        assertTrue(BooleanEvaluator.evaluate(
                "A OR missing",
                Map.of("A", true)
        ));
    }

    @Test
    void throwsForUndefinedVariableWhenEvaluated() {
        IllegalArgumentException exception = assertThrows(
                IllegalArgumentException.class,
                () -> BooleanEvaluator.evaluate("A AND B", Map.of("A", true))
        );

        assertTrue(exception.getMessage().contains("Undefined variable: B"));
    }

    @Test
    void throwsForInvalidCharacter() {
        ExpressionParseException exception = assertThrows(
                ExpressionParseException.class,
                () -> BooleanEvaluator.evaluate("A & B", Map.of("A", true, "B", false))
        );

        assertTrue(exception.getMessage().contains("Unexpected character"));
    }

    @Test
    void throwsForUnclosedParenthesis() {
        ExpressionParseException exception = assertThrows(
                ExpressionParseException.class,
                () -> BooleanEvaluator.evaluate(
                        "A AND (B OR C",
                        Map.of("A", true, "B", false, "C", true)
                )
        );

        assertTrue(exception.getMessage().contains("Expected RPAREN"));
    }
}
```

These tests are not exhaustive, but they cover the areas most likely to break during maintenance. If this evaluator becomes part of a critical path, add randomized tests that generate expression trees, render them into strings, parse them back, and compare parsed evaluation against direct tree evaluation.

## 7. Performance

The evaluator has predictable complexity.

| Stage             | Complexity | Notes                          |
| ----------------- | ---------: | ------------------------------ |
| Tokenization      |     `O(n)` | Scans each character once      |
| Parsing           |     `O(t)` | Consumes each token once       |
| Evaluation        |     `O(k)` | Visits evaluated AST nodes     |
| Cached evaluation |     `O(k)` | Skips tokenization and parsing |

For typical business expressions, parsing usually costs more than evaluation. Evaluation is just a tree walk plus map lookups, and with short-circuiting it may visit only part of the tree.

If the same expression runs many times, cache the parsed AST.

```java
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public final class CachedBooleanEvaluator {
    private final ConcurrentMap<String, Node> astCache = new ConcurrentHashMap<>();

    public boolean evaluate(String expression, Map<String, Boolean> values) {
        Node ast = astCache.computeIfAbsent(expression, this::parse);
        return Evaluator.evaluateAST(ast, values);
    }

    private Node parse(String expression) {
        List<Token> tokens = Tokenizer.tokenize(expression);
        return new Parser(tokens, expression).parse();
    }
}
```

A raw `ConcurrentHashMap` is fine for a demo, but it is not enough for production. If expressions can be created dynamically, an unbounded cache can become a memory leak. Use a bounded cache, record hit rate and eviction count, and validate expressions before they reach the hot path.

Context construction can also dominate runtime. If building the map requires database calls, remote service calls, or expensive computation, optimizing the AST traversal will not fix latency. In that case, either precompute facts upstream or consider lazy variables through `Supplier<Boolean>`, with care around timeouts, retries, and side effects.

## 8. Production Guardrails

A runtime expression evaluator should reject bad rules before they affect requests. The validator should enforce size limits, syntax validity, allowed variables, and any domain-specific restrictions.

```java
import java.util.List;
import java.util.Set;

public final class RuleValidator {
    private static final int MAX_EXPRESSION_LENGTH = 1_000;
    private static final int MAX_TOKEN_COUNT = 200;

    public void validate(String expression, Set<String> allowedVariables) {
        if (expression == null || expression.isBlank()) {
            throw new IllegalArgumentException("Expression must not be blank");
        }

        if (expression.length() > MAX_EXPRESSION_LENGTH) {
            throw new IllegalArgumentException("Expression is too long");
        }

        List<Token> tokens = Tokenizer.tokenize(expression);

        if (tokens.size() > MAX_TOKEN_COUNT) {
            throw new IllegalArgumentException("Expression has too many tokens");
        }

        Node ast = new Parser(tokens, expression).parse();
        validateVariables(ast, allowedVariables);
    }

    private void validateVariables(Node node, Set<String> allowedVariables) {
        if (node instanceof VariableNode variable) {
            if (!allowedVariables.contains(variable.name)) {
                throw new IllegalArgumentException("Variable is not allowed: " + variable.name);
            }

            return;
        }

        if (node instanceof UnaryOpNode unary) {
            validateVariables(unary.operand, allowedVariables);
            return;
        }

        if (node instanceof BinaryOpNode binary) {
            validateVariables(binary.left, allowedVariables);
            validateVariables(binary.right, allowedVariables);
            return;
        }

        throw new IllegalArgumentException("Unknown AST node: " + node.getClass().getName());
    }
}
```

The production concerns are mostly operational rather than algorithmic:

* reject expressions that are too long or too deeply nested
* validate variable names against a rule-specific catalog
* store rule versions instead of mutating rules in place
* support rollback to the previous active version
* log rule ID, version, result, and correlation ID
* record evaluation latency, validation failures, and cache behavior
* test new rules against historical samples before activation

For high-impact decisions, evaluate candidate rules in shadow mode before serving them. The active rule still controls the response, while the candidate rule runs in parallel and emits disagreement metrics. This catches accidental broadening or narrowing before a bad policy starts rejecting users, changing prices, or granting access.

## 9. Extending the Language

The first common extension is comparison syntax.

```text
age >= 18 AND region == "US"
```

That looks small, but it changes the evaluator substantially. The tokenizer must support numbers, strings, and multi-character operators. The parser needs additional precedence levels. The evaluator needs typed values, not only Booleans. Error handling must distinguish syntax errors from type errors.

A larger grammar might look like this:

```text
expression    := orExpression
orExpression  := andExpression ("OR" andExpression)*
andExpression := notExpression ("AND" notExpression)*
notExpression := "NOT" notExpression | comparison
comparison    := value ((">" | ">=" | "<" | "<=" | "==" | "!=") value)?
value         := VARIABLE | NUMBER | STRING | "(" expression ")"
```

At that point, the implementation is becoming a small interpreter. That can still be the right path, especially when the domain requires a constrained language, but it should trigger a build-versus-adopt discussion. Mature expression languages already handle typed values, functions, better diagnostics, and optimization, although they also bring their own security and integration risks.

## 10. When to Use This Pattern

Use normal Java when the rule is simple, stable, and owned by engineers. Hardcoded logic gives strong tooling, compile-time feedback, and predictable performance.

Use a Boolean expression evaluator when the rule is complex enough to need configuration, when tenants need different policies, when rules must be audited, or when the organization needs to change policy faster than the application deployment cycle. The evaluator adds parsing and validation complexity, but it localizes that complexity in one component instead of spreading rule variants across the codebase.

The main design principle is to keep the language small for as long as possible. A Boolean evaluator with well-defined variables is easy to secure and reason about. A general-purpose scripting system embedded in a request path is much harder to operate safely.

## 11. Key Takeaways

A Boolean expression evaluator is a focused interpreter. The tokenizer converts text into tokens, the parser builds an AST, and the evaluator computes a result from request-specific values.

The important implementation details are practical:

* parse the entire expression, including the EOF marker
* preserve positions for useful error messages
* make operator precedence explicit in the parser
* keep AST nodes immutable if they are cached
* implement short-circuiting by evaluating the right side only inside `&&` or `||`
* validate allowed variables before activating a rule
* bound caches and expression size in production

This pattern does not remove business-rule complexity. It moves that complexity into a smaller, testable, observable layer where rule changes can be treated as data changes rather than application releases.
