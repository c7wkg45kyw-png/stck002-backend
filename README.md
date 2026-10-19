# STCK002 Stock API

.NET 10 REST API for the product catalogue, persistent session cart, and atomic checkout. It is the only application that accesses PostgreSQL and owns physical stock deduction.

## Configure and run

```bash
cp appsettings.Development.example.json appsettings.Development.json
dotnet restore
dotnet run
```

Set a real `ConnectionStrings:Postgres` value in `appsettings.Development.json` or through .NET user secrets. Set `Frontend:Origins` to the exact Next.js origins in use (for example `http://localhost:3001` when port 3000 is occupied). The API applies EF Core migrations and inserts deterministic mock products only when the catalogue is empty.

Swagger UI is available at `http://localhost:<port>/swagger`; its OpenAPI document is at `/swagger/v1/swagger.json`. Use the port printed by `dotnet run`.

## Structure

| Area | Purpose |
| --- | --- |
| `Domain` | Product, inventory, cart, and order entities with local invariants |
| `Domain/README.md` | รายละเอียด entity, invariant และเหตุผลการแยก Domain layer |
| `Application/Contracts` | API request/response records |
| `Application/Errors` | Business-rule error types such as `BusinessRuleException` |
| `Application/Interfaces` | Persistence and transaction ports used by services |
| `Application/Services` | Catalogue/cart use cases and checkout orchestration |
| `Application/README.md` | รายละเอียด use case, contracts, errors, interfaces และเหตุผลการแยก Application layer |
| `Controllers` | HTTP routes, session-cookie lookup, and error responses |
| `Controllers/README.md` | รายละเอียดหน้าที่และเหตุผลของ Controller, HTTP error และ session cookie |
| `Infrastructure` | EF Core mapping, repository, transaction adapter, seed initializer, migrations, and observability |
| `Infrastructure/README.md` | รายละเอียดหน้าที่และเหตุผลเชิงโครงสร้างของไฟล์ใน Infrastructure |
| `Infrastructure/Migrations` | EF Core schema history for PostgreSQL |

## ทำไม C# จึงเขียนและแยกโครงสร้างแบบนี้

โค้ดนี้ใช้แนวคิด **Clean Architecture แบบเรียบง่าย** ร่วมกับการแยก Command/Query ตาม use case (CQRS แบบ pragmatic) จุดประสงค์ไม่ใช่เพิ่มจำนวนไฟล์ แต่เพื่อให้กฎสำคัญอย่าง “ห้ามขายเกินสต็อก” อยู่ในจุดที่ทดสอบและแก้ไขได้โดยไม่ผูกกับ HTTP หรือ PostgreSQL โดยตรง

```text
HTTP Request
  → Controller
  → Application Service
  → Application Interface
  → Infrastructure Repository / PostgreSQL
             ↕
          Domain Entity
```

### หน้าที่ของแต่ละ layer

| Layer | เขียนอะไรไว้ที่นี่ | เหตุผล |
| --- | --- | --- |
| `Controllers` | Route, อ่าน session cookie, แปลง HTTP request/response และ error status | Controller จึงสั้น และไม่ใส่ SQL หรือกฎ checkout ที่ซับซ้อน |
| `Application/Services` | ลำดับขั้นตอนของ use case เช่น เพิ่มสินค้าใน cart หรือ checkout | อ่าน flow ธุรกิจได้จากไฟล์เดียว และใช้ fake repository ทดสอบได้ |
| `Application/Interfaces` | สัญญา `IStoreRepository` และ `IStoreTransaction` | Application รู้เพียงว่าต้องอ่าน stock สำหรับ checkout หรือ save; ไม่รู้ว่าใช้ EF Core/Npgsql อย่างไร |
| `Domain` | Entity และกฎที่ติดกับข้อมูล เช่น `InventoryStock.Deduct()` | ป้องกันไม่ให้ชั้นอื่นแก้ `Quantity` โดยตรงจน stock ติดลบ |
| `Infrastructure` | `DbContext`, EF Core mapping, LINQ query, serializable transaction adapter, migrations | รายละเอียดเทคนิคถูกเก็บไว้ที่ขอบระบบ เปลี่ยน database/ORM ได้ง่ายกว่า |

ตัวอย่าง dependency inversion: `CartService` รับ interface ไม่รับ `StockDb` โดยตรง

```csharp
public sealed class CartService(IStoreRepository storeRepository)
{
    // ใช้ storeRepository ผ่านสัญญาเท่านั้น
}
```

รูปแบบนี้เรียกว่า **primary constructor** ของ C# สมัยใหม่: dependency ถูกประกาศที่หัว class, เป็น readonly โดยธรรมชาติ และไม่ต้องเขียน constructor ที่มีเพียงการ assign field ซ้ำ ๆ ส่วน `Program.cs` เป็นจุดเดียวที่ผูก interface เข้ากับ implementation จริงด้วย Dependency Injection:

```csharp
services.AddScoped<IStoreRepository, StoreRepository>();
```

ผลคือ unit test สามารถส่ง fake repository เข้า `CartService` ได้ โดยไม่ต้องเปิด PostgreSQL จริง

### เหตุผลที่แยก `record` สำหรับ request/response

ไฟล์ใน `Application/Contracts` ใช้ `record` เช่น:

```csharp
public sealed record AddCartItemRequest(Guid ProductId, int Quantity);
```

`record` เหมาะกับข้อมูลที่ใช้รับส่งระหว่าง API และ Application เพราะสั้น มี equality ตามค่า และไม่มีพฤติกรรมแก้ state ซับซ้อน ต่างจาก `class` ใน `Domain` ที่เก็บ state และกฎธุรกิจ เช่น `InventoryStock` ต้องเป็นผู้ตัดสต็อกผ่าน `Deduct()` เท่านั้น

