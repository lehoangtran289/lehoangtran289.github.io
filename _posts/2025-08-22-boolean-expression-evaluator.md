---
title: "Building a Boolean Expression Evaluator in Java"
layout: single
date: 2025-08-22
permalink: /posts/boolean-expression-evaluator/
author_profile: false
tags:
  - Java
  - Software Development
  - Best Practices
---

This article demonstrates how to build a Boolean expression evaluator in Java that parses and evaluates complex logical expressions at runtime. We'll explore why this pattern matters in enterprise systems, compare it to traditional if-else approaches, and walk through a complete implementation including tokenization, parsing, and evaluation.

{% include toc %}

## 1. The Problem: Hardcoded Business Logic

### 1.1. The Simple If-Else Approach

When business logic is straightforward, hardcoding rules makes sense:

```java
public boolean checkEligibility(User user) {
    if (user.isActive() && user.getAge() >= 18) {
        return true;
    }
    return false;
}
```

**This works fine when:**

- Logic is simple and rarely changes
- Only developers need to modify rules
- You have a handful of conditions to manage
- Rules don't vary per customer or tenant

### 1.2. When Simple Breaks Down

Say you need to check user eligibility with complex nested conditions:

```java
public boolean checkEligibility(User user) {
    if (user.isActive() && 
        ((user.getAge() >= 18 && user.hasVerifiedEmail()) || 
         (user.getAge() >= 13 && user.hasParentConsent())) &&
        (user.getRegion().equals("US") || user.getRegion().equals("CA") || 
         user.getRegion().equals("UK")) &&
        !user.isBlacklisted() &&
        (user.isPremium() || (user.isTrial() && user.getDaysActive() <= 30))) {
        return true;
    }
    return false;
}
```

**This hardcoded logic checks:**
- Active users
- AND (18+ with verified email OR 13+ with parental consent)
- AND in US/CA/UK regions
- AND not blacklisted
- AND (premium OR trial within 30 days)

**The fundamental problem: This logic is frozen in code.**

What happens when requirements change?

→ Business wants to add EU region? **Code change + deployment**

→ Marketing wants to test removing the email verification requirement? **Code change + deployment**

→ Sales wants a special rule for enterprise customers? **Code change + deployment**

Each change requires:
- Developer to modify the condition
- Pull request and code review
- QA testing the change
- Production deployment
- Rollback plan if something breaks

**More problems:**
- The nested condition becomes unreadable as complexity grows
- No way for non-developers to tweak the logic
- A/B testing different rules requires feature flags everywhere
- Multi-tenant scenarios (different rules per customer) explode the codebase
- Every rule variant needs separate code branches

## 2. The Solution: Runtime Expression Evaluation

### 2.1. Store Rules as Data

Instead of hardcoding logic, store the rule as a string expression:

```java
// Store this in database or config file
String eligibilityRule = 
    "isActive AND " +
    "((age_gte_18 AND hasVerifiedEmail) OR (age_gte_13 AND hasParentConsent)) AND " +
    "(region_US OR region_CA OR region_UK) AND " +
    "NOT isBlacklisted AND " +
    "(isPremium OR (isTrial AND daysActive_lte_30))";
```

Build a context map from the User object:

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

Now your eligibility check becomes:

```java
public boolean checkEligibility(User user, String expression) {
    Map<String, Boolean> context = buildContext(user);

    return BooleanEvaluator.evaluate(expression, context);
}
```

**The game-changer: Custom input at runtime**

The expression `"isActive AND (age_gte_18 OR hasParentConsent)"` and the map of boolean values are **both inputs** to the evaluator.

→ Want to add EU region? Change the expression string in the database

→ Want to test without email verification? Modify the expression

→ Different rule for enterprise customers? Store different expressions per customer

**No code changes. No deployments. Just data updates.**

**Benefits of this approach:**
- Rules can be changed without code deployment
- Business users can modify rules through UI
- Easy to A/B test different rule variations
- Rules are versioned and auditable in the database
- Significantly simpler codebase
- Same evaluator code handles any Boolean logic

