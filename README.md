# Phase 1 - Coding practice improvements.

### Area of improvement
 - Naming convention
 - Segegration of Concern
 - Nesting
 - EF-core Query Optimization (Multiple queries are bad 💀💀)
 - Prebuilds


### Naming Conventions
 - Always use contextual naming convention 
  #### Bad Practice
  ```cs
   var usr = new User();
   var nuser = _userMangementService.Create(usr);
  ```
  ### Good Practice
  ```cs
   var user = new User();
   var createdUser = _userMangementService.Create(user);
  ```

## Segregation of Concerns (Reusability)

Segregation of concerns means splitting different tasks in our code into separate groups. Each group or class then focuses on doing just one thing. This makes our code easier to understand, maintain, and reuse.

### Problem
Sometimes, we write code where one class does too many things. For example, imagine we have a class called UserService. It's supposed to help with managing users. But sometimes, it also does things like fetching user details from the database. This makes it hard to reuse this code in other parts of our program.

### Solution
To solve this, we can split our code into smaller parts. Each part should focus on just one job. For example, instead of having the UserService do everything, we can create a separate class just for getting user details from the database.
Example:

### Bad Practice
In the bad example, our UserService class not only creates and updates users but also fetches user details from the database. This makes it hard to use the user retrieval part in other places without also bringing in user creation and update logic.

```csharp
public class UserService
{
    public User CreateUser(string username)
    {
        // Logic to create a user in the database
    }

    public void UpdateUser(User user)
    {
        // Logic to update a user in the database
    }

    public User GetUser(string username)
    {
        // Logic to retrieve user details from the database
    }
}
```

### Good Practice
In the improved version, we create a new class called UserRetrievalService just for fetching user details from the database. Now, our BookingService can use this new class without worrying about user creation and update logic. This makes our code more organized and easier to work with

```csharp
public class UserService
{
    public User CreateUser(string username)
    {
        // Logic to create a user in the database
    }

    public void UpdateUser(User user)
    {
        // Logic to update a user in the database
    }
}

public class UserRetrievalService
{
    private readonly UserRepository _userRepository;

    public UserRetrievalService(UserRepository userRepository)
    {
        _userRepository = userRepository;
    }

    public User GetUser(string username)
    {
        return _userRepository.GetUser(username);
    }
}

```
## Nesting
Nesting conditions in programming involve placing one conditional statement inside another. While sometimes necessary, excessive nesting can lead to code that is difficult to read, understand, and maintain.

### Challenges of Nesting Conditions
- Readability: Excessive nesting makes code harder to read and comprehend, especially with multiple levels of nesting.

- Maintainability: Code with deep nesting is harder to maintain and modify, as changes in one nested block may impact others.

- Debugging: Identifying and fixing bugs in deeply nested code can be challenging, as it requires tracing through multiple layers of nested blocks.

```csharp
public class UserAuthenticationSystem
{
    public bool AuthenticateUser(string username, string password)
    {
        // Check if user exists
        if (UserExists(username))
        {
            // Check if user's account is active
            if (IsAccountActive(username))
            {
                // Check if password is correct
                if (IsPasswordCorrect(username, password))
                {
                    // Check if two-factor authentication is enabled
                    if (IsTwoFactorEnabled(username))
                    {
                        // Perform two-factor authentication
                        if (PerformTwoFactorAuthentication(username))
                        {
                            return true; // User successfully authenticated
                        }
                    }
                    else
                    {
                        return true; // User successfully authenticated
                    }
                }
            }
        }

        return false; // User authentication failed
    }
}
```


### Solutions:

- Flattening Nesting:
    Flatten nested blocks by refactoring them into smaller, more manageable pieces.

- Guard Clauses:
    Replace deeply nested constructs with guard clauses where possible to handle exceptional cases early.

- Early Return:
    Use early return statements to exit a method or function early if a condition is met.

```csharp
public class UserAuthenticationSystem
{
    public bool AuthenticateUser(string username, string password)
    {
        if (!UserExists(username) || !IsAccountActive(username))
        {
            return false; // User authentication failed
        }

        if (!IsPasswordCorrect(username, password))
        {
            return false; // User authentication failed
        }

        if (IsTwoFactorEnabled(username) && !PerformTwoFactorAuthentication(username))
        {
            return false; // User authentication failed
        }

        return true; // User successfully authenticated
    }
}

```


## EF-core Query Optimization
-  #### Query Only Required Data

```csharp
// Bad Practice
var users = dbContext.Users.ToList(); // Fetching all users with all properties

// Good Practice
var users = dbContext.Users.Select(u => new { u.Id, u.Name }).ToList(); // Fetching only user id and name
```

- #### Avoid Querying Inside Loop Statements (Reduce Multiple Queries)

```csharp
// Bad Practice
var users = dbContext.Users.ToList(); 

foreach(var user in users) 
{
// below code every time fire query into database.
 var order = dbContext.Orders.FirstOrDefault(x => x.UserId == user.Id);
 var orderItems = dbContext.OrderItems.Where(x=> x.OrderId == order.Id).ToList(); 
}

// Good Practice
var query = (from user in dbContext.Users 
join order in dbContext.Orders on user.Id equals order.UserId
join orderItem in dbContext.OrderItems on order.Id equals orderItems.orderId select new
{
 UserId = user.Id,
 OrderId = order.Id   
}).ToList(); 
```
### Knowledge

- We cannot join List and Queryable directly in EF Core due to the difference in their underlying data structures and query execution mechanisms.
- Only queries written in EF Core's query syntax (LINQ to Entities) will be translated to SQL queries and executed on the database server. Operations performed on collections in memory (e.g., List) are executed locally.
- It's recommended to apply filtering, sorting, and projection (selecting specific columns) as early as possible in the query chain to minimize the amount of data retrieved from the database.
- Queryable represents a query that has not yet been executed. The actual query execution happens when terminal operators such as First(), ToList(), FirstOrDefault(), Count(), etc., are applied to the Queryable. This deferred execution allows for query optimization and better performance.



### Prebuilds
- #### Pagination Helper
    
```csharp
public static class PaginationHelper
    {
        public static PagedResultDto<T> Paginate<T>(IQueryable<T> query, PagedAndSortedResultRequestDto filter)
        {
            var count = query.Count();
            query = !filter.Sorting.IsNullOrEmpty()
                ? query.OrderBy(filter.Sorting).Skip(filter.SkipCount).Take(filter.MaxResultCount)
                : query.Skip(filter.SkipCount).Take(filter.MaxResultCount);
            return new PagedResultDto<T>(count, items: query.ToList());
        }
    }
// usages
var paginated = PaginationHelper.Paginate(Query, filter);
```

- #### HttpResponseExtension Helper
```csharp
public static class HttpResponseExtension
{
    public static ResponseDto<T> SendSuccess<T>(this ApplicationService applicationService, T data,
        string message = "Success", HttpStatusCode code = HttpStatusCode.OK)
    {
        return new ResponseDto<T>
        {
            Success = true,
            Code = (int)code,
            Message = message,
            Data = data
        };
    }
}
// usages
this.SendSuccess("Success");
```


- #### Json Extensions
```csharp
public static class JsonExtensions
{ 
    public static string ToJson<T>(this T t) => JsonSerializer.Serialize(t);
}

// usages
var json = obj.ToJson();
```