### เหตุผลที่ใช้ `async`, `Task` และ `CancellationToken`

การอ่าน/เขียน PostgreSQL เป็น I/O ที่ต้องรอ จึงใช้ `async`/`await` เพื่อไม่ให้ thread ของ API ถูกบล็อกขณะรอฐานข้อมูล และทุก public operation รับ `CancellationToken` เพื่อยกเลิกงานที่ client ยกเลิกไปแล้วได้อย่างเหมาะสม

```csharp
public async Task<CartResponse> AddItemAsync(
    string sessionKey,
    AddCartItemRequest request,
    CancellationToken cancellationToken)
{
    var product = await storeRepository.GetProductAsync(
        request.ProductId,
        cancellationToken);
}
```

### เหตุผลที่ checkout อยู่ใน `CartService`

Checkout เป็น business transaction เดียว ไม่ควรกระจายอยู่ใน Controller หรือ Repository:

```text
begin transaction
→ อ่าน cart
→ อ่าน stock สำหรับ checkout ด้วย LINQ ใน serializable transaction
→ ตรวจ stock อีกครั้ง
→ สร้าง Order/OrderItems
→ ตัด stock
→ ล้าง cart
→ commit
```

`CartService.CheckoutAsync` เป็นผู้บอกว่าแต่ละขั้นเกิดตามลำดับใด ส่วน `StoreRepository` ใช้ LINQ อ่าน stock สำหรับ checkout และ `EfStoreTransaction` เป็นผู้ห่อ EF Core transaction แบบ `Serializable` จึงแยก “กฎธุรกิจ” ออกจาก “วิธีคุยกับฐานข้อมูล” ได้ชัดเจน

### เหตุผลที่ entity มี `private set` และ method เฉพาะทาง

ตัวอย่าง `InventoryStock` ไม่เปิด setter ของ `Quantity`:

```csharp
public int Quantity { get; private set; }

public void Deduct(int quantity)
{
    if (!CanFulfil(quantity))
    {
        throw new InvalidOperationException("Stock cannot fulfil the requested quantity.");
    }

    Quantity -= quantity;
}
```

จึงไม่มีโค้ดภายนอกเขียน `stock.Quantity = -1` ได้โดยบังเอิญ และ EF Core ยังสร้าง entity ได้ เพราะมี private parameterless constructor สำหรับ ORM

### เหตุผลที่ Controller จัดการ HTTP error แยกจาก business rule

`BusinessRuleException` ถูกรวมไว้ที่ `Application/Errors` เพราะเป็นความผิดพลาดของกฎธุรกิจ และ factory methods ของ error ทุกกรณี เช่น `InvalidQuantity()` และ `InsufficientStock()` อยู่ในไฟล์เดียวกัน ส่วน `ApiErrorResponse` อยู่ที่ `Infrastructure/Errors` เพราะเป็น JSON contract สำหรับ HTTP API โดย `CartService` เรียก `BusinessRuleException.InsufficientStock()` แล้ว Controller แปลงเป็น `ApiErrorResponse` เพียงครั้งเดียว:

For a repository or PostgreSQL failure, Controllers return `ApiErrorResponse.RepositoryError()` with `REPOSITORY500` and a generic message, without exposing connection or SQL details.

```json
{
  "error_code": "STOCK409",
  "message": "One or more products no longer have sufficient stock.",
  "trace_id": "..."
}
```

ทำให้ Application ไม่ต้องรู้รูปแบบ JSON ขณะที่ Frontend ได้ error contract ที่คงที่

### สิ่งที่ควรรักษาเมื่อเพิ่ม feature ใหม่

1. เพิ่ม request/response `record` ใน `Application/Contracts` หากมีข้อมูลรับส่งใหม่
2. สร้างหรือขยาย service ตาม use case ไม่ใส่ business rule ใน Controller
3. เพิ่ม interface ใน `Application/Interfaces` เมื่อต้องพึ่ง persistence ใหม่
4. ให้ `Infrastructure` implement interface และเพิ่ม EF migration หาก schema เปลี่ยน
5. เพิ่ม test สำหรับ success และ failure path ของ use case นั้น

แนวทางนี้เหมาะกับระบบปัจจุบันที่มีขนาดเล็กถึงกลาง หากในอนาคต use case มากขึ้น สามารถแยก `Application/Services` เป็น feature folders และใช้ MediatR handler ได้ โดยไม่ต้องย้ายกฎใน Domain หรือ Infrastructure ทั้งหมด

## Checkout safety

`CartService.CheckoutAsync` begins a serializable transaction, loads the session cart and checkout stocks through LINQ, revalidates every quantity, creates immutable order snapshots, deducts stock, clears the cart, and commits. Any conflict or failure rolls back the transaction.

## Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/products` | Product catalogue and current stock |
| `GET` | `/api/cart` | Current browser session cart |
| `POST` | `/api/cart/items` | Add quantity for product |
| `PUT` | `/api/cart/items/{itemId}` | Set cart-item quantity; zero removes item |
| `DELETE` | `/api/cart/items/{itemId}` | Remove an item |
| `DELETE` | `/api/cart/items` | Clear the current cart |
| `POST` | `/api/cart/checkout` | Atomic checkout |

## Verify

```bash
dotnet build
# Outputs SQL only; it does not create or update a database.
dotnet ef migrations script --output migrations.sql

# Creates the configured database when it does not exist, then applies migrations.
dotnet ef database update
```

`dotnet ef database update` reads `ConnectionStrings:Postgres` from `appsettings.json`, an optional environment-specific appsettings file, or environment variables. The same value is also used by the running API. PostgreSQL must be available at the configured connection string.
# stck002-backend
