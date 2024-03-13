# Wheelzy

## 1) Database Schema

### Car

- `CarID` (Primary Key, INT)
- `Year` (INT)
- `Make` (VARCHAR)
- `Model` (VARCHAR)
- `SubModel` (VARCHAR)
- `ZipCode` (VARCHAR)

### Buyer

- `BuyerID` (Primary Key, INT)
- `Name` (VARCHAR)
- `Quote` (DECIMAL)
- `IsCurrent` (BOOLEAN)

### CarBuyer

- `CarID` (INT, Foreign Key)
- `BuyerID` (INT, Foreign Key)
- Compound Primary Key (CarID, BuyerID)

### Status

- `StatusID` (Primary Key, INT)
- `Name` (VARCHAR)
- `IsRequiredDate` (BOOLEAN, default false)

### CarStatusHistory

- `CarStatusHistoryID` (Primary Key, INT)
- `CarID` (INT, Foreign Key)
- `StatusID` (INT, Foreign Key)
- `StatusDate` (DATETIME, nullable)
- `ChangedBy` (VARCHAR)
- `ChangeDate` (DATETIME)

  Sql Query:
```sql

 SELECT 
    c.CarID,
    c.Year,
    c.Make,
    c.Model,
    c.SubModel,
    c.ZipCode,
    b.Name AS BuyerName,
    b.Quote,
    s.Name AS StatusName,
    csh.StatusDate
FROM Car c
JOIN CarBuyer cb ON c.CarID = cb.CarID
JOIN Buyer b ON cb.BuyerID = b.BuyerID AND b.IsCurrent = 1
LEFT JOIN CarStatusHistory csh ON c.CarID = csh.CarID
LEFT JOIN Status s ON csh.StatusID = s.StatusID
WHERE csh.StatusDate = (
    SELECT MAX(StatusDate)
    FROM CarStatusHistory
    WHERE CarID = c.CarID
)
```
First I take the information that I need in the Select, when doing Join with the Buyer table I only take the current ones, I also include the status history of each car.
The subquery filters by the most recent status date for each car, making sure that only the most recent status of each car is included.


Entity Framework Query:

```csharp
var result = context.Car .Select(car => new { car.Year, car.Make, car.Model, car.SubModel, CurrentBuyer = car.CarBuyer .Where(cb => (bool)cb.Buyer.IsCurrent) .Select(cb => new { cb.Buyer.Name, cb.Buyer.Quote }) .FirstOrDefault(), CurrentStatus = car.CarStatusHistory .OrderByDescending(csh => csh.ChangeDate) .Select(csh => new { csh.Status.Name, csh.StatusDate }) .FirstOrDefault() }).ToListAsync(); 
```

I take the information I need in the select, when doing join with the Buyer table I only take the current ones, I also include the status history of each car.
I order the state history of the car by ChangeDate in descending order so that the most recent state is first.

Depending on the context it may be better to use one method or another (put the query in a Stored Procedure or do it through Entity Framework).
With Entity it can be easier to develop, and easier to maintain or update.
With a Stored Procedure we can have better performance, or call it from different applications without repeating code.


## 2) Caching Strategy

I would implement a cache system, injecting the IMemoryCache service to the service or controller where it is needed. This way we would not make unnecessary calls to the database, which would ruin the performance of the application.

### 3) UpdateCustomersBalanceByInvoices

If we assume that each invoice belongs to a different customer:

```csharp
public void UpdateCustomersBalanceByInvoices(List<Invoice> invoices)
{
    foreach (var invoice in invoices)
    {
        var customer = dbContext.Customers.Find(invoice.CustomerId);
        if (customer != null)
        {
            customer.Balance -= invoice.Total;
        }
    }
    dbContext.SaveChanges();
}
```
If there are many invoices for the same customer:

```csharp
public void UpdateCustomersBalanceByInvoices(List<Invoice> invoices)
{
    var customerIds = invoices.Select(i => i.CustomerId).Distinct().ToList();
    var customersToUpdate = dbContext.Customers.Where(c => customerIds.Contains(c.CustomerId)).ToList();

    foreach (var invoice in invoices)
    {
        var customer = customersToUpdate.FirstOrDefault(c => c.CustomerId == invoice.CustomerId);
        if (customer != null)
        {
            customer.Balance -= invoice.Total;
        }
    }
    dbContext.SaveChanges();
}
```
-	Return all customers before entering the loop. 
-	Update the invoices and save the database at the end so it does not call for each invoice.


### 4) GetOrders Method

If we assume that each invoice belongs to a different customer:

```csharp
        public async Task<List<OrderDTO>> GetOrders(DateTime dateFrom, DateTime dateTo, List<int> customerIds, List<int> statusIds, bool? isActive)
        {
            var query = dbContext.Orders
                .Where(o => o.OrderDate >= dateFrom && o.OrderDate <= dateTo);

            if (customerIds != null && customerIds.Count > 0)
            {
                query = query.Where(o => customerIds.Contains(o.CustomerId));
            }

            if (statusIds != null && statusIds.Count > 0)
            {
                query = query.Where(o => statusIds.Contains(o.StatusId));
            }

            if (isActive.HasValue)
            {
                query = query.Where(o => o.IsActive == isActive.Value);
            }
            var orders = await query
                .Select(o => new OrderDTO
                {
                    OrderId = o.OrderId,
                    CustomerId = o.CustomerId,
                    StatusId = o.StatusId,
                    IsActive = o.IsActive
                })
                .ToListAsync();

            return orders;
        }

```
This does not cause performance problems because the query is only sent to the database when ToListAsync() is called.

### 5) Workflow

- Identify the project where the Bug is happening
- Download the latest version of the branch where the problem is located.
- Reproduce the bug locally
- Identify and fix the problem in the code
- Modify or create new unit tests
- Create the branch and upload the changes
- Verify that the pipeline has run successfully
- Create the Pull Request
- Verify that the merge has been done successfully to the corresponding branch
- Reproduce the bug in the appropriate environment to make sure it has been resolved
- Notify Bill to check again

