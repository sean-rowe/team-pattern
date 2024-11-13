# Architectural Pattern Guide

## Introduction

In the ever-evolving landscape of software development, creating systems that are **maintainable**, **scalable**, and **robust** is crucial. As applications grow in complexity, adopting an architectural pattern that promotes clear separation of concerns, modularity, and efficient error handling becomes essential.

This guide introduces the **Team Pattern**—a comprehensive architectural framework designed to streamline the development process by defining distinct roles and responsibilities for various components within a system. By adhering to this pattern, development teams can ensure that each part of the application is focused, testable, and easy to maintain, ultimately leading to higher quality and more resilient software.

### Core Components

The Team Pattern is built around six key components, each with a specific role:

1. **Workers**: Handle the core business logic and data transformations, ensuring operations are executed correctly and efficiently. Workers are invoked exclusively by Delegators and must use Error class `assertion` methods to validate data within the state they process.

2. **Fetchers**: Manage data access and retrieval from external systems, abstracting complexities and providing clean interfaces for data consumption. Fetchers return Pydantic models representing the extracted data and do not handle errors internally.

3. **Delegators**: Orchestrate workflows by coordinating between Fetchers, Workers, and Investigators to manage complex business processes seamlessly. Delegators contain only a single `process` method, avoid direct data manipulation, do not manage state beyond the provided state objects, and rely on Investigators for decision-making without interacting with Error components.

4. **Investigators**: Enforce business rules and validations, maintaining data integrity by answering specific true/false questions about the data. They are pure functions that check only one condition per method.

5. **Error Components**: Provide a robust error handling framework, acting as type guards to validate state objects. They utilize Investigators to assess data and raise exceptions if validations fail. Delegators do not interact directly with Error components; instead, Workers and Investigators handle validations and error assertions.

6. **State Models**: Define immutable data structures that traverse the system, ensuring consistency and reliability throughout the processing pipeline. They validate the shape of the data but remain agnostic to business rules, focusing solely on data structure integrity.

### Principles

The Team Pattern is grounded in several fundamental principles:

- **Single Responsibility**: Each component has a distinct responsibility, promoting modularity and ease of maintenance.
- **Immutability**: State models are immutable, preventing unintended side-effects and ensuring consistent data flow.
- **Separation of Concerns**: Business logic, data retrieval, validation, and error handling are handled by dedicated components.
- **Error Handling**: Errors are managed systematically, allowing for predictable and maintainable error propagation.
- **Clear Documentation**: Comprehensive documentation is maintained across all components, facilitating understanding and collaboration within the team.

### Benefits

Adopting the Team Pattern offers numerous advantages:

- **Maintainability**: Clear separation of responsibilities makes the system easier to maintain and extend.
- **Testability**: Modular components with single responsibilities are easier to test in isolation.
- **Scalability**: The pattern supports scalable architectures, allowing components to be scaled independently as needed.
- **Reliability**: Error handling and immutable state models enhance the system's reliability.
- **Clarity**: Comprehensive documentation and clear component boundaries improve overall system clarity, aiding both development and onboarding.

This guide delves into each component of the Team Pattern, providing examples of both compliant and non-compliant implementations to illustrate best practices and common pitfalls. By following the guidelines outlined here, development teams can leverage the Team Pattern to build high-quality, resilient software systems.

## Table of Contents

