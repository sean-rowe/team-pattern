# Team Pattern Architecture Guide

## Core Principles

The **Team Pattern** is an architectural approach designed to create predictable, maintainable, and testable code by enforcing strict separations of concerns. It revolves around immutable state transformations and clear delineation of responsibilities among different components:

- **State Models**: Define immutable data structures with basic validation rules.
- **Workers**: Perform specific data transformations and enforce business rules using Enforcers.
- **Investigators**: Check single business rules and return booleans.
- **Enforcers**: Enforce business rules by throwing typed exceptions.
- **Delegators**: Orchestrate workflows by coordinating Workers.

Each component adheres to strict responsibilities and constraints, enabling a clean architecture that simplifies debugging, testing, and future modifications.

---

## State Models

### Theory and Reasons

**State Models** capture the fundamental structure and basic validation of your data. They represent the immutable state of your domain entities at any point in time. By making state immutable and enforcing basic validation rules, State Models provide a reliable foundation upon which the rest of your system builds. This immutability ensures that once a state is created, it cannot be altered, which eliminates side effects and makes reasoning about your code easier.

State Models focus on structural validation—ensuring that data is correctly typed and formatted. They do not encompass business rules, which may change frequently, but instead enforce the core data integrity that is essential across all contexts.

### Key Characteristics

- Inherit from a base state class (e.g., `BaseState`).
- Are immutable (e.g., `frozen=True` in Pydantic models).
- Use strong typing for all fields.
- Contain clear, focused properties without methods or behaviors.
- Track previous states for state history and auditing.
- Enforce basic data validation that is unlikely to change.

### Good Example

```python
from pydantic import BaseModel
from typing import Optional
from datetime import datetime
from decimal import Decimal

class BaseState(BaseModel):
    previous_state: Optional['BaseState'] = None

    class Config:
        frozen = True

class PaymentState(BaseState):
    amount: Decimal
    currency: str
    timestamp: datetime
```

**Explanation**

- **Immutability**: Using `frozen=True` ensures the state cannot be altered after creation.
- **Strong Typing**: Fields like `amount`, `currency`, and `timestamp` are explicitly typed.
- **No Behavior**: The class contains no methods; it's a pure data container.
- **State Tracking**: `previous_state` allows tracking of state changes over time.

### Bad Example

```python
class PaymentState:
    def __init__(self, amount, currency):
        self.amount = amount  # No typing
        self.currency = currency
        self.is_valid = False  # Mutable state

    def validate(self):
        self.is_valid = self.amount > 0
```

**Issues**

- **Lack of Strong Typing**: Fields are not explicitly typed.
- **Mutable State**: `is_valid` can change, leading to side effects.
- **Methods Included**: Contains behavior (`validate` method), violating separation of concerns.

---

## Workers

### Theory and Reasons

**Workers** are responsible for transforming state in specific, predictable ways. Each Worker method performs a single, well-defined transformation, taking a state object as input and returning a new, transformed state object. Workers also enforce business rules by using **Enforcers** to ensure they can perform the tasks they are designed for.

Workers may need to call external libraries or functions to perform their tasks—which can introduce side effects—but their core responsibility remains focused on data transformation without making decisions based on business logic within the Worker itself. The assumption is that the happy path will occur, and any errors are propagated up to be handled by the code that calls the Delegator, keeping the pattern framework-agnostic.

### Key Characteristics

- **Single Transformation Responsibility**: Perform one specific data transformation.
- **Enforce Business Rules**: Use Enforcers to check business rules before performing tasks.
- **Interaction with External Libraries**: May call external libraries or functions as needed.
- **No Business Logic Decisions**: Do not make decisions based on business rules within the Worker; they enforce rules to ensure they can perform their task.
- **Stateless Methods**: Use static methods or functions; Workers don't maintain internal state.
- **State Input and Output**: Accept one state type and return a new state type.
- **Side Effects Acceptable**: May have side effects due to external interactions but should manage them appropriately.
- **Clear Naming**: Have descriptive names reflecting their single responsibility.

### Good Example

```python
class PaymentWorker:
    @staticmethod
    def process_payment(state: PaymentState) -> ProcessedPaymentState:
        """Processes a payment using an external payment gateway."""
        # Enforce business rules
        PaymentEnforcer.raise_if_invalid_amount(state)

        import payment_gateway  # External library for payment processing

        # Call to external service (side effect: network call)
        response = payment_gateway.charge(
            amount=state.amount,
            currency=state.currency,
            payment_method=state.payment_method
        )

        # Create new state based on response
        return ProcessedPaymentState(
            amount=state.amount,
            currency=state.currency,
            payment_method=state.payment_method,
            transaction_id=response.transaction_id,
            status=response.status,
            previous_state=state
        )
```

