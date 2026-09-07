- **Stranger Fig** - Basically when you have a shit system and rather than do a full rewrite you basically look at isolated functionality and start replacing that by having behind a feature flag of sorts. come from how fig plant takes over a host planet and slowly takes over and kills it. 
- **Requirements** say what the system should do
- **invariants** say what must _never_ be violated while doing it.  “every order has at least one line item”,  The key word is _always_: an invariant isn’t a temporary condition or a goal; it’s a permanent truth about valid states.
- **unbounded coupling** occurs when the dependency between two or more components has no constraint on how far its impact can cascade. Rather than being contained within an explicit boundary, a single change, failure, or scaling demand in one component can trigger an **unpredictable domino effect** across the entire system
- Port 
-

The **port is the blueprint** (literally just an `interface` or an abstract class in code) that says, _"Hey, if you want to work with me, you must provide these exact functions."_ It contains no actual working code.

The **adapter does the actual thing**. It implements that blueprint and writes the specific code to talk to PostgreSQL, MongoDB, Stripe, or a third-party SMS API.


```python
from abc import ABC, abstractmethod

# ==========================================
# 1. THE PORT (The Blueprint)
# ==========================================
class UserRepositoryPort(ABC):
    """
    The application core only knows about this blueprint.
    It doesn't care how or where the data is stored.
    """
    @abstractmethod
    def save_user(self, user_id: str, name: str) -> None:
        pass


# ==========================================
# 2. THE ADAPTERS (The Real Workers)
# ==========================================
class SQLDatabaseAdapter(UserRepositoryPort):
    """This adapter makes the blueprint work with a SQL Database."""
    def save_user(self, user_id: str, name: str) -> None:
        print(f"Executing: INSERT INTO users VALUES ({user_id}, {name})")


class InMemoryDatabaseAdapter(UserRepositoryPort):
    """This adapter makes the blueprint work with a fast RAM/mock dictionary."""
    def __init__(self):
        self.db = {}

    def save_user(self, user_id: str, name: str) -> None:
        self.db[user_id] = name
        print(f"Saved to memory: {self.db}")


# ==========================================
# 3. THE CORE APPLICATION LOGIC
# ==========================================
class RegisterUserUseCase:
    """
    The Core Logic only interacts with the Port (blueprint).
    It is completely decoupled from the real database tech.
    """
    def __init__(self, user_repo: UserRepositoryPort):
        self.user_repo = user_repo  # Injecting the port

    def execute(self, user_id: str, name: str):
        # Business logic goes here (e.g., validate name length)
        print(f"Registering user: {name}")
        
        # Call the port blueprint
        self.user_repo.save_user(user_id, name)


# ==========================================
# 4. RUNNING THE APPLICATION
# ==========================================
# Production setup: Plug in the real SQL adapter
prod_repo = SQLDatabaseAdapter()
app_prod = RegisterUserUseCase(user_repo=prod_repo)
app_prod.execute("1", "Alice")

# Testing setup: Instantly swap to the memory adapter without touching the core!
test_repo = InMemoryDatabaseAdapter()
app_test = RegisterUserUseCase(user_repo=test_repo)
app_test.execute("2", "Bob")

```

### full example  

```python
from abc import ABC, abstractmethod
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

# =====================================================================
# 1. THE OUTBOUND LAYER (Data leaving the app)
# =====================================================================

# Outbound Port (Blueprint)
class UserRepositoryPort(ABC):
    @abstractmethod
    def save_user(self, user_id: str, name: str) -> None:
        pass

# Outbound Adapter (Real Worker)
class SQLDatabaseAdapter(UserRepositoryPort):
    def save_user(self, user_id: str, name: str) -> None:
        # Pretend this writes to a real Postgres database
        print(f"[DB] Successfully inserted {name} with ID {user_id} into SQL.")


# =====================================================================
# 2. THE APPLICATION CORE (Pure business logic - no web, no database)
# =====================================================================

# Inbound Port (The Core Service Blueprint / Entrypoint)
class RegisterUserUseCase:
    def __init__(self, user_repo: UserRepositoryPort):
        # We inject the outbound port here
        self.user_repo = user_repo  

    def execute(self, user_id: str, name: str) -> dict:
        # Core Business Rule Validation
        if len(name) < 2:
            raise ValueError("Name is too short!")
            
        print(f"[Core] Business rules passed for user: {name}.")
        
        # Trigger the Outbound Port
        self.user_repo.save_user(user_id, name)
        
        return {"status": "success", "user_id": user_id}


# =====================================================================
# 3. THE INBOUND ADAPTER (Web traffic entering the app)
# =====================================================================
# This adapter catches HTTP requests, extracts the JSON, and drives the Core.

app = FastAPI()

# Data structure expected by the API
class UserRequest(BaseModel):
    user_id: str
    name: str

# COMPOSITION ROOT: Wire the real infrastructure together at startup
database_adapter = SQLDatabaseAdapter()
user_service = RegisterUserUseCase(user_repo=database_adapter)

@app.post("/users")
def handle_create_user(request: UserRequest):
    """
    This HTTP controller is the Inbound Adapter.
    It knows about HTTP codes and JSON, but it has ZERO database logic.
    """
    try:
        # The Inbound Adapter drives the Core application
        result = user_service.execute(user_id=request.user_id, name=request.name)
        return result
    except ValueError as e:
        # Translate core logic business errors into HTTP Web errors
        raise HTTPException(status_code=400, detail=str(e))

```