1. [Workers](#workers)
   - [❌ Bad Example (Non-Compliant Worker)](#bad-example-non-compliant-worker)
   - [✅ Good Example (Compliant Worker)](#good-example-compliant-worker)
   - [Key Differences](#key-differences)
2. [Fetchers](#fetchers)
   - [❌ Bad Implementation](#bad-implementation)
   - [✅ Good Implementation](#good-implementation)
   - [Key Differences and Improvements in the Good Implementation](#key-differences-and-improvements-in-the-good-implementation)
3. [Delegators](#delegators)
   - [❌ Incorrect Delegator Implementation](#incorrect-delegator-implementation)
   - [✅ Correct Delegator Implementation](#correct-delegator-implementation)
   - [Key Differences](#key-differences-1)
   - [Supporting Components](#supporting-components)
4. [Investigators](#investigators)
   - [❌ Incorrect Implementation](#incorrect-implementation)
   - [✅ Correct Implementation](#correct-implementation)
   - [Key Differences](#key-differences-2)
5. [Error Components](#error-components)
   - [❌ Incorrect Implementation](#incorrect-implementation-1)
   - [✅ Correct Implementation](#correct-implementation-1)
   - [Key Differences and Explanations](#key-differences-and-explanations)
   - [Usage Example](#usage-example)
6. [State Models](#state-models)
   - [❌ Incorrect Implementation](#incorrect-implementation-2)
   - [✅ Correct Implementation](#correct-implementation-2)
   - [Key Aspects](#key-aspects)
   - [Usage Example](#usage-example-1)
   - [Validation Example (Where It Should Go)](#validation-example-where-it-should-go)

---

## Workers

**Workers** handle the core logic and transformations, ensuring that business operations are executed correctly and efficiently. They are exclusively invoked by Delegators and must utilize Error class `assertion` methods to validate the data within the state they process.

### ❌ Bad Example Non-Compliant Worker

```python
from dataclasses import dataclass
from typing import Optional
from datetime import datetime

@dataclass
class UserState:
    email: str
    password: str
    created_at: Optional[datetime] = None

class BadUserRegistrationWorker:
    def process_registration(self, user_state: UserState) -> UserState:
        # ❌ Multiple responsibilities and no error assertions
        if not self.is_valid_email(user_state.email):
            raise ValueError("Invalid email format")

        # ❌ Direct data manipulation without using Error assertions
        user_state.created_at = datetime.now()

        # ❌ Branching logic that should be in Investigators
        if len(user_state.password) < 8:
            return self.handle_weak_password(user_state)
        
        # ❌ Calling another worker (not allowed)
        password_worker = PasswordHashingWorker()
        hashed_password = password_worker.hash_password(user_state.password)
        
        return UserState(
            email=user_state.email.lower(),
            password=hashed_password,
            created_at=user_state.created_at
        )

    def is_valid_email(self, email: str) -> bool:
        # ❌ Worker containing multiple methods (violation)
        return '@' in email

    def handle_weak_password(self, user_state: UserState) -> UserState:
        # ❌ Additional method and branching logic (violation)
        raise ValueError("Password too weak")
```

### ✅ Good Example Compliant Worker

```python
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, EmailStr

class UserState(BaseModel):
    email: EmailStr
    password: str
    created_at: Optional[datetime] = None
    
    class Config:
        frozen = True

class UserRegistrationInvestigator:
    """
    Investigator class to enforce business rules for user registration.
    """
    @staticmethod
    def is_valid_email(email: str) -> bool:
        return '@' in email

    @staticmethod
    def is_valid_password_length(password: str) -> bool:
        return len(password) >= 8

class UserRegistrationErrors:
    @staticmethod
    def assert_valid_email(email: str) -> None:
        if not UserRegistrationInvestigator.is_valid_email(email):
            raise ValueError("Invalid email format")

    @staticmethod
    def assert_valid_password_length(password: str) -> None:
        if not UserRegistrationInvestigator.is_valid_password_length(password):
            raise ValueError("Password too short")

class UserRegistrationWorker:
    """
    Processes user registration data by validating and transforming the input state.

    **Summary:**
        Handles the core transformation of user registration data, ensuring email 
        normalization and timestamp assignment.

    **Remarks:**
        - Designed to be invoked only by Delegators
        - Performs email normalization and timestamp assignment
        - Utilizes Error assertions for validation
        - Avoids branching logic or multiple methods
        - Maintains immutability by creating new state objects

    **Example:**
        ```python
        initial_state = UserState(
            email="User@Example.com",
            password="securepass123"
        )
        worker = UserRegistrationWorker()
        processed_state = worker.process(initial_state)
        ```

    **Throws:**
        - `ValueError`: When email format is invalid
        - `ValueError`: When password length is insufficient

    **See:**
        - `UserState`
        - `UserRegistrationErrors`
        - `UserRegistrationInvestigator`

    **Parameters:**
        - `state (UserState)`: The initial user registration state containing email 
          and password information

    **Returns:**
        - `UserState`: A new immutable state object with normalized email and 
          timestamp
    """
    @staticmethod
    def process(state: UserState) -> UserState:
        # ✅ Validate inputs using Error assertions
        UserRegistrationErrors.assert_valid_email(state.email)
        UserRegistrationErrors.assert_valid_password_length(state.password)

        # ✅ Return new immutable state with transformations
        return UserState(
            email=state.email.lower(),
            password=state.password,
            created_at=datetime.now()
        )
```

### Key Differences

1. **Documentation**: The compliant Worker includes comprehensive docstrings covering Summary, Remarks, Example, Throws, See, Parameters, and Returns.
2. **Single Responsibility**: Focuses solely on core transformations without mixing in business logic decisions.
3. **Error Assertions**: Utilizes dedicated Error class methods that leverage Investigators for validation instead of direct validation.
4. **No Branching**: Avoids conditional logic, delegating decisions to Investigators.
5. **Immutability**: Creates new state objects rather than modifying existing ones.
6. **No Additional Methods**: Contains only the `process` method.
7. **No Worker Chaining**: Does not call other Workers.
8. **Clear Validation**: Uses a separate Error class that relies on Investigators for assertions.
9. **State Management**: Employs Pydantic models with `frozen=True` for immutability.
10. **Focus on Transformation**: Concentrates solely on data transformation without mixing in business logic decisions.

---

## Fetchers

**Fetchers** manage data access and retrieval, abstracting the complexities of external systems and providing clean interfaces for data consumption. They do not create state objects but return one or more Pydantic models representing the extracted data. Fetchers do not handle errors; any errors encountered bubble up to be managed by the system executing the Delegator.

### ❌ Bad Implementation

```python
from typing import Dict, Optional
import requests

class BadUserFetcher:
    def __init__(self):
        self.base_url = "https://api.example.com"
        self.cache = {}  # Breaking pattern: maintaining state

    def get_user(self, user_id: int) -> Dict:
        # No proper error handling - tries to handle errors internally
        try:
            # Breaking pattern: using cache/maintaining state
            if user_id in self.cache:
                return self.cache[user_id]
            
            response = requests.get(f"{self.base_url}/users/{user_id}")
            
            # Breaking pattern: handling errors instead of bubbling up
            if response.status_code != 200:
                return {"error": "User not found"}
            
            # Breaking pattern: direct data manipulation
            user_data = response.json()
            user_data["full_name"] = f"{user_data['first_name']} {user_data['last_name']}"
            
            # Breaking pattern: storing state
            self.cache[user_id] = user_data
            return user_data

        except Exception as e:
            # Breaking pattern: handling errors
            return {"error": str(e)}

    # Breaking pattern: multiple methods
    def update_user_cache(self, user_id: int) -> None:
        if user_id in self.cache:
            del self.cache[user_id]
```

### ✅ Good Implementation

```python
from typing import Optional
import requests
from pydantic import BaseModel, Field
from datetime import datetime

class UserData(BaseModel):
    """Represents user data retrieved from the API"""
    id: int
    username: str
    email: str
    created_at: datetime
    is_active: bool = Field(default=True)

class UserFetcher:
    """Fetches user data from the external API service."""

    def __init__(self, base_url: str):
        """
        Initialize the UserFetcher with the API base URL.

        **Parameters:**
            - `base_url (str)`: The base URL of the API service
        """
        self.base_url = base_url

    async def get_user(self, user_id: int) -> UserData:
        """
        Retrieves user data from the API for a given user ID.

        **Summary:**
            Fetches user information from the external API service and returns it as
            a validated `UserData` model.

        **Remarks:**
            - Performs a direct API call without caching.
            - All errors are propagated to the caller for handling.

        **Example:**
            ```python
            fetcher = UserFetcher("https://api.example.com")
            try:
                user_data = await fetcher.get_user(123)
                print(f"Found user: {user_data.username}")
            except requests.RequestException as e:
                # Handle error at the caller level
                pass
            ```

        **Parameters:**
            - `user_id (int)`: The unique identifier of the user to retrieve

        **Returns:**
            - `UserData`: A Pydantic model containing the validated user data

        **Throws:**
            - `requests.RequestException`: When the API request fails
            - `ValidationError`: When the response data doesn't match the expected schema

        **See:**
            - `UserData`
            - [API Documentation](https://api.example.com/docs#get-user)
        """
        response = requests.get(f"{self.base_url}/users/{user_id}")
        
        # Let the response error propagate up
        response.raise_for_status()
        
        # Return validated Pydantic model
        return UserData(**response.json())
```

### Key Differences and Improvements in the Good Implementation

1. **Documentation**:
   - Comprehensive docstrings following pydoc style.
   - Clear sections for Summary, Remarks, Example, Parameters, Returns, and Throws.
   - Documentation of parameters, returns, and possible exceptions.

2. **Error Handling**:
   - No internal error handling.
   - Allows errors to bubble up to the caller.
   - Clear documentation of what might be thrown.

3. **State Management**:
   - No caching or state maintenance.
   - Pure data fetching functionality.

4. **Data Structure**:
   - Uses Pydantic model for data validation.
   - Clear type hints.
   - No data manipulation.

5. **Method Structure**:
   - Single method for fetching.
   - No additional utility methods.
   - Clear, focused responsibility.

6. **Async Support**:
   - Implements async pattern for I/O operations.
   - Better scalability.

7. **Validation**:
   - Uses Pydantic for automatic validation.
   - Clear data contract through the model.

---

## Delegators

**Delegators** orchestrate the workflow by coordinating between Fetchers, Workers, and Investigators to manage complex business processes seamlessly. They contain only a single `process` method, avoid direct data manipulation, do not manage state beyond the provided state objects, and rely on Investigators for decision-making without interacting with Error components.

### ❌ Incorrect Delegator Implementation

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class UserState:
    user_id: str
    email: Optional[str] = None
    is_verified: bool = False

class BadUserDelegator:
    def process(self, user_id: str) -> UserState:
        # ❌ Direct data manipulation instead of using Workers
        user_state = UserState(user_id=user_id)
        
        # ❌ Direct error handling instead of letting errors bubble up
        try:
            user_data = self.fetch_user_data(user_id)
            user_state.email = user_data.get('email')
        except Exception as e:
            print(f"Error fetching user data: {e}")
            return user_state

        # ❌ Direct business logic instead of using Investigators
        if '@' in user_state.email:
            user_state.is_verified = True
        
        # ❌ Direct state modification instead of creating new state
        self.update_user_status(user_state)
        
        return user_state

    # ❌ Additional methods beyond process
    def fetch_user_data(self, user_id: str) -> dict:
        # Direct data fetching instead of using Fetchers
        return {"email": "user@example.com"}

    # ❌ Additional methods beyond process
    def update_user_status(self, state: UserState) -> None:
        # Direct database update
        pass
```

#### Issues with the Incorrect Implementation:

1. **Multiple Methods Beyond `process`**: Contains methods like `fetch_user_data` and `update_user_status` which violate the single responsibility principle.
2. **Direct Data Manipulation**: Manipulates `UserState` directly instead of delegating to Workers.
3. **Error Handling**: Implements error handling within the Delegator instead of allowing errors to bubble up.
4. **Business Logic**: Contains business logic directly instead of utilizing Investigators.
5. **State Modification**: Modifies state directly instead of creating new immutable states.
6. **Data Fetching**: Fetches data directly instead of using Fetchers.

### ✅ Correct Delegator Implementation

```python
from typing import Optional
from pydantic import BaseModel, Field
from datetime import datetime

class UserState(BaseModel):
    user_id: str
    email: Optional[str] = None
    is_verified: bool = False
    created_at: datetime = Field(default_factory=datetime.utcnow)
    
    class Config:
        frozen = True

class UserFetcher:
    async def get_user(self, user_id: str) -> 'UserData':
        # Implementation of fetching user data
        pass

class EmailWorker:
    async def process(self, state: UserState, user_data: 'UserData') -> UserState:
        # Process email data and return new state
        return UserState(
            user_id=state.user_id,
            email=user_data.email,
            is_verified=state.is_verified,
            created_at=state.created_at
        )

class VerificationWorker:
    async def process(self, state: UserState) -> UserState:
        # Handle verification and return new state
        return UserState(
            user_id=state.user_id,
            email=state.email,
            is_verified=True,
            created_at=state.created_at
        )

class EmailInvestigator:
    def is_eligible_for_verification(self, state: UserState) -> bool:
        # Single condition check
        return state.email is not None and '@' in state.email

class UserDelegator:
    def __init__(
        self,
        user_fetcher: UserFetcher,
        email_worker: EmailWorker,
        verification_worker: VerificationWorker,
        email_investigator: EmailInvestigator
    ):
        self.user_fetcher = user_fetcher
        self.email_worker = email_worker
        self.verification_worker = verification_worker
        self.email_investigator = email_investigator

    async def process(self, user_id: str) -> UserState:
        # Create initial state
        initial_state = UserState(user_id=user_id)
        
        # Fetch user data using Fetcher
        user_data = await self.user_fetcher.get_user(user_id)
        
        # Use Worker to process email data
        state_with_email = await self.email_worker.process(
            initial_state, 
            user_data
        )
        
        # Use Investigator to make decisions
        if self.email_investigator.is_eligible_for_verification(state_with_email):
            # Use Worker to handle verification
            final_state = await self.verification_worker.process(state_with_email)
        else:
            final_state = state_with_email
            
        return final_state
```

### Key Differences

1. **Single `process` Method**: Maintains a single entry point for orchestration logic.
2. **Dependencies Injected Through Constructor**: Enhances testability and flexibility by injecting dependencies.
3. **No Direct Data Manipulation**: Utilizes Workers and Fetchers for data processing.
4. **Error Propagation**: Allows errors to bubble up without handling them internally.
5. **Uses Investigators for Decision-Making**: Delegates business logic decisions to Investigators.
6. **Immutable State**: Works with immutable state objects to ensure consistency.

### Supporting Components

For completeness, here’s how the supporting components might look:

```python
class UserFetcher:
    async def get_user(self, user_id: str) -> UserData:
        # Fetch user data from external system
        return UserData(...)

class EmailWorker:
    async def process(self, state: UserState, user_data: UserData) -> UserState:
        # Process email data and return new state
        return UserState(
            user_id=state.user_id,
            email=user_data.email,
            is_verified=state.is_verified,
            created_at=state.created_at
        )

class VerificationWorker:
    async def process(self, state: UserState) -> UserState:
        # Handle verification and return new state
        return UserState(
            user_id=state.user_id,
            email=state.email,
            is_verified=True,
            created_at=state.created_at
        )

class EmailInvestigator:
    def is_eligible_for_verification(self, state: UserState) -> bool:
        # Single condition check
        return state.email is not None and '@' in state.email
```

---

## Investigators

**Investigators** enforce business rules and validations, maintaining data integrity by answering specific true/false questions about the data. They are pure functions that check only one condition per method.

### ❌ Incorrect Implementation

```python
class OrderInvestigator:
    @staticmethod
    def is_order_valid(order_state: OrderState) -> bool:
        # INCORRECT: Multiple if/then statements
        if order_state.total_amount <= 0:
            return False
        
        if order_state.items_count == 0:
            return False
            
        if order_state.delivery_address is None:
            return False
            
        return True

    @staticmethod
    def can_process_order(order_state: OrderState) -> bool:
        # INCORRECT: Multiple conditions with multiple if/then statements
        if order_state.status == "PENDING":
            if order_state.payment_verified:
                return True
            else:
                return False
        return False
```

The above implementation is incorrect because:

1. **Multiple If/Then Statements**: Uses several conditional checks within a single method instead of consolidating them.
2. **Nested Conditions**: Contains nested if/then statements, making the logic harder to follow.
3. **Split Logic**: Splits related logic across multiple decision points rather than consolidating it.

### ✅ Correct Implementation

```python
class OrderInvestigator:
    @staticmethod
    def is_order_valid(order_state: OrderState) -> bool:
        # CORRECT: Single if statement with multiple conditions combined
        return (
            order_state.total_amount > 0 and 
            order_state.items_count > 0 and 
            order_state.delivery_address is not None
        )

    @staticmethod
    def can_process_order(order_state: OrderState) -> bool:
        # CORRECT: Single condition evaluation
        return order_state.status == "PENDING" and order_state.payment_verified

    @staticmethod
    def is_high_value_order(order_state: OrderState) -> bool:
        # CORRECT: Single condition evaluation
        return order_state.total_amount >= 1000.00

    @staticmethod
    def requires_special_handling(order_state: OrderState) -> bool:
        # CORRECT: Complex condition in a single evaluation
        return (
            order_state.has_fragile_items or
            order_state.requires_refrigeration or
            order_state.is_oversized
        )
```

### Key Differences

1. **Single Truth Evaluation**: Each method evaluates exactly one truth condition, even if it involves multiple factors.
2. **Direct Boolean Returns**: Returns boolean expressions directly without using if/then control flow.
3. **Clarity of Purpose**: Each method has a clear, single responsibility for checking one specific aspect of the order.
4. **No Control Flow**: Avoids using control flow statements (if/then) and relies on boolean operations (`and`, `or`) to combine conditions.

**Remember:**

- It's acceptable to have multiple conditions in a single boolean expression.
- Avoid multiple if/then statements or nested structures.
- Each method should evaluate exactly one truth condition.
- Express logic as a single boolean expression, even if it’s complex.

This approach ensures that Investigators remain focused, maintainable, and easy to test while allowing for complex validation logic when necessary.

---

## Error Components

**Error** components provide an error handling framework, acting as type guards to validate state objects. They utilize Investigators to assess data and raise exceptions if validations fail. Delegators do not interact directly with Error components; instead, Workers and Investigators handle validations and error assertions.

### ❌ Incorrect Implementation

```python
from typing import Optional
from pydantic import BaseModel

class UserState(BaseModel, frozen=True):
    user_id: str
    email: str
    age: int
    is_active: bool

class BadUserError:
    """
    This implementation violates several principles of the architectural pattern
    """
    @staticmethod
    def validate_user(user: UserState) -> None:
        # ❌ Multiple conditions in one method
        # ❌ Direct business logic without using Investigators
        # ❌ Mixing different types of validations
        if not user.is_active:
            raise ValueError("User is not active")
        
        if user.age < 18:
            raise ValueError("User must be 18 or older")
        
        if not user.email or '@' not in user.email:
            raise ValueError("Invalid email format")

    @staticmethod
    def check_user_eligibility(user: UserState, premium_feature: bool = False) -> bool:
        # ❌ Returns boolean instead of raising assertion
        # ❌ Contains business logic that should be in an Investigator
        # ❌ Handles multiple conditions
        return user.is_active and user.age >= 21 and premium_feature

    def process_user(self, user: UserState) -> None:
        # ❌ Contains processing logic that should be in a Worker
        # ❌ Modifies state
        # ❌ Non-static method
        self.validate_user(user)
        # ... more processing
```

### ✅ Correct Implementation

```python
from typing import Optional
from pydantic import BaseModel

class UserState(BaseModel, frozen=True):
    user_id: str
    email: str
    age: int
    is_active: bool

class UserInvestigator:
    """
    Separate Investigator class for business rule validation
    """
    @staticmethod
    def is_active(user: UserState) -> bool:
        return user.is_active

    @staticmethod
    def is_adult(user: UserState) -> bool:
        return user.age >= 18

    @staticmethod
    def has_valid_email(user: UserState) -> bool:
        return bool(user.email and '@' in user.email)

class UserError:
    """
    Correct implementation following the architectural pattern
    """
    @staticmethod
    def assert_user_active(user: UserState) -> None:
        """
        Single assertion method using an Investigator
        """
        if not UserInvestigator.is_active(user):
            raise ValueError("User must be active")

    @staticmethod
    def assert_user_adult(user: UserState) -> None:
        """
        Single assertion method using an Investigator
        """
        if not UserInvestigator.is_adult(user):
            raise ValueError("User must be 18 or older")

    @staticmethod
    def assert_valid_email(user: UserState) -> None:
        """
        Single assertion method using an Investigator
        """
        if not UserInvestigator.has_valid_email(user):
            raise ValueError("Invalid email format")

# Example usage in a Worker
class UserWorker:
    @staticmethod
    def process_user(user: UserState) -> UserState:
        # Worker uses Error assertions
        UserError.assert_user_active(user)
        UserError.assert_user_adult(user)
        UserError.assert_valid_email(user)
        
        # Continue with processing...
        return user
```

### Key Differences and Explanations

1. **Separation of Concerns**:
   - **Incorrect**: Mixes business logic, validation, and processing within the Error class.
   - **Correct**: Separates business rule validation into `UserInvestigator` and assertion methods into `UserError`.

2. **Single Responsibility**:
   - **Incorrect**: Methods perform multiple validations and contain processing logic.
   - **Correct**: Each assertion method focuses on a single validation, leveraging Investigators for business rules.

3. **Pure Functions**:
   - **Incorrect**: Includes state modification and non-static methods.
   - **Correct**: Uses only static methods without modifying state.

4. **Error Handling**:
   - **Incorrect**: Mixes boolean returns with assertions.
   - **Correct**: Consistently uses assertions to raise exceptions.

5. **Business Logic Location**:
   - **Incorrect**: Embeds business logic directly in the Error class.
   - **Correct**: Delegates business logic to Investigators.

6. **Method Structure**:
   - **Incorrect**: Methods perform multiple tasks.
   - **Correct**: Methods are focused and single-purpose.

### Usage Example

```python
# Example Delegator using the correct implementation
class UserDelegator:
    @staticmethod
    async def process(user: UserState) -> UserState:
        try:
            # Delegator doesn't handle errors directly
            return await UserWorker.process_user(user)
        except ValueError as e:
            # Error handling is delegated to the system executing the Delegator
            raise

# System level execution
async def execute_workflow(user_data: dict):
    try:
        user_state = UserState(**user_data)
        result = await UserDelegator.process(user_state)
        return result
    except ValueError as e:
        # Handle errors at the system level
        logger.error(f"Validation error: {str(e)}")
        raise
```

The correct implementation demonstrates:

- **Clear Separation**: Validation logic is separated between Investigators and Error components.
- **Single Responsibility**: Each method handles one specific validation.
- **Pure Functions**: No side effects or state modifications.
- **Consistent Error Handling**: Uses assertions to manage errors uniformly.
- **Proper Delegation**: Business rules are handled by Investigators, keeping Error components focused on validations.

This structure enhances maintainability, testability, and clarity, adhering to the architectural pattern's principles.

---

## State Models

**State** models define immutable data structures that traverse the system, ensuring consistency and reliability throughout the processing pipeline. They validate the shape of the data but remain agnostic to business rules, focusing solely on data structure integrity.

### ❌ Incorrect Implementation

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional, List

class UserProfileState(BaseModel):
    user_id: str
    email: str
    name: str
    age: int
    subscription_status: str
    last_login: datetime
    preferences: dict

    # ❌ Contains business logic
    def is_premium_user(self) -> bool:
        return self.subscription_status == "premium"

    # ❌ Modifies state
    def update_last_login(self):
        self.last_login = datetime.now()

    # ❌ Validates business rules
    def validate_age(self):
        if self.age < 18:
            raise ValueError("User must be 18 or older")

    # ❌ Complex data transformation
    def get_formatted_preferences(self) -> dict:
        formatted = {}
        for key, value in self.preferences.items():
            formatted[key.lower()] = str(value).upper()
        return formatted

    # ❌ External system interaction
    async def sync_with_database(self, db_connection):
        await db_connection.update_user(self.dict())
```

#### Problems with This Implementation:

1. **Business Logic**: Contains methods like `is_premium_user` that enforce business rules.
2. **State Modification**: Allows modification of state through methods like `update_last_login`.
3. **Business Rule Validation**: Implements validation logic directly within the model.
4. **Data Transformation**: Performs data transformations with methods like `get_formatted_preferences`.
5. **External System Interaction**: Interacts with external systems using methods like `sync_with_database`.
6. **Immutability**: Lacks `frozen=True`, making the model mutable.

### ✅ Correct Implementation

```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Dict, Literal, Optional
from uuid import UUID

class UserProfileState(BaseModel):
    """
    Represents the immutable state of a user profile.
    Only validates data structure, not business rules.
    """
    user_id: UUID
    email: str
    name: str
    age: int = Field(ge=0)  # ✅ Structural validation only
    subscription_status: Literal["free", "premium", "enterprise"]
    last_login: datetime
    preferences: Dict[str, str]
    created_at: datetime = Field(default_factory=datetime.utcnow)
    version: int = 1

    class Config:
        frozen = True  # ✅ Ensures immutability
        json_encoders = {
            UUID: str,
            datetime: lambda v: v.isoformat()
        }
        
        # ✅ Optional: Custom error messages for structural validation
        error_msg_templates = {
            'type_error.integer': 'age must be a valid integer',
            'value_error.missing': '{field_name} is required'
        }

    class Meta:
        description = "Represents user profile data at a specific point in time"
        version = "1.0.0"
```

### Key Aspects of the Correct Implementation

1. **Immutability**:
   - Utilizes `frozen=True` in Pydantic models to prevent state modification after creation.
   - No methods are included that attempt to modify the state.

2. **Structure-Only Validation**:
   - Employs Pydantic's `Field` for basic type and range validation.
   - Does not include business rule validations.
   - Uses precise types with `Literal` for enum-like fields.

3. **Clear Purpose**:
   - Represents data structure without embedding business logic or transformations.
   - Includes metadata such as `created_at` and `version` for tracking.

4. **No Business Logic**:
   - Excludes methods for business rules, transformations, or external interactions.

### Usage Example

```python
# Creating a new state instance
from uuid import uuid4

# ✅ Correct usage
try:
    profile = UserProfileState(
        user_id=uuid4(),
        email="user@example.com",
        name="John Doe",
        age=25,
        subscription_status="premium",
        last_login=datetime.utcnow(),
        preferences={"theme": "dark", "language": "en"}
    )
except ValueError as e:
    # Handle structural validation errors
    print(f"Invalid data structure: {e}")

# ✅ Creating a new state from an existing one
new_profile = UserProfileState(
    **profile.dict(),
    last_login=datetime.utcnow(),
    version=profile.version + 1
)

# ❌ This would raise an error due to immutability
# profile.age = 26  # FrozenInstanceError

# ❌ Business logic belongs in Investigators, not here
# if profile.subscription_status == "premium":  # Should be in an Investigator class
```

### Validation Example (Where It Should Go)

```python
# ✅ This belongs in an Investigator class
class UserProfileInvestigator:
    @staticmethod
    def is_premium_user(state: UserProfileState) -> bool:
        return state.subscription_status == "premium"

# ✅ This belongs in an Error class
class UserProfileError:
    @staticmethod
    def assert_valid_age(state: UserProfileState) -> None:
        if not UserProfileInvestigator.is_adult(state):
            raise ValueError(f"User must be 18 or older. Current age: {state.age}")

# ✅ This belongs in a Worker class
class UserProfileWorker:
    @staticmethod
    def format_preferences(state: UserProfileState) -> UserProfileState:
        formatted_prefs = {
            k.lower(): str(v).upper() 
            for k, v in state.preferences.items()
        }
        return UserProfileState(
            **state.dict(),
            preferences=formatted_prefs,
            version=state.version + 1
        )
```

The correct implementation ensures that the **State** class remains focused solely on data structure and validation, leaving business logic, transformations, and other concerns to their respective components in the architecture.

---

## Conclusion

This architectural pattern promotes a **clean**, **maintainable**, and **scalable** codebase by enforcing clear boundaries and responsibilities across different components. Adhering to best practices such as **immutability**, **single responsibility**, and **error handling** ensures that each part of the system is **predictable**, **testable**, and **easy to reason about**.

### Additional Recommendations

1. **Consistent Naming Conventions**: Ensure method and class names clearly describe their functionality.
2. **Dependency Injection**: Inject dependencies (e.g., Workers, Fetchers) to enhance testability and flexibility.
3. **Async Support**: Implement asynchronous patterns where applicable to improve scalability.
4. **Comprehensive Documentation**: Maintain detailed and consistent documentation across all components.
5. **Testing Strategy**:
   - **Unit Tests**: For individual components.
   - **Integration Tests**: To validate interactions between components.
   - **Mocking External Dependencies**: Use mocks for external systems to isolate tests.
6. **Error Handling Strategy**: Define clear strategies for error propagation and handling at different layers.
7. **Scalability Considerations**: Design components to be stateless where possible to facilitate horizontal scaling.
8. **Security Best Practices**:
   - Validate and sanitize all external inputs.
   - Handle sensitive data securely using hashing and encryption.
   - Implement access controls and authentication mechanisms as required.
9. **Version Control and Deployment**:
   - Use version control systems (like Git) effectively.
   - Implement CI/CD pipelines to automate testing and deployment processes.

By following these guidelines and maintaining the integrity of each component's responsibilities, you can build a resilient, efficient, and scalable system that meets complex business requirements and adapts to evolving technological landscapes.

---
