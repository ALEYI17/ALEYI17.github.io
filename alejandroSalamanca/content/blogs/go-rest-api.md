---
title: "Part 1: Go REST API with gin"
date: 2025-01-23T09:50:05-05:00
draft: false
image: https://miro.medium.com/v2/resize:fit:640/1*3Huif4E43KHDro3vv7szTQ.png
author: "Alejandro Salamanca"
github_link: "https://gitlab.com/devopsdemo2374651/CurrencyConversion"
tags:
  - GO
  - REST API
  - gin
---
[GitLab Repository](https://gitlab.com/devopsdemo2374651/CurrencyConversion)
#### Building a Cloud-Native Application: My Journey with Go, GitLab CI, Helm, and CNPG  



#### Introduction  
This project is an application I created to improve my skills in building cloud-native applications. The idea behind this project was to explore key concepts that are essential in today’s software development landscape: developing a REST API in Go, setting up a basic CI/CD pipeline using GitLab, creating Helm charts for Kubernetes deployments, and learning how to self-manage a PostgreSQL database in Kubernetes using the CNPG operator.

I started this project with the goal of honing my skills in Go, a language I’ve been eager to master for its speed, simplicity, and efficiency in building modern applications. Alongside that, I wanted to dive into GitLab CI to understand how pipelines streamline the development process, ensuring automated testing, building, and deployment. Learning the basics of Helm was a natural extension of this journey, as deploying applications in Kubernetes has become an industry standard. Finally, managing a PostgreSQL database in Kubernetes using CNPG (Cloud Native PostgreSQL) taught me the fundamentals of operating databases in a cloud-native environment.

In this blog series, I’ll take you through each step of the process, showing how the project came together and sharing the lessons I learned along the way. I hope this inspires others to take on similar challenges to grow in their professional careers and expand their skill sets in building cloud-native applications.

---

#### Structure of the Blog Series  

This blog will be divided into four parts, each focusing on a critical aspect of the project:  

**Part 1: Building the REST API with Go**  
   In the first part, I’ll cover how I built a REST API using the Go programming language and the Gin framework. This section will include the project structure, key features, and how the API handles user registration and authentication.  

**Part 2: Setting Up a CI/CD Pipeline with GitLab CI**  
   The second part will focus on implementing a CI/CD pipeline in GitLab. I’ll show how the pipeline was configured to automate tasks such as testing, building, and containerizing the application.  

**Part 3: Deploying with Helm in Kubernetes**  
   In the third part, I’ll walk you through the basics of creating a Helm chart to deploy the application to a Kubernetes cluster. This section will demonstrate how Helm simplifies application deployment and management in a cloud-native environment.  

**Part 4: Managing PostgreSQL with CNPG Operator**  
   In the final part, I’ll discuss how I used the CNPG (Cloud Native PostgreSQL) operator to set up and manage a self-managed PostgreSQL database in Kubernetes. This part will explore the operator’s features, such as high availability, backups, and monitoring, and how it integrated seamlessly with the application.  

---
### Part 1: Building the REST API with Go

This REST API is designed to manage personal expenses, with the source code available at [GitLab Repository](https://gitlab.com/devopsdemo2374651/CurrencyConversion). I chose Go as the programming language because of its simplicity and performance, which make it an excellent choice for building scalable and efficient applications. Additionally, many modern cloud-native applications are built in Go, making it a perfect candidate for learning the language's key features while working on a practical project.  

The API is built following the **Controller-Service-Repository** architectural pattern. This pattern promotes a clear separation of concerns, enhancing code maintainability and scalability. Each layer has a distinct responsibility: controllers handle HTTP requests, services manage business logic, and repositories handle database interactions.  

To organize the project effectively, the package structure follows the **cmd/internal/pkg** pattern, which is a common convention in Go applications. This structure helps keep the code modular, organized, and easy to navigate, aligning with best practices in Go development.  



---
### Libraries Used  

This project makes use of several libraries and packages to implement key functionalities in the REST API. Below is an overview of the primary libraries used:  

**Gin**:  
  Gin is a web framework in Go designed for building REST APIs. It provides a lightweight and efficient way to handle HTTP requests and route them to the appropriate controllers. Its simplicity and performance make it ideal for creating scalable APIs.  

**Golang-jwt**:  
  This library is used for generating and verifying JSON Web Tokens (JWTs), which are essential for authentication and user session management. It ensures secure communication between the client and the API by allowing token-based authentication.  

**Crypto**:  
  The standard `crypto` package in Go is used for hashing and encryption. In this project, it’s utilized for securely hashing passwords before storing them in the database, ensuring user data is protected against potential breaches.  

**Database/sql**:  
  Part of Go’s standard library, `database/sql` provides a powerful interface for interacting with relational databases. It is used to establish connections, execute SQL queries, and manage database transactions.  

**PostgreSQL Driver (`github.com/lib/pq`)**:  
  This driver enables Go to interact with PostgreSQL databases. It works seamlessly with `database/sql` to handle queries and manage the database layer efficiently.  

---
### Code Walkthrough  

#### Entities  

Entities in this REST API represent the core data models that the application manages. In this project, there are two main entities: **User** and **Expenses**. These structures define the data schemas and include methods where necessary to encapsulate functionality related to the entity.  


#### 1. **User**  

The `User` structure models the data of a user in the application. Here's the code for the `User` struct:  

```go
package user

import "golang.org/x/crypto/bcrypt"

type User struct {
	ID        int    `json:"id"`
	Userame   string `json:"username"`
	Email     string `json:"email"`
	Password  string `json:"password"`
	Country   string `json:"country"`
	Phone     string `json:"phone,omitempty"`
	CreatedAt string `json:"created_at"`
}
```

**Methods**:  

The `User` struct includes two methods that handle password hashing and comparison, ensuring secure storage and authentication:  

**`HashPassword()`**:  
   This method uses the `bcrypt` library to hash the user's password. It ensures that passwords are never stored as plain text in the database.  

   ```go
   func (us *User) HashPassword() error {
       hashedPassword, err := bcrypt.GenerateFromPassword([]byte(us.Password), bcrypt.DefaultCost)
       if err != nil {
           return err
       }
       us.Password = string(hashedPassword)
       return nil
   }
   ```  

**`ComparePassword()`**:  
   This method verifies a plain text password by comparing it with the hashed password stored in the database.  

   ```go
   func (us *User) ComparePassword(pass string) error {
       err := bcrypt.CompareHashAndPassword([]byte(us.Password), []byte(pass))
       if err != nil {
           return err
       }
       return nil
   }
   ```  

#### 2. **Expenses**  

The `Expenses` structure models the financial transactions or personal expenses of a user. Here's the code for the `Expenses` struct:  

```go
package expenses

type Expenses struct {
	ID          int     `json:"id"`
	Description string  `json:"description"`
	Amount      float64 `json:"amount"`
	Date        string  `json:"date"`
	User_id     int     `json:"user_id"`
}
```

The `Expenses` struct is intentionally kept simple since it primarily acts as a container for expense data. 

### Database Connection and Repositories  

Managing the database connection and interacting with the database are key components of this REST API. This implementation draws inspiration from [this post by Orlando Monteverde](https://dev.to/orlmonteverde/api-rest-con-go-golang-y-postgresql-m0o). Below is a breakdown of how the database connection and repositories are implemented.


#### Database Connection  

The database connection in this application is handled using Go's standard `database/sql` package and the PostgreSQL driver (`github.com/lib/pq`). The connection logic is split into two files: **`postgres.go`** and **`database.go`**.  


**1. `postgres.go`**  

This file is responsible for establishing the database connection. It retrieves the connection URI from the `DATABASE_URI` environment variable and uses it to create a connection.  

Here is the code:  

```go
package db

import (
	"database/sql"
	"os"

	_ "github.com/lib/pq"
)

func getConnection() (*sql.DB, error) {
	uri := os.Getenv("DATABASE_URI")
	return sql.Open("postgres", uri)
}
```

- **`getConnection()`**:  
  - Fetches the database URI from the environment variable `DATABASE_URI`.  
  - Uses `sql.Open` to establish a connection with the PostgreSQL database.  
  - Returns the `*sql.DB` object, which can then be used for executing queries.  

**Key Points**:  
- Environment variables ensure sensitive information like database URIs are not hardcoded.  
- The use of `sql.Open` provides a clean interface for database interaction.  

---

**2. `database.go`**  

This file manages a singleton database object to ensure a single connection pool is used across the application.  

Here’s the code:  

```go
package db

import (
	"database/sql"
	"log"
	"sync"
)

var (
	data *Data
	once sync.Once
)

type Data struct {
	DB *sql.DB
}

func initDB() {
	db, err := getConnection()

	if err != nil {
		log.Panic(err)
	}

	if err := db.Ping(); err != nil {
		log.Panicf("Failed to ping database: %v", err)
	}

	log.Println("Database connection established successfully!")
	data = &Data{DB: db}
}

func New() *Data {
	once.Do(initDB)
	return data
}

func Close() error {
	if data == nil {
		return nil
	}
	return data.DB.Close()
}
```

**Key Benefits**:  
- Centralized connection management simplifies maintenance.  
- The singleton pattern improves performance and ensures efficient resource utilization.  

#### Repositories

Repositories are an abstraction for data access, making it easier to manage interactions with the database. By defining an interface for the repository, you can swap out implementations (e.g., using a mock for testing or changing the database backend) without modifying the business logic.


#### `ExpensesRepo` Interface  

The `ExpensesRepo` interface defines the contract for managing expenses. It contains methods that handle CRUD operations for expenses, including adding, updating, listing, deleting, and fetching a single expense.  

```go
package expenses

type ExpensesRepo interface {
	AddExpense(expense *Expenses) error
	ListExpense(userid int) ([]Expenses, error)
	DeleteExpense(ID int, user_id int) error
	UpdateExpense(ID int, expense Expenses, user_id int) error
	GetOne(ID int, user_id int) (Expenses, error)
}
```

**Purpose**:  
  - Ensures a clean separation of concerns by defining what functions must be implemented for managing expenses.  
  - Allows the actual implementation to vary without affecting other parts of the codebase.  

#### `ExpensesRepoImpl`: Implementation  

The `ExpensesRepoImpl` struct implements the `ExpensesRepo` interface. It uses the `*Data` struct (from the database layer) to interact with the database.  

**Snippet of Implementation**:  

```go
package db

import (
	"CurrencyConversion/pkg/expenses"
	"errors"
	"log"
	"time"
)

type ExpensesRepoImpl struct {
	Data *Data
}
```

This struct takes in the `*Data` struct (created as a singleton in the database connection code) and uses it to execute SQL queries.


**Example Implementation: `ListExpense`**  

This method retrieves all expenses for a given user ID.  

```go
func (eri *ExpensesRepoImpl) ListExpense(userid int) ([]expenses.Expenses, error) {
	q := `SELECT id, description, amount, date FROM expenses WHERE user_id = $1`

	rows, err := eri.Data.DB.Query(q, userid)

	if err != nil {
		log.Printf("Failed to execute query: %v", err)
		return nil, err
	}

	defer rows.Close() // Always close rows to avoid resource leaks

	var result []expenses.Expenses
	for rows.Next() {
		var expense expenses.Expenses

		err := rows.Scan(&expense.ID, &expense.Description, &expense.Amount, &expense.Date)
		if err != nil {
			log.Printf("Error while reading the rows: %v", err)
			return nil, err
		}

		result = append(result, expense)
	}

	return result, nil
}
```

**Key Practices**:  
  - **Deferred Cleanup**: Always defer the closing of `rows` to avoid memory leaks.  
  - **Error Handling**: Logs and returns errors to ensure issues can be traced and fixed.  
  - **Prepared Queries**: Uses placeholders (`$1`) to prevent SQL injection attacks.  


The rest of the methods in `ExpensesRepoImpl` (e.g., `AddExpense`, `DeleteExpense`, `UpdateExpense`, `GetOne`) follow a similar structure:
- SQL query preparation.  
- Executing the query.  
- Handling errors and returning the result.  

You can find the full implementation in your GitLab repository.

### Services

The service layer handles the **business logic** of the application. It sits between the controllers (or handlers) and the repository layer, orchestrating the application's operations and ensuring the proper use of the data retrieved from or sent to the database.


#### `UserService` Interface  

This interface defines the contract for the user-related business logic.  

```go
package user

type UserService interface {
	ListUsers() ([]User, error)
	GetOneUser(id int, email string) (User, error)
	AddUser(u *User) error
	UpdateUser(u User, id int, email string) error
	DeleteUser(id int, email string) error
	GetByEmailAndPassword(email string, password string) (User, error)
}
```

**Purpose**:  
  - Defines the methods for managing user-related operations.  
  - Separates the business logic from the repository layer, making the system easier to extend and test.  

#### `UserServiceImpl`: Implementation  

The `UserServiceImpl` struct implements the `UserService` interface. It depends on the `UserRepo` interface for interacting with the database.

**Implementation Snippet**:  

```go
package services

import (
	"CurrencyConversion/pkg/user"
	"errors"
	"log"
)

type UserServiceImpl struct {
	UserRepo user.UserRepo // Dependency on the UserRepo interface
}

func (usi *UserServiceImpl) ListUsers() ([]user.User, error) {
	return usi.UserRepo.ListAll()
}
```
**_NOTE:_**  In this application, the service layer is straightforward, as there isn’t much business logic beyond interacting with the repository.

#### Key Observations  

**Dependency Injection**:  
   - The `UserServiceImpl` struct depends on the `UserRepo` interface, not its implementation.  
   - This abstraction enables the use of **mock implementations** for testing.  

**Testing Benefits**:  
   - By injecting a mock `UserRepo`, you can test the service layer independently of the database.  


#### Benefits of This Approach  

**Separation of Concerns**:  
  - The service layer isolates business rules, ensuring that controllers and repositories don’t handle unrelated logic.  

**Flexibility**:  
  - Repositories can be swapped or mocked without modifying the service code.  

**Scalability**:  
  - If new business logic is needed in the future, it can be added to the service layer without affecting the repository or controllers.  


The full implementation can be found in your GitLab repository.


### Controller 

The controller layer is responsible for handling **HTTP requests** and returning appropriate responses. It acts as the bridge between the client's request and the application's services.

#### Why Gin?  

Gin is an excellent choice for building REST APIs because:  
- **Performance**: It is lightweight and highly performant.  
- **Middleware Support**: Facilitates tasks like authentication, logging, and error handling.  
- **JSON Management**: Simplifies JSON binding and responses.  
- **Crash-Free**: It handles panics gracefully, ensuring the application remains stable.  


#### `UserController` Interface  

While it's not mandatory to create an interface for the controller, it can be useful for:  
- Defining the structure and expected methods for consistency.  
- Enabling testing and mocking if needed.  

**Code**:  

```go
package user

import "github.com/gin-gonic/gin"

type UserController interface {
	ListUsers(c *gin.Context)
	GetUser(c *gin.Context)
	AddUser(c *gin.Context)
	UpdateUser(c *gin.Context)
	DeleteUser(c *gin.Context)
}
```

- **Purpose**:  
  - Defines the controller's methods for managing HTTP requests related to users.  
  - Each method maps to a specific API endpoint.  


#### `UserControllerImpl`: Implementation  

This struct implements the `UserController` interface and interacts with the `UserService`.  

**Code**:  

```go
package Controller

import (
	"CurrencyConversion/pkg/user"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
)

type UserControllerImpl struct {
	UserService user.UserService
}

// ListUsers handles GET /users
func (uci *UserControllerImpl) ListUsers(c *gin.Context) {
	users, err := uci.UserService.ListUsers()

	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, users)
}
```

#### Key Observations  

**JSON Handling**:  
   - Gin makes it easy to respond with JSON using `c.JSON(statusCode, data)`.  

**HTTP Status Codes**:  
   - The `net/http` package provides standard HTTP status codes, ensuring consistency.  

**Dependency Injection**:  
   - The `UserControllerImpl` depends on the `UserService` interface, not its implementation.  
   - This abstraction makes the controller testable and decoupled.  

**Error Handling**:  
   - Errors are returned as JSON objects with appropriate HTTP status codes.  

You can find the full implementation in your GitLab repository.

### Middleware for JWT 

The middleware for JWT management is a crucial component of the application, as it ensures that only authenticated users can access certain routes and perform specific actions. JWT (JSON Web Tokens) is a widely adopted standard for creating tokens that encode user identity and privileges securely. In this case, the JWT functionality is implemented within the LoginController, although ideally, it should be placed in a dedicated middleware package for better separation of concerns and reusability.

The `LoginController` struct:

**Code**:  

```go
package Controller

import (
	"CurrencyConversion/internal/services"
	"CurrencyConversion/pkg/Auth"
	"net/http"

	"time"
	"github.com/gin-gonic/gin"
	"github.com/golang-jwt/jwt/v5"
)

type LoginController struct {
  UserService *services.UserServiceImpl
  JwtKey string
}
```
**_NOTE:_** It should be better to use the user service interface, instead of the implementation.

The `Login` method:

**Code**:  

```go
func (lc *LoginController) Login(c *gin.Context){
  var authDto Auth.AuthDTO

  if err:= c.ShouldBindJSON(&authDto); err != nil{
    c.JSON(http.StatusBadRequest,gin.H{"Error":"Invalid Input"})
    return
  }

  user,err := lc.UserService.GetBYEmailAndPassword(authDto.Email,authDto.Password)

  if err != nil {
    c.JSON(http.StatusUnauthorized,gin.H{"error":"Invalid credentials"})
    return
  }

  expiration:= time.Now().Add(2 * time.Hour)

  token:= jwt.NewWithClaims(jwt.SigningMethodHS256,jwt.MapClaims{
    "email":user.Email,
    "exp":time.Now().Add(time.Hour * 1).Unix(),
  })

  tokenString,err := token.SignedString([]byte(lc.JwtKey))
  if err != nil {
    c.JSON(http.StatusInternalServerError, gin.H{"error": "Could not generate token"})
    return
  } 
  
  c.JSON(http.StatusOK, gin.H{
        "token": tokenString,
        "expires_at": expiration.Format(time.RFC3339),
    })
}
```
The Login method is responsible for handling user authentication and generating a JWT token. It starts by binding the request body to an AuthDTO object, which contains the user's email and password. If the input is invalid, it responds with a 400 Bad Request. Then, it calls the UserService to validate the credentials using GetBYEmailAndPassword. If authentication fails, a 401 Unauthorized response is returned. Upon successful authentication, the method creates a token with claims, including the user's email and an expiration time (set to 1 hour). The token is signed using the application's JwtKey. If any errors occur during the signing process, a 500 Internal Server Error is sent. Finally, the method returns the signed token and its expiration time in the response, allowing the client to use this token for subsequent requests.

The `JWTAuthMiddleware` method:

**Code**:  

```go
func (lc *LoginController)JWTAuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        tokenString := c.GetHeader("Authorization")
                // Parse token
        token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok{
              return nil, http.ErrAbortHandler
            }
            return []byte(lc.JwtKey), nil
        })
        if err != nil || !token.Valid {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
            c.Abort()
            return
        }

        // Add claims to context
        if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
          c.Set("claims", claims)
          if email, exists := claims["email"].(string); exists{
            c.Set("email", email)
          } else{
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid token claims"})
            c.Abort()
            return
          }
        } else {
          c.JSON(http.StatusUnauthorized, gin.H{"error": "Unauthorized"})
          c.Abort()
          return
        } 

       
        c.Next()
    }
}
```

The JWTAuthMiddleware method ensures secure access to protected routes by validating JWT tokens. It retrieves the token from the Authorization header of the incoming request. The jwt.Parse function is used to parse and validate the token. If the token is missing, invalid, or uses an unsupported signing method, a 401 Unauthorized response is returned, and further request processing is halted with c.Abort(). When the token is valid, its claims are extracted and added to the Gin context using c.Set. This allows subsequent handlers to access user-specific information, such as the email, for processing. If claims are missing or malformed, the middleware responds with another 401 Unauthorized error. Otherwise, the request is passed to the next handler in the pipeline using c.Next(). This middleware effectively protects routes by ensuring only authenticated users with valid tokens can access them.

### Config

In the application, the configuration, initialization, and startup logic is organized into three key files: `config.go`, `initialize.go`, and `main.go`. These files streamline the process of setting up and running the application while ensuring modularity and reusability.



#### **`config.go`**

**Code**:  

```go
package app

import (
	"log"
	"os"
)

type Config struct {
	PORT        string
	DatabaseUri string
  JwtKey string
}

func LoadConfig() *Config {
	var c Config

	c.PORT = os.Getenv("PORT")
	if c.PORT == "" {
		log.Fatal("PORT is not set")
	}

	c.DatabaseUri = os.Getenv("DATABASE_URI")
	if c.DatabaseUri == "" {
		log.Fatal("DATABASE_URI is not set")
	}

	log.Printf("Loaded configuration: PORT=%s, DATABASE_URI=%s", c.PORT, c.DatabaseUri)

  c.JwtKey = os.Getenv("JWT_KEY")
  if c.JwtKey == ""{
    log.Fatalf("JWT key not set")
  } 
	return &c
}
```

The `config.go` file handles the retrieval and management of environment variables critical to the application's operation. This includes variables such as the `PORT` (on which the server listens), the `JWT_KEY` (used for signing and verifying JWTs), and the `DATABASE_URL` (to connect to the database). By centralizing these configurations, the application can easily adapt to different environments (e.g., development, staging, production) without requiring code changes. Once the environment variables are loaded, they are passed to the rest of the application, ensuring all components have access to the necessary configuration.



#### **`initialize.go`**

**Code**:  

```go
package app

import (
	"CurrencyConversion/internal/Controller"
	"CurrencyConversion/internal/db"
	"CurrencyConversion/internal/services"
	"github.com/gin-gonic/gin"
)

func InitializeApp(config *Config) *gin.Engine {

	data := db.New()
	// Expenses setup
	repo := &db.ExpensesRepoImpl{Data: data}
  usersRepo := &db.UserRepoImpl{Data: data}
	service := &services.ExpensesService{ExpenseRepo: repo, UserRepo: usersRepo}
	controller := &Controller.ExpenseContollerImpl{ExpensiveService: service}

	//Users setup
	usersService := &services.UserServiceImpl{UserRepo: usersRepo}
	usersController := &Controller.UserControllerImpl{UserService: usersService}
  
  //login setup
  loginController := &Controller.LoginController{UserService: usersService,JwtKey: config.JwtKey}
	r := gin.Default()

  r.POST("/login", loginController.Login)
  r.POST("/users/register", usersController.AddUser)
  r.Use(loginController.JWTAuthMiddleware())

	// Users endpoints
	r.GET("/users", usersController.ListUsers)
	r.GET("/user/:id", usersController.GetUser)
	r.PUT("/user/update/:id", usersController.UpdateUser)
	r.DELETE("/user/delete/:id", usersController.Delete)

	//Expenses endpoint
	r.POST("/add", controller.AddExpense)
	r.GET("/list", controller.ListExpenses)
	r.GET("/expense/", controller.GetExpense)
	r.PUT("/update/", controller.UpdateExpense)
	r.DELETE("/delete/", controller.DeleteExpense)

	return r
}
```

The `initialize.go` file serves as the core of the application's setup process. It is responsible for creating the repositories, services, and controllers that the application depends on. These components are instantiated in a modular manner, promoting separation of concerns and making it easier to test individual parts of the system. Once the components are ready, this file defines the routes for the API endpoints, binding specific controllers to their respective HTTP methods and paths. This ensures that each endpoint is properly wired to handle incoming requests with the correct business logic.



#### **`main.go`**

**Code**:  

```go
package main

import (
	"CurrencyConversion/internal/app"
	"log"
)

func main() {

	config := app.LoadConfig()
	if config.PORT == "" {
		log.Fatal("PORT is not set")
	}

	r := app.InitializeApp(config)

	log.Printf("Starting server on port %s...", config.PORT)

  err:= r.Run(":" + config.PORT)
  if err != nil {
    log.Fatalf("Failed to start server =D")
  }
  
}
```

The `main.go` file acts as the entry point of the application. It starts by loading the configuration using the `LoadConfig` function. If the `PORT` variable is missing, the program logs an error and terminates, ensuring the server doesn't start with incomplete configurations. Next, it calls the `InitializeApp` function from `initialize.go` to set up the application, including the routes and middleware. Finally, it starts the server by calling the Gin router's `Run` method, binding it to the specified port. If the server fails to start for any reason, an error message is logged, and the application exits gracefully.

### Final thoughts

I hope you enjoyed this blog and found it helpful! This is just the beginning of a series where I’ll explore more about building REST APIs in Go and beyond. If you have any questions, feedback, or observations, feel free to reach out to me via email—I’d love to hear from you!

This is my first implementation of a REST API in Go, and while it may not be perfect, it’s a step toward writing better, more robust, and high-quality code. I’m excited to continue learning, refining my skills, and tackling more complex projects in the future. Stay tuned for the next part, and thank you for reading!