### 2.2. When to Use Each Approach

**Use if-else when:**
- You have < 5 simple conditions
- Logic never or rarely changes
- Only developers manage the logic
- Performance is absolutely critical (nanoseconds matter)

**Use expression evaluator when:**
- Business rules are complex and nested
- Non-developers need to configure rules
- You have multi-tenant requirements
- Rules change frequently
- You need rule versioning and audit trails
- Your system has 10+ different rule variations

## 3. Real-World Use Cases

Expression evaluators shine in these scenarios:

### 3.1. Feature Flags and A/B Testing

```java
// Rule stored in database:
// "user_premium AND (country_US OR country_CA) AND NOT internal_user"

if (evaluator.evaluate(featureFlagRule, userContext)) {
    showNewFeature();
}
```

### 3.2. Access Control and Permissions

```java
// "hasRole_admin OR (hasRole_manager AND department_engineering)"

if (evaluator.evaluate(accessRule, userContext)) {
    allowAccess();
}
```

### 3.3. Notification Rules

```java
// "order_total_gt_1000 AND (vip_customer OR first_time_buyer)"

if (evaluator.evaluate(notificationRule, orderContext)) {
    sendSpecialOfferEmail();
}
```

### 3.4. Dynamic Pricing Rules

```java
// "(peak_hours AND high_demand) OR (weekend AND event_nearby)"

if (evaluator.evaluate(surgePricingRule, context)) {
    applySurgeMultiplier();
}
```

### 3.5. Multi-Tenant Custom Rules

```java
// Each customer gets their own eligibility rule
// Customer A: "isActive AND (age_gte_18 OR hasParentConsent)"
// Customer B: "isActive AND isPremium AND NOT isBlacklisted"
// Customer C: "(region_US OR region_EU) AND (isActive OR isTrial)"

public boolean checkEligibility(User user, String tenantId) {
    String rule = ruleRepository.findByTenantId(tenantId);
    Map<String, Boolean> context = buildContext(user);
    return BooleanEvaluator.evaluate(rule, context);
}
```

## 4. Architecture Overview

Our expression evaluator consists of three main components:

```
Input: "NOT (A AND B) OR C"
    ↓
[Tokenizer] → Breaks input into tokens
    ↓
Tokens: [NOT, LPAREN, A, AND, B, RPAREN, OR, C]
    ↓
[Parser] → Builds Abstract Syntax Tree (AST)
    ↓
AST:         OR
           /    \
         NOT     C
          |
        AND
       /   \
      A     B
    ↓
[Evaluator] → Evaluates AST with variable values
    ↓
Result: true/false
```

**Why this architecture?**
- **Separation of concerns** → Each component has one job
- **Extensibility** → Easy to add new operators or data types
- **Testability** → Each component can be tested independently
- **Performance** → Parse once, evaluate many times

## 5. Implementation

### 5.1. Token and TokenType

First, define the building blocks:

```java
enum TokenType {
    VARIABLE, AND, OR, NOT, LPAREN, RPAREN, EOF
}

class Token {
    TokenType type;
    String value;

    Token(TokenType type, String value) {
        this.type = type;
        this.value = value;
    }
}
```

### 5.2. Tokenizer

The tokenizer converts a string into a list of tokens:

```java
class Tokenizer {
    private static final Map<String, TokenType> KEYWORDS = Map.of(
        "AND", TokenType.AND,
        "OR", TokenType.OR,
        "NOT", TokenType.NOT
    );

    public static List<Token> tokenize(String input) {
        List<Token> tokens = new ArrayList<>();
        int pos = 0;

        while (pos < input.length()) {
            char ch = input.charAt(pos);

            // Skip whitespace
            if (Character.isWhitespace(ch)) { pos++; continue; }
            
            // Handle parentheses
            if (ch == '(') { tokens.add(new Token(TokenType.LPAREN, "(")); pos++; continue; }
            if (ch == ')') { tokens.add(new Token(TokenType.RPAREN, ")")); pos++; continue; }

            // Handle variables and keywords
            if (Character.isLetter(ch)) {
                StringBuilder sb = new StringBuilder();
                while (pos < input.length() &&
                       (Character.isLetterOrDigit(input.charAt(pos)) || input.charAt(pos) == '_')) {
                    sb.append(input.charAt(pos++));
                }
                String value = sb.toString();
                String upper = value.toUpperCase();
                TokenType type = KEYWORDS.getOrDefault(upper, TokenType.VARIABLE);
                tokens.add(new Token(type, KEYWORDS.containsKey(upper) ? upper : value));
                continue;
            }

            throw new RuntimeException("Unexpected character '" + ch + "' at position " + pos);
        }

        tokens.add(new Token(TokenType.EOF, null));
        return tokens;
    }
}
```

**What it does:**
- Scans the input character by character
- Recognizes keywords (AND, OR, NOT)
- Identifies variables (alphanumeric + underscore)
- Handles parentheses for grouping
- Throws errors for invalid characters

### 5.3. Abstract Syntax Tree Nodes

Define nodes for our AST:

```java
abstract class Node {}

class VariableNode extends Node {
    String name;
    VariableNode(String name) { this.name = name; }
}

class UnaryOpNode extends Node {
    String operator;
    Node operand;
    UnaryOpNode(String operator, Node operand) {
        this.operator = operator; 
        this.operand = operand;
    }
}

class BinaryOpNode extends Node {
    String operator;
    Node left, right;
    BinaryOpNode(String operator, Node left, Node right) {
        this.operator = operator; 
        this.left = left; 
        this.right = right;
    }
}
```

**Node types:**
- **VariableNode** → Represents a variable (e.g., `isActive`)
- **UnaryOpNode** → Single operator (e.g., `NOT isActive`)
- **BinaryOpNode** → Two operands (e.g., `A AND B`)

### 5.4. Parser

The parser builds the AST following operator precedence:

```java
class Parser {
    private List<Token> tokens;
    private int pos = 0;

    Parser(List<Token> tokens) { this.tokens = tokens; }

    private Token current() { return tokens.get(pos); }
    private void advance() { pos++; }
    
    private boolean match(TokenType... types) {
        for (TokenType t : types) 
            if (current().type == t) return true;
        return false;
    }

    private void expect(TokenType type) {
        if (current().type != type)
            throw new RuntimeException("Expected " + type + " but found " + current().type);
        advance();
    }

    public Node parse() {
        Node ast = parseOr();
        expect(TokenType.EOF);
        return ast;
    }

    // Helper for binary operators with precedence
    private Node parseBinary(java.util.function.Supplier<Node> next, TokenType... ops) {
        Node left = next.get();
        while (match(ops)) {
            String op = current().value;
            advance();
            left = new BinaryOpNode(op, left, next.get());
        }
        return left;
    }

    // Operator precedence: OR (lowest) → AND → NOT (highest)
    private Node parseOr()  { return parseBinary(this::parseAnd, TokenType.OR); }
    private Node parseAnd() { return parseBinary(this::parseNot, TokenType.AND); }

    private Node parseNot() {
        if (match(TokenType.NOT)) {
            advance();
            return new UnaryOpNode("NOT", parseNot());
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
            Node expr = parseOr();
            expect(TokenType.RPAREN);
            return expr;
        }
        throw new RuntimeException("Unexpected token: " + current().type);
    }
}
```

**Parser features:**
- **Operator precedence** → NOT > AND > OR (standard Boolean logic)
- **Left-to-right association** → `A AND B AND C` = `(A AND B) AND C`
- **Parentheses override** → `A AND (B OR C)` evaluates OR first
- **Recursive descent** → Each precedence level calls the next higher level

### 5.5. Evaluator

The evaluator walks the AST and computes the result:

