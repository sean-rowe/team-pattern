# Team Pattern Architecture Guide

## 1. Workers

Workers transform state in specific, predictable ways. Each Worker method takes a state object and returns a new one, making a single well-defined transformation. This pure-function approach means every data change is explicit and trackable - there are no side effects or hidden modifications.

The restriction to accepting only state objects creates a clear contract: Workers don't fetch data, don't handle external services, and don't coordinate complex operations. They transform data in one specific way, making their behavior easy to understand, test, and modify.

This separation has concrete benefits in long-term maintenance. When a business process changes, you know exactly which Worker needs to be modified. When debugging, you can trace data transformations step by step through Worker methods. When adding new features, you can create new Workers without touching existing ones.

```python
from datetime import datetime
from pydantic import BaseModel, EmailStr

class UserState(BaseModel):
    email: EmailStr
    created_at: datetime
    is_verified: bool = False

    class Config:
        frozen = True

class UserWorker:
    """Transforms user data in specific ways."""
    
    @staticmethod
    def normalize_email(state: UserState) -> UserState:
        """Creates new state with normalized email."""
        return UserState(
            email=state.email.lower(),
            created_at=state.created_at,
            is_verified=state.is_verified
        )

    @staticmethod
    def mark_as_verified(state: UserState) -> UserState:
        """Creates new state with verified status."""
        return UserState(
            email=state.email,
            created_at=state.created_at,
            is_verified=True
        )
```

Consider how this Worker handles email normalization. It doesn't check if the email is valid (that's handled by the State Model), doesn't verify business rules (that's for Investigators), and doesn't coordinate with other operations (that's for Delegators). It does one thing: converts the email to lowercase and returns a new state.

This focused responsibility means that when the email normalization rules change (maybe we need to handle new email formats), we modify just this Worker. When we need to track how an email was changed, we can look at the Worker's transformations. When we add new email-related features, we add new Worker methods without changing existing ones.

Testing becomes straightforward because each transformation is isolated:

```python
def test_email_normalization():
    # Given a user state with mixed-case email
    initial_state = UserState(
        email="User@Example.com",
        created_at=datetime.utcnow(),
        is_verified=False
    )
    
    # When we normalize the email
    worker = UserWorker()
    new_state = worker.normalize_email(initial_state)
    
    # Then we get a new state with lowercase email
    assert new_state.email == "user@example.com"
    # And other fields remain unchanged
    assert new_state.created_at == initial_state.created_at
    assert new_state.is_verified == initial_state.is_verified
    # And the original state is unchanged
    assert initial_state.email == "User@Example.com"
```

## 2. Investigators

Investigators embody your organization's business rules as pure functions. Unlike data validation, which checks if data is correctly formatted and valid, Investigators check if data meets specific business requirements. These requirements often change as your business evolves, which is why they're kept separate from structural validation.

Think of Investigators like policy enforcers. They don't transform data (that's for Workers), don't validate data structure (that's for State Models), and don't handle errors (that's for Error components). They answer one specific question about whether a state meets a business requirement.

The power of this separation becomes clear when business rules change. If your company decides to modify the eligibility rules for a holiday promotion, you know exactly where to make that change. If you need to add new rules, you add new methods. If you need to understand why a user wasn't eligible for something, you can look at each rule in isolation.

```python
class PromotionInvestigator:
    """Checks business rules for promotional eligibility."""
    
    @staticmethod
    def is_eligible_for_holiday_promotion(state: UserState) -> bool:
        """Business rule: Holiday promotion only for users registered > 30 days."""
        return (datetime.now() - state.created_at).days > 30
    
    @staticmethod
    def can_receive_loyalty_discount(state: UserState) -> bool:
        """Business rule: Company policy requires verified status for loyalty program."""
        return state.is_verified
```

Each method in an Investigator returns a simple boolean. They don't throw errors (that's what Error components do using these Investigators). They don't have side effects. They check one specific condition about the state of your data.

This makes testing business rules straightforward and comprehensive:

```python
def test_holiday_promotion_eligibility():
    # Given a user who registered 31 days ago
    state = UserState(
        email="user@example.com",
        created_at=datetime.now() - timedelta(days=31),
        is_verified=True
    )
    
    # When we check holiday promotion eligibility
    investigator = PromotionInvestigator()
    
    # Then they should be eligible
    assert investigator.is_eligible_for_holiday_promotion(state) == True
    
    # Given a user who registered 29 days ago
    recent_state = UserState(
        email="new@example.com",
        created_at=datetime.now() - timedelta(days=29),
        is_verified=True
    )
    
    # Then they should not be eligible
    assert investigator.is_eligible_for_holiday_promotion(recent_state) == False
```

When auditing your system's business rules, each Investigator serves as clear documentation of what rules exist and how they're enforced. When adding new features, you can easily see what rules might need to be considered. When debugging why something happened, you can test each business rule independently.

## 3. Error Components

Error components translate business rule violations into actionable errors. While Investigators determine if rules are being followed, Error components decide what happens when they're not. They maintain a clear separation between detecting a problem (Investigators) and handling it (Error components).

This separation serves multiple purposes. First, it allows you to change how you handle rule violations without changing the rules themselves. Second, it provides a central place to manage error messages and error types. Third, it makes it easy to add new error handling without touching existing business logic.

```python
class PromotionError(Exception):
    """Error class for promotion-related business rule violations."""
    
    @staticmethod
    def raise_if_ineligible_for_holiday_promotion(state: UserState) -> None:
        """Enforces holiday promotion eligibility rule."""
        if not PromotionInvestigator.is_eligible_for_holiday_promotion(state):
            raise PromotionError("User must be registered for at least 30 days for holiday promotion")
    
    @staticmethod
    def raise_if_ineligible_for_loyalty_discount(state: UserState) -> None:
        """Enforces loyalty program rules."""
        if not PromotionInvestigator.can_receive_loyalty_discount(state):
            raise PromotionError("User must be verified to receive loyalty discount")
```

Notice how the Error class:
1. Inherits from Exception
2. Uses Investigators to check conditions
3. Only raises instances of itself
4. Has clear, specific error messages
5. Contains all error-raising methods related to its domain

This makes error handling predictable and maintainable:

```python
def test_promotion_errors():
    # Given an unverified user
    state = UserState(
        email="user@example.com",
        created_at=datetime.now(),
        is_verified=False
    )
    
    # When we check loyalty eligibility
    with pytest.raises(PromotionError) as error:
        PromotionError.raise_if_ineligible_for_loyalty_discount(state)
    
    # Then we get the specific error message
    assert str(error.value) == "User must be verified to receive loyalty discount"
```

Error components also provide a natural place to add logging, monitoring, or other error handling behaviors without cluttering business logic. When you need to change how errors are handled, you know exactly where to look.

## 4. State Models

State Models capture data structure and basic validation rules that transcend specific business contexts. Unlike business rules in Investigators, these validations represent fundamental truths about your data that rarely change and apply across different uses.

```python
from pydantic import BaseModel, EmailStr, Field, validator
from datetime import datetime
from uuid import UUID
from decimal import Decimal

class OrderState(BaseModel):
    """Represents an order's state at any point in time."""
    order_id: UUID
    customer_id: UUID
    items: list[UUID]
    total_amount: Decimal = Field(ge=Decimal('0'))  # Must be non-negative
    shipping_address: str = Field(min_length=1)     # Can't be empty
    created_at: datetime = Field(default_factory=datetime.utcnow)
    status: Literal['pending', 'processing', 'completed'] = 'pending'

    @validator('shipping_address')
    def address_must_contain_street_info(cls, v: str) -> str:
        """Basic structural validation of address."""
        parts = v.split()
        if len(parts) < 3:  # Must have number, street type, and name at minimum
            raise ValueError('Address must contain street number and name')
        return v

    class Config:
        frozen = True  # Data is immutable
        json_encoders = {  # How to serialize special types
            UUID: str,
            datetime: lambda dt: dt.isoformat(),
            Decimal: str
        }
```