**Explanation**

- **Enforces Business Rules**: Calls `PaymentEnforcer.raise_if_invalid_amount(state)` before proceeding.
- **External Service Call**: Interacts with a payment gateway.
- **Side Effects**: Network communication with the payment service.
- **No Business Logic Decisions**: Does not decide whether to process the payment; it ensures it can and then processes it.
- **Single Responsibility**: Focuses solely on processing the payment.
- **State Tracking**: Returns a new state with the transaction details.

### Bad Example

```python
class PaymentWorker:
    def process_payment(self, state: PaymentState):
        if state.amount <= 0:
            # Business logic decision within Worker
            raise ValueError("Invalid amount")

        # Proceed with payment processing
        # ...
```

**Issues**

- **Business Logic Decisions**: Makes decisions based on `state.amount` within the Worker.
- **Throws Generic Exception**: Raises `ValueError` instead of using an Enforcer.
- **Mixed Responsibilities**: Both enforces rules and processes payment.

### Managing Side Effects in Workers

Workers can:

- **Enforce Business Rules**: Use Enforcers to ensure they can perform their tasks.
- **Perform External Calls**: Call external APIs or services as needed.
- **Handle Responses**: Process responses from external calls to create new state objects.

Workers should not:

- **Make Business Logic Decisions**: Do not decide whether to perform an action based on business rules; they enforce rules to ensure they can perform their task.
- **Contain Complex Branching**: Avoid complex conditional logic that could obscure the Worker's purpose.

---

## Investigators

### Theory and Reasons

**Investigators** encapsulate business rules as pure functions that return booleans. They answer specific questions about whether a state meets certain business criteria. By separating business rules from data transformations and validations, Investigators make it easier to modify and test business logic independently.

This separation is crucial because business rules often change as the organization evolves. By isolating them, changes can be made in one place without affecting other components.

### Key Characteristics

- Implement one business rule per method.
- Use static methods only.
- Return a boolean value.
- Do not modify state or have side effects.
- Have clear naming conventions, often starting with "is_" or "can_".

### Good Example

```python
class PaymentInvestigator:
    @staticmethod 
    def is_valid_amount(state: PaymentState) -> bool:
        """Checks if the payment amount is within the valid range."""
        return Decimal('0') < state.amount < Decimal('10000')
```

**Explanation**

- **Single Rule**: Checks only if the amount is valid.
- **Pure Function**: No side effects or state modifications.
- **Boolean Return**: Returns `True` or `False`.
- **Clear Naming**: Method name clearly states its purpose.

### Bad Example

```python
class PaymentInvestigator:
    def validate_payment(self, state: PaymentState) -> str:
        if state.amount <= 0:
            state.is_valid = False  # Modifies state
            return "invalid"  # Non-boolean return

        state.amount = round(state.amount, 2)  # Performs transformation
        return "valid"
```

**Issues**

- **State Modification**: Changes `state.is_valid`.
- **Data Transformation**: Alters `state.amount`.
- **Non-Boolean Return**: Returns strings instead of booleans.
- **Contains Multiple Responsibilities**: Validates and transforms data.

---

## Enforcers

### Theory and Reasons

**Enforcers** enforce business rules by using Investigators and throwing typed exceptions when rules are violated. Workers use Enforcers to ensure they can perform their tasks. By separating rule enforcement into Enforcers, the system maintains a clear mechanism for handling errors, keeping error handling separate from business logic and transformations.

This separation allows for consistent error management across the system and makes it easier to change how errors are handled without modifying business logic.

### Key Characteristics

- Inherit from a base `Exception` class.
- Store the error state for context.
- Use static methods to raise exceptions.
- Utilize Investigators to check conditions.
- Throw only their own exception type.
- Provide clear, specific error messages.

### Good Example

```python
class PaymentError(Exception):
    """Custom exception class for payment-related errors."""
    def __init__(self, message: str, state: BaseState):
        super().__init__(message)
        self.error_state = state

class PaymentEnforcer:
    """Enforces business rules and raises exceptions when violated."""

    @staticmethod
    def raise_if_invalid_amount(state: PaymentState) -> None:
        if not PaymentInvestigator.is_valid_amount(state):
            raise PaymentError("Invalid payment amount", state)
```

**Explanation**

