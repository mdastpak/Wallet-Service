# Money Handling Best Practices

## Overview

Strategies for accurate financial calculations avoiding floating-point precision issues.

---

## Storage Format

**Always use minor units (integer storage)**:
- USD: Store in cents (100 cents = $1.00)
- EUR: Store in cents
- BTC: Store in satoshis (100,000,000 satoshis = 1 BTC)

**Database Type**: `NUMBER(19,0)` (64-bit integer)

---

## Money Class

```java
public final class Money {
    private final long amount;  // Minor units
    private final Currency currency;

    private Money(long amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public static Money of(BigDecimal majorUnits, Currency currency) {
        int scale = currency.getDefaultFractionDigits();
        long minorUnits = majorUnits.movePointRight(scale).longValueExact();
        return new Money(minorUnits, currency);
    }

    public static Money ofMinor(long minorUnits, Currency currency) {
        return new Money(minorUnits, currency);
    }

    public BigDecimal toMajorUnits() {
        int scale = currency.getDefaultFractionDigits();
        return BigDecimal.valueOf(amount).movePointLeft(scale);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException();
        }
        return new Money(this.amount + other.amount, this.currency);
    }

    public Money multiply(BigDecimal factor) {
        long result = BigDecimal.valueOf(amount)
            .multiply(factor)
            .setScale(0, RoundingMode.HALF_UP)
            .longValue();
        return new Money(result, this.currency);
    }
}
```

---

## Examples

### Storing $100.50
```java
Money money = Money.of(new BigDecimal("100.50"), Currency.getInstance("USD"));
// Stored as: 10050 (cents)
```

### Calculation (20% discount)
```java
Money originalPrice = Money.ofMinor(10000, USD);  // $100.00
BigDecimal discountRate = new BigDecimal("0.20");  // 20%
Money discount = originalPrice.multiply(discountRate);  // $20.00
Money finalPrice = originalPrice.subtract(discount);  // $80.00
```

---

## Rounding Rules

- **User-facing amounts**: Round HALF_UP
- **Internal calculations**: Exact (no rounding until final)
- **Tax calculations**: Follow regulatory rounding rules

---

## Currency Handling

```java
@Entity
public class Transaction {

    @Column(nullable = false)
    private long amount;  // Minor units

    @Column(length = 3, nullable = false)
    private String currency;  // ISO 4217 code (USD, EUR, BTC)

    @Transient
    public Money getMoney() {
        return Money.ofMinor(amount, Currency.getInstance(currency));
    }
}
```

---

## Never Use

- ❌ `float` or `double` for money
- ❌ Direct decimal arithmetic without precision control
- ❌ String concatenation for calculations

---

## Always Use

- ✅ Integer storage (minor units)
- ✅ `BigDecimal` for calculations
- ✅ Explicit rounding modes
- ✅ Currency awareness

---

See [Ledger Model](../02-domain-model/ledger-model.md) for financial accounting principles.