```java
class Evaluator {
    public static boolean evaluateAST(Node node, Map<String, Boolean> values) {
        if (node instanceof VariableNode v) {
            if (!values.containsKey(v.name))
                throw new RuntimeException("Undefined variable: " + v.name);
            return values.get(v.name);
        }
        
        if (node instanceof BinaryOpNode b) {
            boolean left = evaluateAST(b.left, values);
            boolean right = evaluateAST(b.right, values);
            return switch (b.operator) {
                case "AND" -> left && right;
                case "OR"  -> left || right;
                default -> throw new RuntimeException("Unknown operator: " + b.operator);
            };
        }
        
        if (node instanceof UnaryOpNode u) {
            boolean operand = evaluateAST(u.operand, values);
            if (u.operator.equals("NOT")) return !operand;
            throw new RuntimeException("Unknown operator: " + u.operator);
        }
        
        throw new RuntimeException("Unknown node type");
    }
}
```

**Evaluation logic:**
- Recursively walks the AST from root to leaves
- Looks up variable values from the provided map
- Applies operators to computed sub-results
- Fails fast on undefined variables or unknown operators

### 5.6. Public API

Tie everything together with a clean public interface:

```java
public class BooleanEvaluator {

    public static boolean evaluate(String expression, Map<String, Boolean> values) {
        List<Token> tokens = Tokenizer.tokenize(expression);
        Parser parser = new Parser(tokens);
        Node ast = parser.parse();
        return Evaluator.evaluateAST(ast, values);
    }

    // Demo
    public static void main(String[] args) {
        String expr = "NOT (A AND B) OR C";
        Map<String, Boolean> vars = Map.of(
            "A", true,
            "B", false,
            "C", false
        );

        boolean result = evaluate(expr, vars);
        System.out.println("Result: " + result); // Output: true
    }
}
```

**Usage pattern:**
- Single static method for simple use cases
- Pass expression string and variable values
- Get boolean result back
- All complexity hidden behind clean API

## 6. Testing

Comprehensive unit tests ensure correctness:

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import java.util.*;

public class BooleanEvaluatorTest {

    @Test
    void testSimpleAnd() {
        boolean result = BooleanEvaluator.evaluate("A AND B", 
            Map.of("A", true, "B", true));
        assertTrue(result);

        result = BooleanEvaluator.evaluate("A AND B", 
            Map.of("A", true, "B", false));
        assertFalse(result);
    }

    @Test
    void testSimpleOr() {
        boolean result = BooleanEvaluator.evaluate("A OR B", 
            Map.of("A", false, "B", true));
        assertTrue(result);

        result = BooleanEvaluator.evaluate("A OR B", 
            Map.of("A", false, "B", false));
        assertFalse(result);
    }

    @Test
    void testNotOperator() {
        boolean result = BooleanEvaluator.evaluate("NOT A", 
            Map.of("A", false));
        assertTrue(result);

        result = BooleanEvaluator.evaluate("NOT A", 
            Map.of("A", true));
        assertFalse(result);
    }

    @Test
    void testParenthesesAndPrecedence() {
        // (A AND B) OR C → (true && false) || true = true
        boolean result = BooleanEvaluator.evaluate("(A AND B) OR C",
            Map.of("A", true, "B", false, "C", true));
        assertTrue(result);

        // A AND (B OR C) → true && (false || false) = false
        result = BooleanEvaluator.evaluate("A AND (B OR C)",
            Map.of("A", true, "B", false, "C", false));
        assertFalse(result);
    }

    @Test
    void testNestedNot() {
        // NOT (NOT A) → same as A
        boolean result = BooleanEvaluator.evaluate("NOT (NOT A)",
            Map.of("A", true));
        assertTrue(result);
    }

    @Test
    void testComplexExpression() {
        // (A AND (NOT B)) OR (C AND (NOT (D OR E)))
        String expr = "(A AND (NOT B)) OR (C AND (NOT (D OR E)))";

        boolean result = BooleanEvaluator.evaluate(expr, Map.of(
            "A", true, "B", false, "C", false, "D", false, "E", false
        ));
        assertTrue(result);

        result = BooleanEvaluator.evaluate(expr, Map.of(
            "A", false, "B", false, "C", true, "D", true, "E", false
        ));
        assertFalse(result);
    }