- **Used by Workers**: Workers call Enforcers to ensure they can perform their tasks.
- **Typed Exception**: `PaymentError` inherits from `Exception`.
- **Uses Investigator**: Checks business rule via `PaymentInvestigator`.
- **Stores State**: Keeps the state at the time of error.
- **Clear Message**: Provides specific error messages.
- **Separation of Concerns**: `PaymentEnforcer` focuses solely on enforcing rules and raising exceptions.

### Bad Example

```python
class PaymentEnforcer:
    @staticmethod
    def validate_payment(state: PaymentState):
        if state.amount <= 0:  # Direct check instead of using Investigator
            state.is_valid = False  # Modifies state
            raise ValueError()  # Throws generic exception
```

**Issues**

- **Direct Check**: Bypasses the Investigator.
- **State Modification**: Alters `state.is_valid`.
- **Generic Exception**: Raises `ValueError` instead of a custom exception.
- **Mixed Responsibilities**: Validates and handles errors.

---

## Delegators

### Theory and Reasons

**Delegators** orchestrate workflows by coordinating Workers. They define the recipe for how different components interact to achieve a business process. Delegators assume that the happy path will occur and do not use Enforcers directly. They rely on Workers to enforce business rules and handle any exceptions that may arise.

By centralizing workflow logic in Delegators, the system gains clarity and modularity, making it easier to understand, test, and modify workflows without affecting other parts of the system. The code that calls the Delegator is responsible for handling any errors, keeping the pattern framework-agnostic.

### Key Characteristics

- Coordinate Workers to define workflows.
- Assume that business rules have been enforced by Workers.
- Do not implement business logic directly.
- Do not handle errors; exceptions propagate to be handled by calling code.
- Have a single public method, typically `process`, that takes and returns state objects.
- Focus on orchestrating tasks in the correct order.

### Good Example

```python
class PaymentDelegator:
    def __init__(self, payment_worker: PaymentWorker):
        self.payment_worker = payment_worker

    def process(self, state: PaymentState) -> ProcessedPaymentState:
        # Delegator assumes that Workers will enforce business rules
        # Simply calls the Worker to process payment
        return self.payment_worker.process_payment(state)
```

**Explanation**

- **Workflow Coordination**: Orchestrates the flow by calling Workers.
- **No Business Logic**: Does not contain direct checks or transformations.
- **No Error Handling**: Assumes happy path; exceptions are handled by calling code.
- **Clear Process Flow**: Focuses on sequencing tasks.

### Bad Example

```python
class PaymentDelegator:
    def process(self, state: PaymentState):
        try:
            if state.amount <= 0:  # Contains business logic
                state.is_valid = False
            else:
                state.amount = round(state.amount, 2)  # Performs transformation
                state.is_valid = True
        except Exception as e:
            state.status = 'error'  # Handles errors internally
```

**Issues**

- **Business Logic Included**: Contains direct checks and transformations.
- **State Mutation**: Modifies `state.is_valid` and `state.amount`.
- **Error Handling**: Catches and handles exceptions internally.
- **Lack of Clarity**: Combines multiple responsibilities, making it hard to maintain.

---

## Mapping from Gherkin

### Theory and Reasons

Gherkin is a domain-specific language used for behavior-driven development (BDD). Mapping Gherkin specifications directly to pattern components ensures that your code aligns closely with business requirements and acceptance criteria. This mapping enhances traceability, making it easier to ensure that all scenarios are covered and that the code fulfills the specified behaviors.

### Mapping Rules

- **Feature** → Group of related Delegators.
- **Rule** → Specific Delegator class.
- **Given** → Initial State Model.
- **When** → Worker enforcement (via Enforcers).
- **Then** → Worker transformation.
- **And/But** → Additional Workers or steps.
- **Scenario** → Workflow defined in a Delegator's process method.
- **Examples** → Test cases using the pattern.

### Example

```gherkin
Feature: Payment Processing
  Rule: New payments must be validated

  Scenario: Valid payment processed
    Given a payment of $100 USD
    Then payment should be processed
```

### Mapping to Code

- **Given a payment of $100 USD** → `PaymentState`
- **Then payment should be processed** → `PaymentWorker.process_payment`

**Code Implementation**

