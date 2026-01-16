# Pricing Engine Service

## Purpose

Calculates pricing for PoS products using Drools rule engine. Handles state taxes, customer discounts, bundle discounts, and competitor migration discounts.

---

## Service Details

| Property | Value |
|----------|-------|
| Port | 8084 |
| Package | `com.maip.pricingengine` |
| Database | `maip_pricing_engine` |
| OpenTelemetry | `maip-pricing-engine-service` |

---

## Kafka Topics

| Topic | Type | Description |
|-------|------|-------------|
| `maip.pricing.calculated` | Producer | Price calculation completed |
| `maip.pricing.rule-updated` | Producer | Pricing rule changed |
| `maip.product.created` | Consumer | Sync product data |
| `maip.customer.updated` | Consumer | Sync customer discounts |

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/pricing/calculate` | Calculate price for products |
| POST | `/api/v1/pricing/simulate` | Simulate pricing scenarios |
| GET | `/api/v1/pricing/rules` | List active rules |
| GET | `/api/v1/pricing/rules/{id}` | Get rule details |

---

## Drools Rule Files

Location: `pricing-engine-service/pricing-engine-rules/src/main/resources/rules/`

| File | Purpose |
|------|---------|
| `state-pricing.drl` | Base pricing by state |
| `state-tax.drl` | State tax rates |
| `customer-discount.drl` | Customer-specific discounts |
| `product-discount.drl` | Product-specific discounts |
| `bundle-discount.drl` | Multi-product bundle discounts |
| `competitor-discount.drl` | Discounts for owning competitor products |

---

## Build Commands

### Build All

```bash
cd backend/microservices/pricing-engine-service
./mvnw clean verify
```

### Build Rules Only

```bash
./mvnw clean verify -pl pricing-engine-rules
```

### Test Rules Only

```bash
./mvnw test -pl pricing-engine-rules -Dtest=PricingRulesTest
```

---

## Run Locally

```bash
cd backend/microservices/pricing-engine-service
./mvnw spring-boot:run -pl pricing-engine-service-impl -Dspring.profiles.active=local
```

---

## Rule Examples

### State Tax Rule

```drools
rule "State Tax - California"
    when
        $calc : PriceCalculation(state == "CA")
    then
        $calc.applyTaxRate(0.0725);
        update($calc);
end

rule "State Tax - Texas"
    when
        $calc : PriceCalculation(state == "TX")
    then
        $calc.applyTaxRate(0.0625);
        update($calc);
end
```

### Bundle Discount Rule

```drools
rule "Bundle Discount - 3+ Terminals"
    when
        $calc : PriceCalculation(terminalCount >= 3)
    then
        $calc.applyDiscount("BUNDLE_3_PLUS", 0.15);
        update($calc);
end
```

### Competitor Migration Rule

```drools
rule "Competitor Product - Migration Discount"
    when
        $calc : PriceCalculation()
        $customer : Customer(competitorProductCount > 0) from $calc.customer
    then
        $calc.applyDiscount("COMPETITOR_MIGRATION", 0.12);
        update($calc);
end
```

---

## Key Classes

### PriceCalculation (Drools Fact)

```java
@Data
public class PriceCalculation {
    private UUID customerId;
    private String state;
    private List<Product> products;
    private int terminalCount;
    private int accessoryCount;
    private boolean hasTerminal;
    private Customer customer;
    
    private BigDecimal subtotal;
    private BigDecimal taxAmount;
    private BigDecimal discountAmount;
    private BigDecimal totalAmount;
    private List<AppliedDiscount> appliedDiscounts;
    
    public void applyTaxRate(double rate) { ... }
    public void applyDiscount(String code, double rate) { ... }
}
```

### Request/Response

```java
@Value
public class PriceRequest {
    UUID customerId;
    String state;
    List<UUID> productIds;
}

@Value @Builder
public class PriceResponse {
    BigDecimal subtotal;
    BigDecimal taxAmount;
    BigDecimal discountAmount;
    BigDecimal totalAmount;
    List<AppliedDiscount> appliedDiscounts;
    List<LineItem> lineItems;
}
```

---

## Adding New Rules

### Step 1: Create Rule File

```bash
touch pricing-engine-rules/src/main/resources/rules/{new-rule}.drl
```

### Step 2: Write Rule

```drools
package com.maip.pricingengine.rules

import com.maip.pricingengine.model.PriceCalculation

rule "New Rule Name"
    when
        $calc : PriceCalculation(/* conditions */)
    then
        $calc.applyDiscount("NEW_DISCOUNT", 0.10);
        update($calc);
end
```

### Step 3: Add Test

```java
@Test
void testNewRule() {
    PriceCalculation calc = new PriceCalculation();
    // setup
    kieSession.insert(calc);
    kieSession.fireAllRules();
    // assertions
}
```

### Step 4: Build and Test

```bash
./mvnw test -pl pricing-engine-rules
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Rule compilation failed` | Syntax error in .drl | Check rule syntax |
| `ClassNotFoundException` | Missing import in rule | Add import statement |
| `No rules fired` | Conditions not met | Debug with `DroolsLogger` |
| `Infinite loop` | Rule modifies and re-triggers | Add `no-loop true` |
