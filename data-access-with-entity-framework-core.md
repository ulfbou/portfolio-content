# Data Access With Entity Framework Core

## Understanding Entity Framework Core

Entity Framework Core is a modern object-relational mapping (ORM) framework for .NET. It simplifies data access by allowing developers to work with databases using .NET objects and LINQ queries. In this guide, we'll explore how to use Entity Framework Core to interact with databases in .NET applications.

## Basic Steps to Use EF Core

1. **Install EF Core**:
   - Install the necessary NuGet packages for your database provider (e.g., Microsoft.EntityFrameworkCore.SqlServer).
2. **Define DbContext**:
   - Create a DbContext class that inherits from DbContext. This class represents the connection to your database and defines your entity sets.
	 ```csharp
	 public class MyDbContext : DbContext
	 {
	 	public MyDbContext(DbContextOptions<MyDbContext> options)
 		 	: base(options)
		{
		}

		public DbSet<Product> Products { get; set; }
	 }
	 ```
3. **Configure DbContext**:
	- Register your DbContext in your application's dependency injection container.
		```csharp
		services.AddDbContext<MyDbContext>(options =>
			options.UseSqlServer(connectionString));
		```
4. **Perform CRUD Operations**:
	- **Create**:
		```csharp
		Product product = new Product { Name = "Product 1", Price = 19.99 };
		context.Products.Add(product);
		await context.SaveChangesAsync();
		```
	- **Read**:
		```csharp
		var products = await context.Products.ToListAsync();
		```
	- **Update**:
		```csharp
		var product = await context.Products.FindAsync(productId);
		product.Price = 29.99;
		await context.SaveChangesAsync();
		```
	- **Delete**:
		```csharp
		context.Products.Remove(product);
		await context.SaveChangesAsync();
		```

## Advanced Features of EF Core

- **LINQ Queries**: Use LINQ to query your database:
	```csharp
	var expensiveProducts = context.Products.Where(p => p.Price > 100);
	```
- **Raw SQL Queries**: Execute raw SQL queries for complex scenarios:
	```csharp
	var products = context.Products.FromSqlRaw("SELECT * FROM Products WHERE Price > 100");
	```
- **Migrations**: Manage database schema changes using migrations.
- **Asynchronous Operations**: Perform database operations asynchronously to improve performance.

## Best Practices for Using EF Core

- Use Asynchronous Operations: Leverage asynchronous methods to avoid blocking the main thread.
- Optimize Queries: Write efficient LINQ queries and use appropriate indexing.
- Consider Lazy Loading and Eager Loading: Choose the appropriate loading strategy based on your needs.
- Handle Exceptions Gracefully: Implement proper error handling and logging.
- Use Transactions: Encapsulate multiple database operations within a transaction to ensure data consistency.

By following these guidelines and utilizing the powerful features of EF Core, you can efficiently and effectively interact with your database in C# applications.