```python
class PaymentState(BaseState):
    amount: Decimal
    currency: str

class PaymentInvestigator:
    @staticmethod
    def is_valid_amount(state: PaymentState) -> bool:
        return Decimal('0') < state.amount <= Decimal('10000')

class PaymentEnforcer:
    @staticmethod
    def raise_if_invalid_amount(state: PaymentState) -> None:
        if not PaymentInvestigator.is_valid_amount(state):
            raise PaymentError("Invalid payment amount", state)

class PaymentWorker:
    @staticmethod
    def process_payment(state: PaymentState) -> ProcessedPaymentState:
        # Enforce business rules
        PaymentEnforcer.raise_if_invalid_amount(state)

        # Process the payment (e.g., call payment gateway)
        # ...

        # Return new state
        return ProcessedPaymentState(
            amount=state.amount,
            currency=state.currency,
            previous_state=state
        )

class PaymentDelegator:
    def __init__(self, payment_worker: PaymentWorker):
        self.payment_worker = payment_worker

    def process(self, state: PaymentState) -> ProcessedPaymentState:
        # Simply calls the Worker
        return self.payment_worker.process_payment(state)
```

---

## Criticism and Responses

### Criticism 1: Overhead of Multiple Components

**Critique**: The pattern introduces a lot of components (Workers, Investigators, Enforcers, Delegators) for what could be accomplished with fewer classes and methods. This may lead to increased complexity and overhead in development and maintenance.

**Response**: While the pattern introduces multiple components, each has a clear and single responsibility. This separation enhances maintainability by making the codebase more modular and easier to understand. When changes are needed, they can be made in isolation without unintended side effects on other parts of the system. The initial overhead pays off in the long term through easier debugging, testing, and adaptability to changing requirements.

### Criticism 2: Performance Concerns with Immutable States

**Critique**: The use of immutable state objects and the creation of new state instances for every transformation could lead to performance issues, especially in systems with high throughput requirements.

**Response**: While immutability does introduce some overhead due to object creation, it provides significant benefits in terms of thread safety, predictability, and ease of reasoning about the code. In many cases, modern garbage-collected languages and optimizations like copy-on-write minimize the performance impact. If performance becomes a critical concern, specific optimizations can be applied where necessary without compromising the overall architecture.

### Criticism 3: Lack of Flexibility in Error Handling

**Critique**: The pattern's approach to error handling, where Enforcers throw exceptions and Delegators assume the happy path, might not allow for nuanced error management strategies within the pattern itself.

**Response**: The pattern encourages letting exceptions propagate to the code that calls the Delegator, keeping the pattern framework-agnostic. This allows for error handling strategies, such as retries or fallbacks, to be implemented at a higher level, tailored to the specific context and requirements. This separation maintains the purity and predictability of the pattern's components.

### Criticism 4: Steep Learning Curve

**Critique**: The strict rules and separation of concerns require developers to understand and adhere to a specific way of structuring code, which may have a steep learning curve and could lead to inconsistencies if not followed strictly.

**Response**: Any architectural pattern introduces some learning overhead. However, the benefits of a well-defined structure outweigh the initial learning curve. Clear guidelines and documentation, like this guide, help bring team members up to speed. Additionally, code reviews and automated linting can help enforce consistency. Over time, the team will benefit from the increased maintainability and clarity provided by the pattern.

### Criticism 5: Not Suitable for All Scenarios

**Critique**: The pattern may not be suitable for all types of applications or domains, particularly those that require more dynamic behavior or have less stringent requirements for immutability and separation of concerns.

**Response**: It's true that no single architectural pattern is a one-size-fits-all solution. The Team Pattern is particularly well-suited for applications where predictability, maintainability, and clear separation of business logic are paramount. In domains where flexibility and rapid prototyping are more critical, a less rigid architecture might be more appropriate. It's essential to evaluate the pattern against the specific needs and context of the project.

---

**Summary**

- **Workers Enforce Business Rules**: Workers use Enforcers to ensure they can perform their tasks, calling assertions as needed.
- **Delegators Assume Happy Path**: Delegators do not use Enforcers directly and assume that everything is fine.
- **Error Handling Is External**: The code that calls the Delegator is responsible for handling errors, keeping the pattern framework-agnostic.
- **Workers Can Call External Libraries**: It's acceptable for Workers to interact with external libraries or services as needed for their tasks.
- **Side Effects Are Managed**: While side effects may occur, they are managed within the Worker and do not impact the overall architecture.
- **No Business Logic Decisions in Workers**: Workers enforce rules but do not make decisions based on business logic; they ensure they can perform their tasks.
- **Focus on Data Transformation**: The primary role of Workers is to transform input state into a new state, using whatever processes are necessary.
- **Enforcers Handle Rule Violations**: Enforcers are responsible for enforcing business rules by raising typed exceptions.

By adhering to these principles, the Team Pattern remains robust and adaptable, accommodating practical needs without compromising its core benefits. This approach ensures that components are modular, testable, and maintainable, and that the system remains flexible and framework-agnostic.
