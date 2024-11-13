# Architectural Pattern Guide

## Introduction

Creating systems that are **maintainable**, **scalable**, and **reliable** is crucial in software development. As applications grow in complexity, adopting an architectural pattern that promotes clear separation of concerns, modularity, and efficient error handling becomes essential.

This guide introduces the **Team Pattern**—a comprehensive architectural framework designed to streamline the development process by defining distinct roles and responsibilities for various components within a system. By adhering to this pattern, development teams can ensure that each part of the application is focused, testable, and easy to maintain, ultimately leading to higher quality and more dependable software.

### Core Components

The Team Pattern is built around six key components, each with a specific role:

1. **Workers**: Handle the core business logic and data transformations, ensuring operations are executed correctly and efficiently. Workers are invoked exclusively by Delegators and must use Error class methods to validate data within the state they process.

2. **Fetchers**: Manage data access and retrieval from external systems, abstracting complexities and providing clean interfaces for data consumption. Fetchers return **State Objects** representing the extracted and validated data and utilize Error components for validation. They do not handle errors internally beyond validation.

3. **Delegators**: Orchestrate workflows by coordinating between Fetchers, Workers, and Investigators to manage complex business processes seamlessly. Delegators contain only a single `process` method, avoid direct data manipulation, do not manage state beyond the provided state objects, and rely on Investigators for decision-making without interacting with Error components.

4. **Investigators**: Enforce business rules and validations, maintaining data integrity by answering specific true/false questions about the data. They are pure functions that check only one condition per method.

5. **Error Components**: Provide an error handling framework, acting as type guards to validate state objects. They utilize Investigators to assess data and raise exceptions if validations fail. Delegators do not interact directly with Error components; instead, Workers and Fetchers handle validations and error assertions.

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

This guide delves into each component of the Team Pattern, providing examples of both compliant and non-compliant implementations to illustrate best practices and common pitfalls. By following the guidelines outlined here, development teams can leverage the Team Pattern to build high-quality, dependable software systems.

## Table of Contents