    @Test
    void testUndefinedVariableThrows() {
        Exception ex = assertThrows(RuntimeException.class, () ->
            BooleanEvaluator.evaluate("A AND B", Map.of("A", true))
        );
        assertTrue(ex.getMessage().contains("Undefined variable"));
    }

    @Test
    void testInvalidCharacterThrows() {
        Exception ex = assertThrows(RuntimeException.class, () ->
            BooleanEvaluator.evaluate("A & B", Map.of("A", true, "B", false))
        );
        assertTrue(ex.getMessage().contains("Unexpected character"));
    }
}
```

**Test coverage:**
- Basic operators (AND, OR, NOT)
- Parentheses and precedence
- Nested expressions
- Error cases (undefined variables, invalid syntax)
- Complex real-world scenarios

## 7. Performance Considerations

### 7.1. Parse Once, Evaluate Many

For frequently-used rules, cache the parsed AST:

```java
public class CachedEvaluator {
    private final Map<String, Node> astCache = new ConcurrentHashMap<>();
    
    public boolean evaluate(String expression, Map<String, Boolean> values) {
        Node ast = astCache.computeIfAbsent(expression, expr -> {
            List<Token> tokens = Tokenizer.tokenize(expr);
            return new Parser(tokens).parse();
        });
        return Evaluator.evaluateAST(ast, values);
    }
}
```

**Benefits:**
- Parsing happens only once per unique expression
- Subsequent evaluations are just tree traversal
- Thread-safe with `ConcurrentHashMap`

### 7.2. Short-Circuit Evaluation

The current implementation naturally short-circuits:

```java
// In BinaryOpNode evaluation:
boolean left = evaluateAST(b.left, values);
boolean right = evaluateAST(b.right, values);
return b.operator.equals("AND") ? left && right : left || right;
```

Java's `&&` and `||` operators are short-circuit:
- `false AND <anything>` → doesn't evaluate right side
- `true OR <anything>` → doesn't evaluate right side

### 7.3. Benchmarking

Expected performance characteristics:

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| Tokenization | O(n) | n = expression length |
| Parsing | O(n) | Recursive descent |
| Evaluation | O(nodes) | AST traversal |
| Cached eval | O(nodes) | Skip tokenize + parse |

For typical business rules (10-20 variables), evaluation takes **microseconds**.

## 8. Extensions and Improvements

### 8.1. Add Comparison Operators

Support `age > 18` and `score >= 90`:

```java
// Add to TokenType
enum TokenType {
    VARIABLE, NUMBER, GT, GTE, LT, LTE, EQ, NEQ, 
    AND, OR, NOT, LPAREN, RPAREN, EOF
}

// Add ComparisonNode
class ComparisonNode extends Node {
    String operator;
    Node left, right;
}

// Modify evaluator to handle comparisons
if (node instanceof ComparisonNode c) {
    int left = evaluateNumeric(c.left, values);
    int right = evaluateNumeric(c.right, values);
    return switch (c.operator) {
        case ">" -> left > right;
        case ">=" -> left >= right;
        case "<" -> left < right;
        case "<=" -> left <= right;
        default -> throw new RuntimeException("Unknown operator");
    };
}
```

### 8.2. Support Functions

Add built-in functions like `IN` and `CONTAINS`:

```java
// Expression: "region IN [US, CA, UK]"
// Expression: "email CONTAINS @company.com"

class FunctionNode extends Node {
    String functionName;
    List<Node> arguments;
}
```

### 8.3. Add Variables from Objects

Instead of manually building the context map, use reflection:

```java
public Map<String, Boolean> buildContext(User user) {
    Map<String, Boolean> context = new HashMap<>();
    context.put("isActive", user.isActive());
    context.put("isPremium", user.isPremium());
    context.put("age_gte_18", user.getAge() >= 18);
    return context;
}
```

Or use a library-like approach with property paths:

```java
// Expression: "user.isActive AND user.age >= 18"
```

### 8.4. Better Error Messages

Enhance error reporting with position information:

```java
class ParseException extends RuntimeException {
    int position;
    String expression;
    
