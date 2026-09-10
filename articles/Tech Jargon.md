- **Stranger Fig** - Basically when you have a shit system and rather than do a full rewrite you basically look at isolated functionality and start replacing that by having behind a feature flag of sorts. come from how fig plant takes over a host planet and slowly takes over and kills it. 
- **Requirements** say what the system should do
- **invariants** say what must _never_ be violated while doing it.  “every order has at least one line item”,  The key word is _always_: an invariant isn’t a temporary condition or a goal; it’s a permanent truth about valid states.
- **unbounded coupling** occurs when the dependency between two or more components has no constraint on how far its impact can cascade. Rather than being contained within an explicit boundary, a single change, failure, or scaling demand in one component can trigger an **unpredictable domino effect** across the entire system
- Port 
-


### Ports and Adapters
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

#### full example  

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



**low ceremony** means ==having very little overhead, boilerplate code, or administrative process required to get something done==. [[1](https://stackoverflow.com/questions/68092498/what-does-low-ceremony-mean), [2](https://www.linkedin.com/posts/david-otano_possible-hot-take-theres-too-much-ceremony-activity-7401317359464136705-bBZ2)]

Code and Frameworks

- **Less Boilerplate:** You write a small amount of code to achieve a goal, without repetitive interfaces, base classes, or configuration files. [[1](https://jasperfx.net/news/wolverine-undisputed-champion-low-ceremony-code), [2](https://stackoverflow.com/questions/68092498/what-does-low-ceremony-mean)]

- **Convention over Configuration:** The system makes smart default choices so you do not have to declare every tiny setting manually. [[1](https://learn.microsoft.com/en-us/archive/msdn-magazine/2009/february/patterns-in-practice-convention-over-configuration)]

- **Example:** Moving from a heavy, multi-layered enterprise framework to a tool like [.NET Minimal APIs](https://www.telerik.com/blogs/low-ceremony-high-value-tour-minimal-apis-dotnet-6), where a basic web server takes just a couple of lines instead of dozens. [[1](https://www.telerik.com/blogs/low-ceremony-high-value-tour-minimal-apis-dotnet-6)]

Process and Workflow

- **Fewer Rules:** Teams skip heavy documentation, rigid gatekeeping, and bureaucratic approvals in favor of fast action.

- **Focus on Value:** The energy goes into solving the actual problem rather than filling out tickets, updating tracking boards, or satisfying pipeline checks


### Facade 

**The 1-line answer:**

> A facade is a class that provides a **static shortcut** to a dynamic service stored inside Laravel’s Service Container.

**The 2-line answer:**

> A facade is a class that provides a **static shortcut** to a dynamic service stored inside Laravel’s Service Container.  
> It acts as a **proxy**, intercepting your static call and running it on a real object instance behind the scenes.

**The 3-line answer:**

> A facade is a class that provides a **static shortcut** to a dynamic service stored inside Laravel’s Service Container.  
> It acts as a **proxy**, intercepting your static call and running it on a real object instance behind the scenes.  
> This gives you the benefit of **clean, memorable syntax** without sacrificing the ability to test or swap out the underlying code.


#### full example

Step 1: The Underlying Class (The "Worker")

This is a standard PHP class with a regular, non-static method.

php

```
namespace App\Services;

class SMSGateway 
{
    // A regular instance method
    public function send(string $message): string 
    {
        return "SMS Sent: {$message}";
    }
}
```

Use code with caution.

Step 2: Register it in the Service Container

Laravel needs to know about this class. We register it in an existing service provider (like `app/Providers/AppServiceProvider.php`) using a unique string key (`'sms'`).

php

```
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Services\SMSGateway;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Bind the string key 'sms' to our SMSGateway class
        $this->app->bind('sms', function () {
            return new SMSGateway();
        });
    }
}
```

Use code with caution.

Step 3: Create the Facade Class

Now, we create the facade. Instead of writing custom logic, we simply extend Laravel's core `Facade` class and implement **`getFacadeAccessor()`**. This tells Laravel which string key to look up in the container.

php

```
namespace App\Facades;

use Illuminate\Support\Facades\Facade;

class SMS extends Facade 
{
    /**
     * Get the registered name of the component.
     */
    protected static function getFacadeAccessor(): string
    {
        return 'sms'; // Matches the container key from Step 2
    }
}
```

Use code with caution.

---

The Result: How you use it

Now, anywhere in your application (like a controller or route file), you can call the method **statically**, even though it was defined as a non-static instance method in Step 1.

php

```
use App\Facades\SMS;

// This looks static, but under the hood Laravel pulls the SMSGateway 
// instance from the container and runs ->send() on it.
$response = SMS::send('Hello World!'); 

echo $response; // Outputs: SMS Sent: Hello World!
```

Use code with caution.

Would you like to see how easy it is to **mock this custom `SMS` facade** in a test so you don't send real text messages during testing?