1. [Workers](#workers)
   - [❌ Bad Example (Non-Compliant Worker)](#bad-example-non-compliant-worker)
   - [✅ Good Example (Compliant Worker)](#good-example-compliant-worker)
   - [Key Differences](#key-differences)
2. [Fetchers](#fetchers)
   - [❌ Bad Implementation](#bad-implementation)
   - [✅ Good Implementation (Compliant Fetcher)](#good-implementation-compliant-fetcher)
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

**Workers** handle the core logic and transformations, ensuring that business operations are executed correctly and efficiently. They are exclusively invoked by Delegators and must utilize Error class methods to validate the data within the state they process. Workers can contain multiple independent methods, each responsible for a specific task. However, Worker methods must not call other Workers or their methods, maintaining independence and ensuring Workers are the final step in their respective processing pipelines.

### ❌ Bad Example (Non-Compliant Worker)

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

        # ❌ Direct data manipulation on a state object
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
        # ❌ Worker containing multiple methods that aren't themselves workers (violation) 
        return '@' in email

    def handle_weak_password(self, user_state: UserState) -> UserState:
        # ❌ Throwing an exception within a worker (violation)
        raise ValueError("Password too weak")
```

### ✅ Good Example (Compliant Worker)

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
    def raise_if_invalid_email(email: str) -> None:
        if not UserRegistrationInvestigator.is_valid_email(email):
            raise ValueError("Invalid email format")

    @staticmethod
    def raise_if_invalid_password_length(password: str) -> None:
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
        - Utilizes Error methods for validation
        - Maintains immutability by creating new state objects

    **Example:**
        ```python
        initial_state = UserState(
            email="User@Example.com",
            password="securepass123"
        )
        worker = UserRegistrationWorker()
        processed_state = worker.normalize_email(initial_state)
        processed_state = worker.assign_creation_timestamp(processed_state)
        worker.log_registration(processed_state)
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
    def normalize_email(state: UserState) -> UserState:
        """
        Normalizes the user's email to lowercase.

        **Parameters:**
            - `state (UserState)`: The user registration state.

        **Returns:**
            - `UserState`: A new state with the email normalized.
        """
        # ✅ Validate email using Error methods
        UserRegistrationErrors.raise_if_invalid_email(state.email)

        # ✅ Return new immutable state with normalized email
        return UserState(
            email=state.email.lower(),
            password=state.password,
            created_at=datetime.now()
        )
    
    @staticmethod
    def assign_creation_timestamp(state: UserState) -> UserState:
        """
        Assigns the current datetime as the creation timestamp.

        **Parameters:**
            - `state (UserState)`: The user registration state.

        **Returns:**
            - `UserState`: A new state with the creation timestamp assigned.
        """
        return UserState(
            email=state.email,
            password=state.password,
            created_at=datetime.now()
        )
    
    @staticmethod
    def log_registration(state: UserState) -> None:
        """
        Logs the user registration details.

        **Parameters:**
            - `state (UserState)`: The user registration state.
        """
        print(f"User registered with email: {state.email} at {state.created_at}")
```

### Key Differences

1. **Declarative Method Names**: Instead of generic names like `process_registration`, methods are named based on their specific tasks, such as `normalize_email`, `assign_creation_timestamp`, and `log_registration`.
   
2. **Single Parameter (State Object)**: All Worker methods accept only the `UserState` object, ensuring that Workers do not depend on external parameters. Any additional data required must be encapsulated within the state object.
   
3. **Multiple Independent Methods**: The Worker contains multiple methods that handle distinct tasks without interdependencies or calls to other Workers.
   
4. **Comprehensive Documentation**: Each method includes detailed docstrings explaining its purpose, parameters, returns, and potential exceptions.
   
5. **Error Handling**: Utilizes dedicated Error class methods that leverage Investigators for validation instead of performing direct validation.
   
6. **No Branching**: Avoids conditional logic within Worker methods, delegating decision-making to Investigators.
   
7. **Immutability**: Creates new state objects rather than modifying existing ones, maintaining immutability throughout the processing.
   
8. **No Worker Chaining**: Methods do not call other Workers, ensuring that Workers remain the final step in their respective processing pipelines.
   
9. **Clear Separation of Concerns**: Each method within the Worker handles its specific responsibility without overlapping functionalities.

---

## Fetchers

**Fetchers** manage data access and retrieval, abstracting the complexities of external systems and providing clean interfaces for data consumption. They utilize **Error Components** for validation and return **State Objects** representing the extracted and validated data. Fetchers do not handle errors internally beyond validation; any errors encountered during data fetching propagate up to be managed by the system executing the Delegator.

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

#### Issues with the Incorrect Implementation:

1. **State Management**: Maintains internal state through caching, which violates the pattern's statelessness principle.
2. **Error Handling**: Handles errors internally instead of allowing them to propagate to be managed by higher-level components.
3. **Data Manipulation**: Directly manipulates fetched data, introducing business logic into Fetchers.
4. **Multiple Methods**: Contains additional utility methods beyond data fetching, violating single responsibility.

### ✅ Good Implementation (Compliant Fetcher)

```python
from typing import Optional
import requests
from pydantic import BaseModel, Field, ValidationError, EmailStr
from datetime import datetime
from uuid import UUID

class UserState(BaseModel):
    user_id: UUID
    email: EmailStr
    username: str
    created_at: datetime
    is_active: bool = Field(default=True)
    
    class Config:
        frozen = True

class FetcherInvestigator:
    """
    Investigator class to validate fetched data.
    """
    @staticmethod
    def has_required_fields(data: dict) -> bool:
        required_fields = {"id", "username", "email", "created_at", "is_active"}
        return required_fields.issubset(data.keys())

    @staticmethod
    def is_email_valid(email: str) -> bool:
        return '@' in email

class FetcherErrors:
    """
    Error component to handle validation errors in Fetchers.
    """
    @staticmethod
    def raise_if_missing_fields(data: dict) -> None:
        if not FetcherInvestigator.has_required_fields(data):
            missing = {"id", "username", "email", "created_at", "is_active"} - data.keys()
            raise ValueError(f"Missing fields in fetched data: {missing}")

    @staticmethod
    def raise_if_invalid_email(email: str) -> None:
        if not FetcherInvestigator.is_email_valid(email):
            raise ValueError("Invalid email format in fetched data")

class UserFetcher:
    """Fetches user data from the external API service."""

    def __init__(self, base_url: str):
        """
        Initialize the UserFetcher with the API base URL.

        **Parameters:**
            - `base_url (str)`: The base URL of the API service
        """
        self.base_url = base_url

    async def get_user(self, state: UserState) -> UserState:
        """
        Retrieves user data from the API for a given user ID.

        **Summary:**
            Fetches user information from the external API service, validates it, and returns it as
            a validated `UserState` model.

        **Remarks:**
            - Performs a direct API call without caching.
            - All errors are propagated to the caller for handling.
            - Utilizes Error Components for validation.

        **Example:**
            ```python
            fetcher = UserFetcher("https://api.example.com")
            try:
                user_state = await fetcher.get_user(123)
                print(f"Found user: {user_state.username}")
            except (requests.RequestException, ValueError, ValidationError) as e:
                # Handle error at the caller level
                pass
            ```

        **Parameters:**
            - `state (UserState)`: The user state object containing the current user state

        **Returns:**
            - `UserState`: An immutable state object containing the validated user data

        **Throws:**
            - `requests.RequestException`: When the API request fails
            - `ValueError`: When fetched data fails validation checks
            - `ValidationError`: When the response data doesn't match the expected schema

        **See:**
            - `UserState`
            - `FetcherInvestigator`
            - `FetcherErrors`
            - [API Documentation](https://api.example.com/docs#get-user)
        """
        response = requests.get(f"{self.base_url}/users/{state.user_id}")
        
        # Let the response error propagate up
        response.raise_for_status()
        
        # Parse JSON response
        data = response.json()
        
        # Validate presence of required fields
        FetcherErrors.raise_if_missing_fields(data)
        
        # Validate email format
        FetcherErrors.raise_if_invalid_email(data.get("email", ""))
        
        # Convert fetched data to UserState
         return UserState(
             user_id=UUID(data["id"]),
             email=data["email"],
             username=data["username"],
             created_at=datetime.fromisoformat(data["created_at"]),
             is_active=data["is_active"]
         )
```

### Key Differences and Improvements in the Good Implementation

1. **State Object Return**:
   - **Bad**: Returns raw dictionaries with potential errors.
   - **Good**: Returns a **State Object** (`UserState`) that is immutable and validated.

2. **Assertions and Validations**:
   - **Good**: Utilizes **Error Components** (`FetcherErrors`) and **Investigators** (`FetcherInvestigator`) to validate fetched data before converting it into a state object.
   - **Error Handling**: Raises exceptions (`ValueError`, `ValidationError`) directly within the Fetcher when validations fail, allowing higher-level components to handle them appropriately.

3. **Immutable State Models**:
   - **Good**: Uses Pydantic's `BaseModel` with `frozen=True` to ensure immutability, preventing unintended side-effects.

4. **Clear Separation of Concerns**:
   - **Good**: Fetchers focus solely on data retrieval and initial validation, leaving business logic to Workers and Investigators.

5. **Comprehensive Documentation**:
   - **Good**: Detailed docstrings explaining the purpose, parameters, returns, and exceptions of each method.

6. **No Internal State Management**:
   - **Good**: Does not maintain internal caches or states, adhering to the statelessness principle of Fetchers.

7. **Error Propagation**:
   - **Good**: Allows errors to propagate up, enabling centralized error handling at the system level.

8. **Data Conversion and Validation**:
   - **Good**: Converts raw fetched data into structured `UserState` objects, ensuring data integrity through type validation and explicit field definitions.

### Example Usage in a Delegator

To illustrate how the compliant Fetcher integrates within the overall architecture, here's how it can be utilized within a Delegator:

```python
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

    async def process(self, user_id: int) -> UserState:
        """
        Orchestrates the user verification workflow.

        **Parameters:**
            - `user_id (int)`: The unique identifier of the user.

        **Returns:**
            - `UserState`: The final user state after processing.
        """
        # Fetch user data using Fetcher
        user_state = await self.user_fetcher.get_user(user_id)
        
        # Use Worker to normalize email
        state_with_normalized_email = self.email_worker.normalize_email(user_state)
        
        # Use Investigator to make decisions
        if self.email_investigator.is_eligible_for_verification(state_with_normalized_email):
            # Use Worker to handle verification
            final_state = self.verification_worker.verify_user_email(state_with_normalized_email)
        else:
            final_state = state_with_normalized_email
            
        return final_state
```

**Note:** Since Fetchers now return state objects, Delegators can directly work with these validated and immutable objects, ensuring that downstream Workers receive consistent and reliable data.

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

    def fetch_user_data(self, user_id: str) -> dict:
        # Direct data fetching instead of using Fetchers
        return {"email": "user@example.com"}

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
from uuid import UUID

class UserState(BaseModel):
    user_id: UUID
    email: Optional[str] = None
    is_verified: bool = False
    created_at: datetime = Field(default_factory=datetime.utcnow)
    
    class Config:
        frozen = True

class UserData(BaseModel):
    id: int
    email: str

class UserFetcher:
    async def get_user(self, user_id: int) -> UserState:
        # Fetch user data from external system
        # Example implementation
        # In a real scenario, this would perform an API call or database query
        # For demonstration, we simulate fetched data
        fetched_data = {
            "id": user_id,
            "email": "User@Example.com",
            "username": "exampleuser",
            "created_at": "2023-10-01T12:00:00Z",
            "is_active": True
        }
        
        # Validate and convert fetched data to UserState
        FetcherErrors.raise_if_missing_fields(fetched_data)
        FetcherErrors.raise_if_invalid_email(fetched_data.get("email", ""))
        
        return UserState(
            user_id=UUID(int=fetched_data["id"]),  # Assuming UUID based on ID
            email=fetched_data["email"],
            is_verified=False,
            created_at=datetime.fromisoformat(fetched_data["created_at"].replace("Z", "+00:00"))
        )

class EmailWorker:
    def normalize_email(self, state: UserState) -> UserState:
        """
        Normalizes the user's email to lowercase.

        **Parameters:**
            - `state (UserState)`: The current user state.

        **Returns:**
            - `UserState`: A new state with the normalized email.
        """
        # Validate email using Error methods
        UserRegistrationErrors.raise_if_invalid_email(state.email)
        
        # Return new immutable state with normalized email
        return UserState(
            user_id=state.user_id,
            email=state.email.lower() if state.email else None,
            is_verified=state.is_verified,
            created_at=state.created_at
        )

class VerificationWorker:
    def verify_user_email(self, state: UserState) -> UserState:
        """
        Marks the user's email as verified.

        **Parameters:**
            - `state (UserState)`: The current user state.

        **Returns:**
            - `UserState`: A new state with the email marked as verified.
        """
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

    async def process(self, user_id: int) -> UserState:
        """
        Orchestrates the user verification workflow.

        **Parameters:**
            - `state: UserState`: The unique identifier of the user.

        **Returns:**
            - `UserState`: The final user state after processing.
        """
        # Fetch user data using Fetcher
        user_state = await self.user_fetcher.get_user(state.user_id)
        
        # Use Worker to normalize email
        state_with_normalized_email = self.email_worker.normalize_email(user_state)
        
        # Use Investigator to make decisions
        if self.email_investigator.is_eligible_for_verification(state_with_normalized_email):
            # Use Worker to handle verification
            final_state = self.verification_worker.verify_user_email(state_with_normalized_email)
        else:
            final_state = state_with_normalized_email
            
        return final_state
```

### Key Differences

1. **Single `process` Method**: Maintains a single entry point for orchestration logic.
   
2. **Dependencies Injected Through Constructor**: Enhances testability and flexibility by injecting dependencies.
   
3. **No Direct Data Manipulation**: Utilizes Workers and Fetchers for data processing instead of manipulating state directly within the Delegator.
   
4. **Error Propagation**: Allows errors to bubble up without handling them internally, adhering to the principle that Delegators should not manage errors directly.
   
5. **Uses Investigators for Decision-Making**: Delegates business logic decisions to Investigators, keeping Delegators focused on orchestration.
   
6. **Immutable State**: Works with immutable state objects to ensure consistency.
   
7. **Multiple Independent Worker Methods**: Each Worker method performs a distinct, independent task without interdependencies or calls to other Workers.
   
8. **Clear Separation of Concerns**: Each component handles its specific responsibility without overlapping functionalities.

### Supporting Components

For completeness, here’s how the supporting components might look:

```python
class UserFetcher:
    async def get_user(self, user_id: int) -> UserState:
        # Fetch user data from external system
        # Example implementation
        # In a real scenario, this would perform an API call or database query
        fetched_data = {
            "id": user_id,
            "email": "User@Example.com",
            "username": "exampleuser",
            "created_at": "2023-10-01T12:00:00Z",
            "is_active": True
        }
        
        # Validate and convert fetched data to UserState
        FetcherErrors.raise_if_missing_fields(fetched_data)
        FetcherErrors.raise_if_invalid_email(fetched_data.get("email", ""))
        
        return UserState(
           user_id=UUID(int=fetched_data["id"]),  # Assuming UUID based on ID
           email=fetched_data["email"],
           is_verified=False,
           created_at=datetime.fromisoformat(fetched_data["created_at"].replace("Z", "+00:00"))
         )

class EmailWorker:
    def normalize_email(self, state: UserState) -> UserState:
        # Normalize email and return new state
        return UserState(
            user_id=state.user_id,
            email=state.email.lower() if state.email else None,
            is_verified=state.is_verified,
            created_at=state.created_at
        )

class VerificationWorker:
    def verify_user_email(self, state: UserState) -> UserState:
        # Verify email and return new state
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
          return True

        if order_state.name === 'SomeName'
          return true
          
        return False
```

The above implementation is incorrect because:

1. **Multiple If/Then Statements**: Uses several conditional checks within a single method instead of consolidating them.
2. **Split Logic**: Splits related logic across multiple decision points rather than consolidating it.

### ✅ Correct Implementation

```python
class OrderInvestigator:
    @staticmethod
    def has_positive_total_amount(order_state: OrderState) -> bool:
        # Checks if the total amount is positive
        return order_state.total_amount > 0

    @staticmethod
    def has_items(order_state: OrderState) -> bool:
        # Checks if there is at least one item
        return order_state.items_count > 0

    @staticmethod
    def has_delivery_address(order_state: OrderState) -> bool:
        # Checks if the delivery address is provided
        return order_state.delivery_address is not None

    @staticmethod
    def is_order_pending(order_state: OrderState) -> bool:
        # Checks if the order status is pending
        return order_state.status == "PENDING"

    @staticmethod
    def is_payment_verified(order_state: OrderState) -> bool:
        # Checks if the payment is verified
        return order_state.payment_verified

    @staticmethod
    def has_fragile_items(order_state: OrderState) -> bool:
        # Checks if the order has fragile items
        return order_state.has_fragile_items

    @staticmethod
    def requires_refrigeration(order_state: OrderState) -> bool:
        # Checks if the order requires refrigeration
        return order_state.requires_refrigeration

    @staticmethod
    def is_oversized(order_state: OrderState) -> bool:
        # Checks if the order is oversized
        return order_state.is_oversized
```

### Key Differences

1. **Single Truth Evaluation**: Each method evaluates exactly one truth condition, even if it involves multiple factors.
2. **Direct Boolean Returns**: Returns boolean expressions directly without using if/then control flow.
3. **Clarity of Purpose**: Each method has a clear, single responsibility for checking one specific aspect of the order.
4. **No Control Flow**: Avoids using control flow statements (`if/then`) and relies on boolean operations (`and`, `or`) to combine conditions.

**Remember:**

- It's acceptable to have multiple conditions in a single boolean expression.
- Avoid multiple if/then statements or nested structures.
- Each method should evaluate exactly one truth condition.
- Express logic as a single boolean expression, even if it’s complex.

This approach ensures that Investigators remain focused, maintainable, and easy to test while allowing for complex validation logic when necessary.

---

## Error Components

**Error Components** provide an error handling framework, acting as type guards to validate state objects. They utilize **Investigators** to assess data and raise exceptions if validations fail. Delegators do not interact directly with Error components; instead, Workers and Fetchers handle validations and error assertions.

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
    def raise_if_user_not_active(user: UserState) -> None:
        """
        Raises an exception if the user is not active.
        """
        if not UserInvestigator.is_active(user):
            raise UserError("User must be active")

    @staticmethod
    def raise_if_user_not_adult(user: UserState) -> None:
        """
        Raises an exception if the user is not an adult.
        """
        if not UserInvestigator.is_adult(user):
            raise UserError("User must be 18 or older")

    @staticmethod
    def raise_if_invalid_email(user: UserState) -> None:
        """
        Raises an exception if the user's email is invalid.
        """
        if not UserInvestigator.has_valid_email(user):
            raise UserError("Invalid email format")

# Example usage in a Worker
class UserWorker:
    @staticmethod
    def process_user(user: UserState) -> UserState:
        # Worker uses Error methods to validate
        UserError.raise_if_user_not_active(user)
        UserError.raise_if_user_not_adult(user)
        UserError.raise_if_invalid_email(user)
        
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
   - **Correct**: Consistently uses methods that raise exceptions directly.

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

    async def process(self, user_id: int) -> UserState:
        """
        Orchestrates the user verification workflow.

        **Parameters:**
            - `state (UserState)`: The unique identifier of the user.

        **Returns:**
            - `UserState`: The final user state after processing.
        """
         # Fetch user data using Fetcher
         user_state = await self.user_fetcher.get_user(state.user_id)
         
         # Use Worker to normalize email
         state_with_normalized_email = self.email_worker.normalize_email(user_state)
         
         # Use Investigator to make decisions
         if self.email_investigator.is_eligible_for_verification(state_with_normalized_email):
             # Use Worker to handle verification
             final_state = self.verification_worker.verify_user_email(state_with_normalized_email)
         else:
             final_state = state_with_normalized_email
             
         return final_state
```

**Note:** The Delegator remains focused on orchestrating the workflow without handling errors directly. Errors raised by Workers or Fetchers propagate up to be managed by the system executing the Delegator.

---

## State Models

**State Models** define immutable data structures that traverse the system, ensuring consistency and reliability throughout the processing pipeline. They validate the shape of the data but remain agnostic to business rules, focusing solely on data structure integrity.

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
from pydantic import BaseModel, Field, EmailStr
from datetime import datetime
from typing import Dict, Literal, Optional
from uuid import UUID

class UserProfileState(BaseModel):
    """
    Represents the immutable state of a user profile.
    Only validates data structure, not business rules.
    """
    user_id: UUID
    email: EmailStr
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

    @staticmethod
    def is_adult(state: UserProfileState) -> bool:
        return state.age >= 18

# ✅ This belongs in an Error class
class UserProfileError:
    @staticmethod
    def raise_if_invalid_age(state: UserProfileState) -> None:
        if not UserProfileInvestigator.is_adult(state):
            raise ValueError(f"User must be 18 or older. Current age: {state.age}")

    @staticmethod
    def raise_if_not_premium(state: UserProfileState) -> None:
        if not UserProfileInvestigator.is_premium_user(state):
            raise ValueError("User is not a premium member")

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

By following these guidelines and maintaining the integrity of each component's responsibilities, you can build a dependable, efficient, and scalable system that meets complex business requirements and adapts to evolving technological landscapes.

---

*This guide serves as a comprehensive reference for implementing the architectural pattern, ensuring best practices are followed, and promoting a robust software development lifecycle.*