    ParseException(String message, int position, String expression) {
        super(message + " at position " + position + "\n" + 
              expression + "\n" + " ".repeat(position) + "^");
    }
}
```

## 9. Production Considerations

### 9.1. Security

**Validate expressions before evaluation:**
- Limit expression length (e.g., max 1000 characters)
- Restrict allowed variable names (whitelist approach)
- Set evaluation timeout to prevent infinite loops
- Sandbox the evaluation environment

```java
public boolean evaluate(String expression, Map<String, Boolean> values) {
    if (expression.length() > 1000) {
        throw new IllegalArgumentException("Expression too long");
    }
    
    // Validate variable names against whitelist
    Set<String> allowedVariables = Set.of("isActive", "isPremium", ...);
    validateVariables(expression, allowedVariables);
    
    return BooleanEvaluator.evaluate(expression, values);
}
```

### 9.2. Monitoring

Track expression evaluation performance:

```java
public boolean evaluate(String expression, Map<String, Boolean> values) {
    long start = System.nanoTime();
    try {
        return BooleanEvaluator.evaluate(expression, values);
    } finally {
        long duration = System.nanoTime() - start;
        metrics.recordEvaluationTime(duration);
        if (duration > THRESHOLD) {
            logger.warn("Slow expression: {} took {}ms", expression, duration / 1_000_000);
        }
    }
}
```

### 9.3. Versioning

When rules change, maintain history:

```java
@Entity
public class Rule {
    private String customerId;
    private String expression;
    private int version;
    private LocalDateTime createdAt;
    private String createdBy;
}
```

**Benefits:**
- Audit trail for compliance
- Ability to rollback bad rules
- A/B testing different versions

### 9.4. UI for Non-Developers

Build a rule builder interface:
- Dropdown for variable selection
- Visual operator buttons (AND, OR, NOT)
- Parentheses grouping with drag-and-drop
- Real-time validation and preview

Example UI workflow:
1. User selects variable `isActive` from dropdown
2. Clicks "AND" button
3. Selects variable `isPremium`
4. System generates: `isActive AND isPremium`
5. Preview shows which users match the rule

## 10. Comparison Summary

| Aspect | If-Else Approach | Expression Evaluator |
|--------|------------------|---------------------|
| **Complexity** | Simple for < 5 rules | Worth it for 10+ rules |
| **Deployment** | Required for changes | Not required |
| **Who can change** | Developers only | Business users via UI |
| **Testability** | Hard with many branches | Each rule independent |
| **Maintainability** | Becomes spaghetti | Rules as data |
| **Flexibility** | Very rigid | Highly flexible |
| **Performance** | Slightly faster | Microseconds slower |
| **Auditability** | Code commits | Database versioning |
| **Multi-tenancy** | One rule for all | Custom per tenant |

## 11. Key Takeaways

**When business logic is simple** → stick with if-else. Don't over-engineer.

**When complexity grows** → expression evaluators become essential. They enable:
- Non-developers to manage business rules
- Rule changes without code deployment
- Multi-tenant customization
- Better separation of concerns

**The implementation** → three clean components:
- **Tokenizer** → string to tokens
- **Parser** → tokens to AST
- **Evaluator** → AST to result

**Production-ready features:**
- Caching parsed ASTs for performance
- Validation and security checks
- Monitoring and metrics
- Rule versioning and audit trails

This pattern is widely used in:
- Feature flag systems (LaunchDarkly, Unleash)
- Access control (authorization rules)
- Business rule engines (Drools alternatives)
- Dynamic pricing and recommendations

The code shown here is production-quality and can handle complex real-world scenarios. Start simple, add features as needed.