Notice what belongs here:
- Type definitions (UUID, Decimal, datetime)
- Format constraints (non-empty strings, non-negative numbers)
- Basic data integrity rules (addresses need certain parts)
- Immutability configuration
- Serialization rules

And what doesn't:
- Business rules about who can order
- Validation of order totals against user limits
- Shipping restrictions by region
- Discount eligibility rules

These State Models provide guarantees about data shape that every other component can rely on. When a Worker receives a state object, it knows the data is structurally valid and can focus on its transformations.

```python
def test_order_state_validation():
    # Valid state passes
    valid_state = OrderState(
        order_id=uuid4(),
        customer_id=uuid4(),
        items=[uuid4()],
        total_amount=Decimal('50.00'),
        shipping_address='123 Main Street',
    )
    assert valid_state.total_amount == Decimal('50.00')

    # Invalid: negative amount
    with pytest.raises(ValidationError) as error:
        OrderState(
            order_id=uuid4(),
            customer_id=uuid4(),
            items=[uuid4()],
            total_amount=Decimal('-10.00'),
            shipping_address='123 Main Street',
        )
    assert "ensure this value is greater than or equal to 0" in str(error.value)

    # Invalid: malformed address
    with pytest.raises(ValidationError) as error:
        OrderState(
            order_id=uuid4(),
            customer_id=uuid4(),
            items=[uuid4()],
            total_amount=Decimal('50.00'),
            shipping_address='123',  # Missing street information
        )
```

## 5. Delegators

Delegators orchestrate workflows by coordinating other components. They contain exactly one public method - `process` - which takes a state object and returns a state object. Think of them as the workflow definition: they determine what happens next based on the current state.

```python
class OrderProcessingDelegator:
    """Coordinates the order processing workflow."""

    def __init__(
        self,
        inventory_worker: InventoryWorker,
        pricing_worker: PricingWorker,
        order_investigator: OrderInvestigator
    ):
        self.inventory_worker = inventory_worker
        self.pricing_worker = pricing_worker
        self.order_investigator = order_investigator

    def process(self, state: OrderState) -> OrderState:
        """Orchestrates order processing workflow."""
        # Apply inventory checks
        if self.order_investigator.can_fulfill_from_stock(state):
            state = self.inventory_worker.reserve_inventory(state)
        else:
            state = self.inventory_worker.backorder_items(state)

        # Apply pricing based on conditions
        if self.order_investigator.is_bulk_order(state):
            state = self.pricing_worker.apply_bulk_discount(state)

        return state
```

Key aspects of Delegators:
1. Only accept state objects as input
2. Only return state objects as output
3. Don't implement business logic
4. Don't transform data directly
5. Don't handle errors (let them propagate)
6. Follow a clear workflow pattern

Testing focuses on the orchestration flow:

```python
def test_order_delegator():
    # Given an initial order state
    state = OrderState(
        order_id=uuid4(),
        customer_id=uuid4(),
        items=[uuid4()],
        total_amount=Decimal('100.00'),
        shipping_address='123 Main Street'
    )
    
    # And mocked components
    inventory_worker = Mock(
        reserve_inventory=Mock(return_value=replace(state, inventory_reserved=True))
    )
    pricing_worker = Mock(
        apply_bulk_discount=Mock(return_value=replace(state, total_amount=Decimal('90.00')))
    )
    order_investigator = Mock(
        can_fulfill_from_stock=Mock(return_value=True),
        is_bulk_order=Mock(return_value=True)
    )

    # When we process the order
    delegator = OrderProcessingDelegator(
        inventory_worker=inventory_worker,
        pricing_worker=pricing_worker,
        order_investigator=order_investigator
    )
    result = delegator.process(state)

    # Then the workflow executed correctly
    inventory_worker.reserve_inventory.assert_called_once()
    pricing_worker.apply_bulk_discount.assert_called_once()
    assert result.total_amount == Decimal('90.00')
```

The true power of Delegators comes from their clarity. Reading a Delegator's process method tells you exactly how your system handles a particular workflow. When requirements change, you know exactly where to modify the flow. When debugging, you can trace the exact path state took through your system.
