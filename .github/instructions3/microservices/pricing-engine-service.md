# Pricing Engine Service - MAIP

## Overview
Calculates pricing for PoS products using Drools rule engine. Handles state taxes, customer discounts, bundle discounts, and competitor product discounts.

## Details
- **Package:** `com.maip.pricingengine`
- **Port:** 8084
- **Database:** `maip_{env}_pricing_engine`
- **Rule Engine:** Drools 8.x
- **OpenTelemetry Name:** `maip-pricing-engine-service`

## Kafka Topics

| Topic | Type | Description |
|-------|------|-------------|
| maip.pricing.calculated | Producer | Price calculation completed |
| maip.pricing.rule-updated | Producer | Pricing rule changed |
| maip.product.created | Consumer | Sync product data |
| maip.customer.updated | Consumer | Sync customer discounts |

## API Endpoints

```
POST /api/v1/pricing/calculate     # Calculate price for products
POST /api/v1/pricing/simulate      # Simulate pricing scenarios
GET  /api/v1/pricing/rules         # List active rules
GET  /api/v1/pricing/rules/{id}    # Get rule details
```

## Drools Rule Files

Location: `pricing-engine-service/pricing-engine-rules/src/main/resources/rules/`

```
rules/
├── state-pricing.drl        # Base pricing by state
├── state-tax.drl            # State tax rates
├── customer-discount.drl    # Customer-specific discounts
├── product-discount.drl     # Product-specific discounts
├── bundle-discount.drl      # Multi-product bundle discounts
└── competitor-discount.drl  # Discounts for owning competitor products
```

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

rule "Bundle Discount - Terminal + Accessories"
    when
        $calc : PriceCalculation(hasTerminal == true, accessoryCount >= 2)
    then
        $calc.applyDiscount("TERMINAL_ACCESSORY_BUNDLE", 0.10);
        update($calc);
end
```

### Existing Customer Discount
```drools
rule "Existing Customer - Has MAIP Products"
    when
        $calc : PriceCalculation()
        $customer : Customer(existingMaipProductCount > 0) from $calc.customer
    then
        $calc.applyDiscount("EXISTING_CUSTOMER", 0.05);
        update($calc);
end

rule "Competitor Product - Migration Discount"
    when
        $calc : PriceCalculation()
        $customer : Customer(competitorProductCount > 0) from $calc.customer
    then
        $calc.applyDiscount("COMPETITOR_MIGRATION", 0.12);
        update($calc);
end
```

## Build & Run

```bash
cd backend/microservices/pricing-engine-service

# Build with rules
./mvnw clean verify

# Test rules only
./mvnw test -pl pricing-engine-rules -Dtest=PricingRulesTest

# Run locally
./mvnw spring-boot:run -pl pricing-engine-service-impl \
  -Dspring.profiles.active=local
```

## Deploy to EKS

```bash
kubectl apply -f DevOps/Kubernetes/services/pricing-engine/ -n maip-services
kubectl rollout status deployment/pricing-engine-service -n maip-services
```

## Key Classes

```java
// Fact object for Drools
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

// Request/Response
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

## Logs

```bash
kubectl logs -f -l app=pricing-engine-service -n maip-services --tail=100
```
