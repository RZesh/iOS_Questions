# 📱 Senior iOS Developer — Interview Q&A Guide (Part 2)

> Covers: Struct vs Class decisions, Closures deep-dive, GCD vs NSOperation, Protocols, SOLID, Core Data, TCA, VIPER, Authentication Flows, Offline Storage, Navigation, and more.

---

## 1. Struct vs Class — How to Decide

### Q1: How do you decide whether to use a `struct` or a `class` when creating data models?

**Answer:**
Apple's official guideline: **"Use structs by default. Use classes only when you need identity or shared mutable state."**

**Decision flowchart:**
```
Do you need inheritance? → YES → Use class
Do you need identity (===)? → YES → Use class
Do you need shared mutable state? → YES → Use class (or actor)
Do you need Objective-C interop? → YES → Use class (NSObject)
Everything else? → Use struct ✅
```

| Factor | `struct` | `class` |
|---|---|---|
| Type | Value type (copied) | Reference type (shared) |
| Memory | Stack (fast) | Heap (slower) |
| Thread safety | ✅ Inherently safe (each thread gets a copy) | ❌ Needs synchronization |
| ARC overhead | ❌ None | ✅ Reference counting overhead |
| Mutability | `mutating` keyword needed | Mutable by default |
| `deinit` | ❌ Not available | ✅ Available |
| Inheritance | ❌ No | ✅ Yes |
| `Sendable` | ✅ Usually implicit | ❌ Must be explicitly managed |

**Real-World Example 1 — API Response Models (Use Struct):**
```swift
// ✅ STRUCT — Data comes from API, is immutable, no identity needed
struct UserProfile: Codable, Sendable, Hashable {
    let id: String
    let name: String
    let email: String
    let avatarURL: URL?
}

struct OrderItem: Codable, Sendable {
    let productId: String
    let quantity: Int
    let price: Double
}

// These are parsed from JSON, displayed in UI, and discarded
// No two instances need to "be the same object" — value semantics are perfect
let user1 = UserProfile(id: "1", name: "Rajesh", email: "r@e.com", avatarURL: nil)
var user2 = user1  // independent copy
// Modifying user2 does NOT affect user1 ✅
```

**Real-World Example 2 — Shared State Manager (Use Class/Actor):**
```swift
// ✅ CLASS/ACTOR — Shared state, single instance, needs identity
actor BluetoothManager {
    static let shared = BluetoothManager()
    
    private var connectedDevices: [UUID: CBPeripheral] = [:]
    private var isScanning = false
    
    func connect(to device: CBPeripheral) {
        connectedDevices[device.identifier] = device
    }
    
    func disconnect(device: UUID) {
        connectedDevices[device.identifier] = nil
    }
}

// Multiple screens (DeviceList, DeviceDetail, Settings) all need 
// the SAME instance of BluetoothManager — reference semantics required
// Using a struct here would create independent copies = broken state
```

---

### Q2: When would you use a class even for a data model?

**Answer:**
Use a class for data models when:
1. The model has **identity** — two objects with same data are NOT equal (e.g., two `UIViewController` instances)
2. The model is **observed** via `ObservableObject` (pre-iOS 17) or `@Observable`
3. You need **Core Data** (`NSManagedObject` is a class)
4. The model is **very large** and copying is expensive

```swift
// ✅ Class — Observable ViewModel (needs reference semantics for SwiftUI binding)
@Observable
class CartViewModel {
    var items: [CartItem] = []
    var total: Double { items.reduce(0) { $0 + $1.price * Double($1.quantity) } }
    
    func addItem(_ item: CartItem) {
        if let index = items.firstIndex(where: { $0.productId == item.productId }) {
            items[index].quantity += 1
        } else {
            items.append(item)
        }
    }
}

// ✅ Struct — The data inside the ViewModel
struct CartItem: Identifiable, Hashable {
    let id = UUID()
    let productId: String
    let name: String
    let price: Double
    var quantity: Int
}
```

---

## 2. Type Inference vs Type Annotation

### Q3: What is the difference between Type Inference and Type Annotation?

**Answer:**

| Concept | Type Inference | Type Annotation |
|---|---|---|
| Who decides the type? | **Compiler** assumes based on value | **Developer** explicitly declares |
| Syntax | `let x = 42` | `let x: Int = 42` |
| Readability | Less verbose | More explicit |
| When needed | Default for simple cases | Required for ambiguous cases |

```swift
// TYPE INFERENCE — compiler figures out the type
let name = "Rajesh"          // compiler infers String
let age = 30                 // compiler infers Int
let pi = 3.14               // compiler infers Double (NOT Float!)
let items = [1, 2, 3]       // compiler infers [Int]
let isActive = true          // compiler infers Bool

// TYPE ANNOTATION — you explicitly state the type
let price: Float = 9.99     // Without annotation, would be Double
let count: Int64 = 1000000  // Without annotation, would be Int
let callback: (String) -> Void = { print($0) }
var optionalName: String? = nil  // Must annotate — compiler can't infer Optional from nil

// WHEN ANNOTATION IS REQUIRED:
// 1. Empty collections
let emptyArray: [String] = []          // ❌ let emptyArray = [] won't compile

// 2. Protocol types
let service: UserServiceProtocol = MockUserService()

// 3. Disambiguating overloads
let value: CGFloat = 10  // Not Int or Double
```

**Real-World Example 1 — API Response Parsing:**
```swift
// Type inference makes JSON decoding clean:
let user = try JSONDecoder().decode(UserProfile.self, from: data)
// compiler infers: user is UserProfile

// But annotation is needed for protocol-based DI:
let service: NetworkServiceProtocol = URLSessionNetworkService()
```

**Real-World Example 2 — SwiftUI Property Wrappers:**
```swift
@State private var isLoading = false          // inferred as Bool
@State private var selectedTab: TabType = .home  // annotation needed for enum
@State private var items: [Item] = []          // annotation needed for empty array
```

---

## 3. Higher-Order Functions for Performance

### Q4: What are Higher-Order Functions and how do they optimize performance?

**Answer:**
Higher-order functions are functions that **take other functions as parameters** or **return functions**. Swift's collection higher-order functions (`map`, `filter`, `reduce`, `compactMap`, `flatMap`, `sorted`) replace manual loops with optimized, declarative operations.

```swift
struct Product {
    let id: String
    let name: String
    let price: Double
    let category: String
    let isAvailable: Bool
}

let products: [Product] = fetchProducts()

// ❌ MANUAL LOOP — verbose, error-prone
var availableElectronicsNames: [String] = []
for product in products {
    if product.isAvailable && product.category == "Electronics" {
        availableElectronicsNames.append(product.name.uppercased())
    }
}
availableElectronicsNames.sort()

// ✅ HIGHER-ORDER — declarative, chainable, optimized
let result = products
    .filter { $0.isAvailable && $0.category == "Electronics" }
    .map { $0.name.uppercased() }
    .sorted()
```

**Key Higher-Order Functions:**
```swift
// map — transform each element
let prices = products.map { $0.price }  // [Double]

// compactMap — transform + remove nils
let imageURLs = products.compactMap { URL(string: $0.imageURLString) }  // [URL]

// filter — keep elements matching condition
let affordable = products.filter { $0.price < 100 }

// reduce — combine into single value
let totalPrice = products.reduce(0) { $0 + $1.price }

// flatMap — flatten nested arrays
let allTags: [String] = products.flatMap { $0.tags }  // [[String]] → [String]

// sorted — custom ordering
let byPrice = products.sorted { $0.price < $1.price }

// contains — check existence (short-circuits!)
let hasExpensive = products.contains { $0.price > 1000 }

// first(where:) — find first match (short-circuits!)
let firstElectronic = products.first { $0.category == "Electronics" }
```

**Real-World Example 1 — Optimizing a Search Filter:**
```swift
// User types "iph" in search bar — filter + sort + limit results
func searchProducts(query: String, in products: [Product]) -> [Product] {
    products
        .filter { $0.name.localizedCaseInsensitiveContains(query) }
        .sorted { $0.name.count < $1.name.count }  // shorter names first
        .prefix(20)  // limit to 20 results for performance
        .map { $0 }  // convert Slice back to Array
}
```

**Real-World Example 2 — Dashboard Aggregation:**
```swift
struct Order { let amount: Double; let status: String; let date: Date }

func dashboardStats(from orders: [Order]) -> (total: Double, pending: Int, avgOrder: Double) {
    let completed = orders.filter { $0.status == "completed" }
    let total = completed.reduce(0) { $0 + $1.amount }
    let pending = orders.filter { $0.status == "pending" }.count
    let avg = completed.isEmpty ? 0 : total / Double(completed.count)
    return (total, pending, avg)
}
```

---

## 4. weak vs unowned — Why Does unowned Exist?

### Q5: What is the difference between `weak` and `unowned`, and why was `unowned` created?

**Answer:**
Both break retain cycles, but they differ in safety and performance:

| Feature | `weak` | `unowned` |
|---|---|---|
| Optional? | ✅ Must be Optional (`weak var x: T?`) | ❌ Non-optional (`unowned var x: T`) |
| When object is freed | Auto-set to `nil` | **Dangling reference** — crashes if accessed |
| Performance | Slight overhead (nil-checking, zeroing) | **Slightly faster** (no zeroing overhead) |
| Safety | ✅ Safe — always check for nil | ❌ Unsafe — crashes on use-after-free |
| Use when | Referenced object might die first | Referenced object ALWAYS outlives you |

**Why was `unowned` created?**
1. **Performance** — `weak` has overhead from optional wrapping and side-table management. In tight loops with millions of references, `unowned` is measurably faster.
2. **API clarity** — `unowned` documents the relationship: "this object cannot exist without its parent." It's a contract.
3. **Convenience** — Avoids constant `guard let self = self else { return }` unwrapping.

**Real-World Example 1 — Customer ↔ CreditCard (unowned is perfect):**
```swift
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) { self.name = name }
    deinit { print("Customer \(name) deallocated") }
}

class CreditCard {
    let number: String
    unowned let owner: Customer  // Card CANNOT exist without a customer
    
    init(number: String, owner: Customer) {
        self.number = number
        self.owner = owner
    }
    deinit { print("Card \(number) deallocated") }
}

var rajesh: Customer? = Customer(name: "Rajesh")
rajesh?.card = CreditCard(number: "4111", owner: rajesh!)
rajesh = nil  // Both deallocated ✅ — no retain cycle
```

**Real-World Example 2 — ViewModel Closure (weak is safer):**
```swift
class OrderListViewModel {
    var orders: [Order] = []
    private var apiTask: Task<Void, Never>?
    
    func fetchOrders() {
        apiTask = Task { [weak self] in  // weak — ViewModel might be dismissed
            guard let self else { return }
            let result = try? await APIService.fetchOrders()
            self.orders = result ?? []
        }
    }
    
    deinit {
        apiTask?.cancel()
    }
}
// User navigates away → ViewModel is deallocated → [weak self] becomes nil → safe ✅
// If we used [unowned self] and the Task is slow, we'd crash 💥
```

---

## 5. GCD vs NSOperation

### Q6: What is the difference between GCD (Grand Central Dispatch) and NSOperation?

**Answer:**

| Feature | GCD (`DispatchQueue`) | NSOperation (`OperationQueue`) |
|---|---|---|
| Level | Low-level C API | High-level OOP wrapper over GCD |
| Cancellation | ❌ Cannot cancel dispatched work | ✅ Built-in `cancel()` |
| Dependencies | ❌ Not supported | ✅ `addDependency()` between operations |
| Max concurrency | ❌ No built-in limit | ✅ `maxConcurrentOperationCount` |
| State observation | ❌ No | ✅ KVO on `isFinished`, `isExecuting` |
| Reusability | ❌ Closures are one-off | ✅ Subclass `Operation` for reuse |
| Simplicity | ✅ Very simple API | ⚠️ More boilerplate |
| Performance | ✅ Slightly faster (closer to metal) | ⚠️ Slight overhead from OOP |
| Modern replacement | `async/await` + `Task` | `async/await` + `TaskGroup` |

```swift
// GCD — Simple, fire-and-forget
DispatchQueue.global(qos: .userInitiated).async {
    let data = processImage(image)
    DispatchQueue.main.async {
        self.imageView.image = data
    }
}

// NSOperation — Complex workflow with dependencies
let downloadOp = BlockOperation { downloadImage() }
let filterOp = BlockOperation { applyFilter() }
let saveOp = BlockOperation { saveToDisk() }

filterOp.addDependency(downloadOp)   // filter AFTER download
saveOp.addDependency(filterOp)       // save AFTER filter

let queue = OperationQueue()
queue.maxConcurrentOperationCount = 3
queue.addOperations([downloadOp, filterOp, saveOp], waitUntilFinished: false)
```

**Real-World Example 1 — Image Processing Pipeline (NSOperation):**
```swift
class ImageDownloadOperation: Operation {
    let url: URL
    var outputImage: UIImage?
    
    init(url: URL) { self.url = url }
    
    override func main() {
        guard !isCancelled else { return }  // ✅ Check cancellation
        let data = try? Data(contentsOf: url)
        guard !isCancelled else { return }  // ✅ Check again after slow work
        outputImage = data.flatMap { UIImage(data: $0) }
    }
}

// Usage — cancel all downloads when user scrolls away
let queue = OperationQueue()
queue.maxConcurrentOperationCount = 4

func loadImages(urls: [URL]) {
    queue.cancelAllOperations()  // Cancel previous batch ✅
    for url in urls {
        let op = ImageDownloadOperation(url: url)
        queue.addOperation(op)
    }
}
```

**Real-World Example 2 — Background Sync (GCD):**
```swift
// Simple one-shot background work — GCD is perfect
func syncDataInBackground() {
    DispatchQueue.global(qos: .background).async {
        let pendingChanges = CoreDataStack.shared.fetchPendingChanges()
        
        for change in pendingChanges {
            APIService.upload(change)
        }
        
        DispatchQueue.main.async {
            NotificationCenter.default.post(name: .syncCompleted, object: nil)
        }
    }
}
```

> **Modern approach:** Both GCD and NSOperation are largely replaced by `async/await` and `TaskGroup` in new code. Use GCD/NSOperation only in legacy codebases.

---

## 6. Closures — Complete Deep Dive

### Q7: What are the types of closures in Swift?

**Answer:**
```swift
// 1. NAMED CLOSURE (regular function)
func greet(name: String) -> String {
    return "Hello \(name)"
}

// 2. INLINE CLOSURE (anonymous function)
let greet = { (name: String) -> String in
    return "Hello \(name)"
}

// 3. TRAILING CLOSURE
[1, 2, 3].map { $0 * 2 }

// 4. ESCAPING CLOSURE — survives beyond function call
func fetchData(completion: @escaping (Data) -> Void) {
    URLSession.shared.dataTask(with: url) { data, _, _ in
        completion(data!)  // Called AFTER fetchData returns
    }.resume()
}

// 5. NON-ESCAPING CLOSURE — runs within function scope (default)
func process(items: [Int], transform: (Int) -> Int) -> [Int] {
    return items.map(transform)  // Runs immediately, doesn't escape
}

// 6. AUTOCLOSURE — wraps expression into a closure automatically
func log(_ message: @autoclosure () -> String, level: LogLevel = .debug) {
    guard level >= .currentLevel else { return }  // message() never called if not needed
    print(message())  // Only evaluated when actually needed ✅
}
log("User \(expensiveComputation()) logged in")  // Not computed if level is too low
```

---

### Q8: Is a closure a value type or a reference type?

**Answer:**
**Closures are REFERENCE types.** They are allocated on the heap and captured by reference.

```swift
class Counter {
    var count = 0
}

let counter = Counter()
let increment: () -> Void = {
    counter.count += 1  // closure captures `counter` by REFERENCE
}

increment()
increment()
print(counter.count)  // 2 — same object was modified

// This is why closures can create RETAIN CYCLES with classes:
// self → closure (stored as property) → self (captured) = CYCLE
```

**Real-World Example 1 — Proving closures are reference types:**
```swift
func makeCounter() -> () -> Int {
    var count = 0
    return {
        count += 1  // captures `count` by reference — shares the same variable
        return count
    }
}

let counter1 = makeCounter()
print(counter1())  // 1
print(counter1())  // 2
print(counter1())  // 3 — same captured `count` variable

let counter2 = makeCounter()
print(counter2())  // 1 — NEW captured variable
```

**Real-World Example 2 — Closure reference type causing retain cycle in BLE manager:**
```swift
// ❌ Retain cycle
class DeviceViewController: UIViewController {
    var onDeviceFound: ((CBPeripheral) -> Void)?
    
    func startScan() {
        onDeviceFound = { device in
            self.showDevice(device)  // ❌ self → onDeviceFound → self
        }
    }
}

// ✅ Fix
func startScan() {
    onDeviceFound = { [weak self] device in
        self?.showDevice(device)
    }
}
```

---

### Q9: What is a Capture List and how does it work?

**Answer:**
A capture list defines **how** variables are captured by a closure — by value or by reference, and with what strength (strong/weak/unowned).

```swift
var x = 10
var y = 20

// WITHOUT capture list — captures by REFERENCE
let closure1 = {
    print(x, y)  // will print current values when called
}
x = 99
closure1()  // prints "99 20" — x was modified after closure creation

// WITH capture list — captures by VALUE (snapshot at creation time)
let closure2 = { [x, y] in
    print(x, y)  // prints values frozen at creation time
}
x = 999
closure2()  // prints "99 20" — captured the value at creation, not current

// REFERENCE TYPE in capture list
class Logger { var tag = "default" }
var logger = Logger()

let closure3 = { [weak logger] in
    print(logger?.tag ?? "nil")  // weak reference — might be nil
}

logger = Logger()  // original logger has no more strong refs
closure3()  // prints "nil" — original logger was deallocated
```

**Real-World Example — Timer in a ViewModel:**
```swift
class DashboardViewModel {
    var refreshTimer: Timer?
    var data: [String] = []
    
    func startAutoRefresh() {
        refreshTimer = Timer.scheduledTimer(
            withTimeInterval: 30,
            repeats: true
        ) { [weak self] _ in  // capture list breaks retain cycle
            guard let self else { return }
            Task { await self.refreshData() }
        }
    }
    
    deinit {
        refreshTimer?.invalidate()
        print("✅ DashboardViewModel deallocated")
    }
}
```

---

### Q10: What is `@autoclosure` and where is it used?

**Answer:**
`@autoclosure` automatically wraps an **expression** into a **closure** without the caller needing `{ }` braces. It's used for **lazy evaluation** — the expression is only computed when the closure is called.

```swift
// WITHOUT autoclosure — caller must write { }
func logVerbose(_ message: () -> String) {
    if isVerboseMode {
        print(message())
    }
}
logVerbose({ "User \(computeExpensiveId()) logged in" })  // ugly { }

// WITH autoclosure — caller writes a normal expression
func logVerbose(_ message: @autoclosure () -> String) {
    if isVerboseMode {
        print(message())  // Only evaluated if verbose mode is on
    }
}
logVerbose("User \(computeExpensiveId()) logged in")  // clean! ✅
```

**Real-World Example 1 — Swift's built-in `assert()`:**
```swift
// assert uses @autoclosure — condition is NOT evaluated in Release builds
assert(array.count > 0, "Array must not be empty")
// The `array.count > 0` expression is wrapped in a closure
// In Release mode, the closure is never called → zero performance cost
```

**Real-World Example 2 — Default value with lazy evaluation:**
```swift
func fetchSetting(_ key: String, default defaultValue: @autoclosure () -> String) -> String {
    return UserDefaults.standard.string(forKey: key) ?? defaultValue()
}

// The expensive default is only computed if the key doesn't exist:
let theme = fetchSetting("theme", default: computeDefaultTheme())
```

---

### Q11: If I have a closure in Class A and navigate to Class B, does the closure still exist?

**Answer:**
**It depends on whether the closure is stored and who owns it.**

```swift
// SCENARIO 1: Closure stored as property — YES, it persists
class ClassA: UIViewController {
    var onComplete: (() -> Void)?  // stored property
    
    func goToB() {
        let classB = ClassB()
        classB.callback = { [weak self] in
            self?.handleCompletion()  // ClassA's closure is still alive
        }
        navigationController?.pushViewController(classB, animated: true)
    }
}

class ClassB: UIViewController {
    var callback: (() -> Void)?  // holds reference to closure
    
    func done() {
        callback?()  // calls ClassA's closure — it still exists
        navigationController?.popViewController(animated: true)
    }
}
// The closure exists because ClassB holds a strong reference to it.
// ClassB holds closure → closure captures [weak self] → ClassA (weak)
```

```swift
// SCENARIO 2: Closure NOT stored — it's deallocated when function ends
class ClassA: UIViewController {
    func goToB() {
        let closure = { print("Hello") }
        // closure exists only in this function's scope
        let classB = ClassB()
        navigationController?.pushViewController(classB, animated: true)
        // closure is deallocated when goToB() returns
    }
}
```

**Real-World Example — Completion handler pattern:**
```swift
// Express Scripts app — prescription refill flow
class PrescriptionListVC: UIViewController {
    func showRefillScreen(for prescription: Prescription) {
        let refillVC = RefillViewController()
        refillVC.onRefillComplete = { [weak self] updatedPrescription in
            // This closure exists as long as refillVC exists
            self?.updatePrescription(updatedPrescription)
            self?.navigationController?.popViewController(animated: true)
        }
        navigationController?.pushViewController(refillVC, animated: true)
    }
}
```

---

## 7. Protocols — Deep Dive

### Q12: How do you make a protocol available only for classes (not structs)?

**Answer:**
Use `AnyObject` (or the older `class` keyword) as a constraint:

```swift
// ✅ Class-only protocol — needed for weak delegates
protocol DeviceManagerDelegate: AnyObject {
    func didDiscoverDevice(_ device: Device)
    func didFailWithError(_ error: Error)
}

class DeviceManager {
    weak var delegate: DeviceManagerDelegate?  // weak requires AnyObject protocol
}

// ❌ This would NOT compile:
// struct MyHandler: DeviceManagerDelegate { }  // Error: struct can't conform to AnyObject

// ✅ Only classes can conform:
class MyViewController: UIViewController, DeviceManagerDelegate {
    func didDiscoverDevice(_ device: Device) { }
    func didFailWithError(_ error: Error) { }
}
```

**Why does this matter?**
- `weak` can only be used with reference types (classes)
- Delegate protocols MUST be `AnyObject` so the delegate can be `weak`
- Without `weak`, you get a retain cycle: Object → Delegate → Object

---

### Q13: What is `associatedtype` in protocols?

**Answer:**
`associatedtype` is a **placeholder type** in a protocol that the conforming type fills in. It makes protocols generic.

```swift
// Protocol with associatedtype
protocol Repository {
    associatedtype Item  // placeholder — conforming type decides what Item is
    
    func getAll() async throws -> [Item]
    func getById(_ id: String) async throws -> Item?
    func save(_ item: Item) async throws
    func delete(_ id: String) async throws
}

// Conforming type 1: Item = User
class UserRepository: Repository {
    typealias Item = User  // fills in the placeholder
    
    func getAll() async throws -> [User] { /* ... */ }
    func getById(_ id: String) async throws -> User? { /* ... */ }
    func save(_ item: User) async throws { /* ... */ }
    func delete(_ id: String) async throws { /* ... */ }
}

// Conforming type 2: Item = Order
class OrderRepository: Repository {
    typealias Item = Order
    
    func getAll() async throws -> [Order] { /* ... */ }
    func getById(_ id: String) async throws -> Order? { /* ... */ }
    func save(_ item: Order) async throws { /* ... */ }
    func delete(_ id: String) async throws { /* ... */ }
}
```

**Real-World Example — Generic Data Source:**
```swift
protocol DataSource {
    associatedtype Model: Identifiable
    var items: [Model] { get }
    func fetch() async throws
}

class ProductDataSource: DataSource {
    typealias Model = Product
    @Published var items: [Product] = []
    
    func fetch() async throws {
        items = try await APIService.fetchProducts()
    }
}
```

---

## 8. SOLID Principles in iOS

### Q14: Explain SOLID principles with iOS examples.

**Answer:**

**S — Single Responsibility Principle:**
Each class should have ONE reason to change.
```swift
// ❌ BAD: ViewController does networking + UI + data parsing
class UserVC: UIViewController {
    func fetchUser() { /* networking */ }
    func parseJSON() { /* parsing */ }
    func showAlert() { /* UI */ }
}

// ✅ GOOD: Separated responsibilities
class UserViewController: UIViewController { /* only UI */ }
class UserService: { /* only networking */ }
class UserParser: { /* only parsing */ }
```

**O — Open/Closed Principle:**
Open for extension, closed for modification.
```swift
// ✅ Use protocols to extend behavior without modifying existing code
protocol PaymentMethod {
    func processPayment(amount: Double) async throws -> PaymentResult
}

class CreditCardPayment: PaymentMethod { /* ... */ }
class ApplePayPayment: PaymentMethod { /* ... */ }
class UPIPayment: PaymentMethod { /* ... */ }  // Added later — no existing code changed
```

**L — Liskov Substitution:**
Subtypes must be substitutable for their base types.
```swift
// ✅ Any Repository<User> can be swapped without breaking code
let repo: any Repository = isTest ? MockUserRepository() : APIUserRepository()
```

**I — Interface Segregation:**
Don't force types to implement methods they don't need.
```swift
// ❌ BAD: One fat protocol
protocol Worker {
    func code()
    func design()
    func test()
    func manage()
}

// ✅ GOOD: Segregated protocols
protocol Coder { func code() }
protocol Designer { func design() }
protocol Tester { func test() }

class IOSDeveloper: Coder, Tester {
    func code() { }
    func test() { }
}
```

**D — Dependency Inversion:**
Depend on abstractions (protocols), not concretions (classes).
```swift
// ✅ ViewModel depends on protocol, not concrete class
class OrderViewModel {
    private let service: OrderServiceProtocol  // protocol, not OrderService
    
    init(service: OrderServiceProtocol) {  // injected
        self.service = service
    }
}
```

---

## 9. Protocol-Oriented Programming

### Q15: What is Protocol-Oriented Programming and how is it different from OOP?

**Answer:**

| Feature | OOP (Class Inheritance) | POP (Protocol Composition) |
|---|---|---|
| Code reuse | Inherit from base class | Protocol extensions with default implementations |
| Multiple inheritance | ❌ Not allowed | ✅ Conform to multiple protocols |
| Value types | ❌ Classes only | ✅ Structs AND classes |
| Diamond problem | ✅ Possible | ❌ Avoided |
| Flexibility | Rigid hierarchy | Composable, mix-and-match |

```swift
// POP — compose behaviors via protocols
protocol Cacheable {
    var cacheKey: String { get }
    var cacheExpiry: TimeInterval { get }
}

extension Cacheable {
    var cacheExpiry: TimeInterval { 300 }  // default: 5 minutes
}

protocol Trackable {
    var analyticsName: String { get }
}

// Any type can adopt any combination:
struct UserProfile: Cacheable, Trackable, Codable {
    let id: String
    let name: String
    
    var cacheKey: String { "user_\(id)" }
    var analyticsName: String { "user_profile" }
}
```

**Real-World Example — IoT Device Protocols:**
```swift
protocol Controllable {
    func turnOn() async throws
    func turnOff() async throws
}

protocol Dimmable {
    func setLevel(_ level: Double) async throws  // 0.0 to 1.0
}

protocol ColorChangeable {
    func setColor(_ color: UIColor) async throws
}

// Smart bulb supports all three
struct SmartBulb: Controllable, Dimmable, ColorChangeable {
    func turnOn() async throws { /* MQTT publish */ }
    func turnOff() async throws { /* MQTT publish */ }
    func setLevel(_ level: Double) async throws { /* ... */ }
    func setColor(_ color: UIColor) async throws { /* ... */ }
}

// Smart socket only supports on/off
struct SmartSocket: Controllable {
    func turnOn() async throws { /* ... */ }
    func turnOff() async throws { /* ... */ }
}
```

---

## 10. Core Data — Relations & Delete Rules

### Q16: What are relationships in Core Data and what are the delete rules?

**Answer:**

**Relationship types:**
```
One-to-One:   User ←→ Profile
One-to-Many:  User ←→ [Order]
Many-to-Many: Student ←→ [Course]
```

**Delete Rules:**

| Rule | What Happens When Parent is Deleted | Use Case |
|---|---|---|
| **Cascade** | All related children are DELETED too | User deleted → delete all their Orders |
| **Nullify** | Children's relationship is set to `nil` | Author deleted → Books still exist, `author = nil` |
| **Deny** | Deletion is BLOCKED if children exist | Can't delete Category if it has Products |
| **No Action** | Nothing happens (can leave orphans) | Rarely used — you manage manually |

```swift
// Core Data Model Setup:
// Entity: User
//   - name: String
//   - orders: [Order] (To-Many, Delete Rule: CASCADE)

// Entity: Order  
//   - amount: Double
//   - user: User (To-One, Delete Rule: NULLIFY)

// Usage:
let user = User(context: context)
user.name = "Rajesh"

let order1 = Order(context: context)
order1.amount = 99.99
order1.user = user  // sets the relationship

let order2 = Order(context: context)
order2.amount = 149.99
order2.user = user

try context.save()

// CASCADE delete — deleting user also deletes order1 and order2
context.delete(user)
try context.save()  // user, order1, order2 are ALL deleted
```

**Real-World Example — Express Scripts (Prescription Management):**
```swift
// Patient → Prescriptions (Cascade) — delete patient = delete prescriptions
// Prescription → Pharmacy (Nullify) — delete prescription, pharmacy still exists
// Pharmacy → Prescriptions (Deny) — can't delete pharmacy with active prescriptions
```

---

### Q17: What is the Core Data stack?

**Answer:**
```
┌─────────────────────────────────────────┐
│        NSPersistentContainer             │  ← Convenience wrapper (modern API)
│  ┌─────────────────────────────────┐    │
│  │    NSManagedObjectContext        │    │  ← Scratch pad for CRUD operations
│  │    (viewContext / background)    │    │
│  └──────────┬──────────────────────┘    │
│             │                            │
│  ┌──────────▼──────────────────────┐    │
│  │ NSPersistentStoreCoordinator    │    │  ← Mediator between context & store
│  └──────────┬──────────────────────┘    │
│             │                            │
│  ┌──────────▼──────────────────────┐    │
│  │    NSPersistentStore             │    │  ← Actual SQLite file on disk
│  │    (SQLite / In-Memory)          │    │
│  └─────────────────────────────────┘    │
│                                          │
│  NSManagedObjectModel (.xcdatamodeld)    │  ← Schema definition (entities)
└─────────────────────────────────────────┘
```

```swift
// Modern Core Data setup:
class CoreDataStack {
    static let shared = CoreDataStack()
    
    let container: NSPersistentContainer
    
    var viewContext: NSManagedObjectContext {
        container.viewContext
    }
    
    init() {
        container = NSPersistentContainer(name: "MyAppModel")
        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Core Data failed: \(error)")
            }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
    }
    
    // Background context for heavy operations
    func performBackgroundTask(_ block: @escaping (NSManagedObjectContext) -> Void) {
        container.performBackgroundTask(block)
    }
}
```

---

## 11. ARC — Compile Time or Runtime?

### Q18: Is ARC a compile-time or runtime mechanism?

**Answer:**
**ARC is a COMPILE-TIME mechanism.** The Swift/Objective-C compiler inserts `retain` and `release` calls at compile time. There is NO runtime garbage collector.

```swift
// What YOU write:
func processUser() {
    let user = User(name: "Rajesh")  // created
    print(user.name)
    // function ends
}

// What the COMPILER generates (conceptually):
func processUser() {
    let user = User(name: "Rajesh")  // swift_retain(user)  ← compiler inserts
    print(user.name)
    // swift_release(user)  ← compiler inserts at end of scope
}
```

**However, the actual COUNTING happens at runtime:**
- Compile time: Compiler decides WHERE to insert retain/release
- Runtime: The reference count integer is incremented/decremented

**Real-World Example — Proving it's compile-time:**
```swift
// Xcode shows ARC operations in the disassembly:
// You can see swift_retain and swift_release calls in the compiled binary
// There's NO background thread scanning for unused objects (unlike Java GC)
```

---

## 12. Authentication Flow — Access Token & Refresh Token

### Q19: How do you implement an authentication flow with access tokens and refresh tokens?

**Answer:**

```
Login Flow:
┌──────┐   credentials    ┌──────┐   access_token    ┌─────┐
│ App  │ ───────────────→  │ Auth │ ──────────────→   │ App │
│      │                   │Server│   refresh_token   │     │
└──────┘                   └──────┘                   └─────┘

Token Refresh Flow (when access_token expires):
┌──────┐   refresh_token   ┌──────┐   new_access_token  ┌─────┐
│ App  │ ────────────────→  │ Auth │ ──────────────────→  │ App │
│      │                    │Server│   new_refresh_token  │     │
└──────┘                    └──────┘                      └─────┘
```

| Token | Lifespan | Purpose | Storage |
|---|---|---|---|
| **Access Token** | Short (15min–1hr) | Authenticate API requests | Keychain |
| **Refresh Token** | Long (days–months) | Get new access tokens | Keychain |

```swift
actor AuthManager {
    static let shared = AuthManager()
    
    private var accessToken: String?
    private var refreshToken: String?
    private var isRefreshing = false
    
    // Store tokens securely
    func saveTokens(access: String, refresh: String) {
        accessToken = access
        refreshToken = refresh
        try? KeychainManager.save(key: "access_token", data: Data(access.utf8))
        try? KeychainManager.save(key: "refresh_token", data: Data(refresh.utf8))
    }
    
    // Get valid access token (refresh if expired)
    func getValidAccessToken() async throws -> String {
        if let token = accessToken, !isTokenExpired(token) {
            return token
        }
        return try await refreshAccessToken()
    }
    
    private func refreshAccessToken() async throws -> String {
        guard !isRefreshing else {
            // Another refresh is in progress — wait for it
            try await Task.sleep(for: .milliseconds(500))
            return try await getValidAccessToken()
        }
        
        isRefreshing = true
        defer { isRefreshing = false }
        
        guard let refresh = refreshToken else {
            throw AuthError.notAuthenticated
        }
        
        let response = try await APIService.refreshToken(refresh)
        saveTokens(access: response.accessToken, refresh: response.refreshToken)
        return response.accessToken
    }
    
    private func isTokenExpired(_ token: String) -> Bool {
        // Decode JWT and check expiry
        guard let payload = decodeJWT(token),
              let exp = payload["exp"] as? TimeInterval else { return true }
        return Date().timeIntervalSince1970 > exp - 60  // 60s buffer
    }
}

// Usage in API Client — auto-attaches token:
class APIClient {
    func request<T: Decodable>(_ endpoint: String) async throws -> T {
        let token = try await AuthManager.shared.getValidAccessToken()
        
        var request = URLRequest(url: URL(string: baseURL + endpoint)!)
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        
        let (data, response) = try await URLSession.shared.data(for: request)
        
        if (response as? HTTPURLResponse)?.statusCode == 401 {
            // Token expired mid-request — retry with fresh token
            let newToken = try await AuthManager.shared.getValidAccessToken()
            request.setValue("Bearer \(newToken)", forHTTPHeaderField: "Authorization")
            let (retryData, _) = try await URLSession.shared.data(for: request)
            return try JSONDecoder().decode(T.self, from: retryData)
        }
        
        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

---

### Q20: What is the difference between Authentication and Authorization?

**Answer:**

| Concept | Authentication (AuthN) | Authorization (AuthZ) |
|---|---|---|
| Question answered | "**Who** are you?" | "**What** can you do?" |
| Mechanism | Login (email/password, biometric, OAuth) | Roles, permissions, scopes |
| Token | Proves identity | Defines allowed actions |
| Example | User logs in with Apple ID | User has "admin" role → can delete posts |
| iOS Implementation | `ASAuthorizationAppleIDProvider`, Firebase Auth | JWT claims, API permission checks |

```swift
// Authentication — "Who are you?"
let appleIDProvider = ASAuthorizationAppleIDProvider()
let request = appleIDProvider.createRequest()
request.requestedScopes = [.fullName, .email]

// Authorization — "What can you do?"
struct UserPermissions: Codable {
    let canEditProfile: Bool
    let canDeletePosts: Bool
    let canManageUsers: Bool  // admin only
    let maxUploadSizeMB: Int
}

// Check authorization before allowing action:
func deletePost(_ post: Post) async throws {
    guard currentUser.permissions.canDeletePosts else {
        throw AppError.unauthorized("You don't have permission to delete posts")
    }
    try await apiClient.delete("/posts/\(post.id)")
}
```

---

## 13. Structured Concurrency — actor vs class vs struct

### Q21: When do you use `actor` vs `class` vs `struct`?

**Answer:**

| Feature | `struct` | `class` | `actor` |
|---|---|---|---|
| Type | Value | Reference | Reference |
| Thread safe | ✅ Copies isolate data | ❌ Manual locking needed | ✅ Compiler-enforced |
| Inheritance | ❌ | ✅ | ❌ |
| Identity (===) | ❌ | ✅ | ✅ |
| External access | Synchronous | Synchronous | `await` required |
| `Sendable` | ✅ Usually automatic | ❌ Manual effort | ✅ Automatic |
| Use case | Data models, DTOs | ViewModels, UIKit classes | Shared mutable state |

```swift
// STRUCT — Simple data, no sharing needed
struct DeviceReading: Sendable {
    let temperature: Double
    let humidity: Double
    let timestamp: Date
}

// CLASS — ViewModel that SwiftUI observes
@Observable
class HomeViewModel {
    var readings: [DeviceReading] = []
    var isLoading = false
}

// ACTOR — Shared state accessed from multiple threads
actor DeviceCache {
    private var cache: [UUID: DeviceReading] = [:]
    
    func store(_ reading: DeviceReading, for deviceId: UUID) {
        cache[deviceId] = reading
    }
    
    func latest(for deviceId: UUID) -> DeviceReading? {
        cache[deviceId]
    }
}
```

---

## 14. Task Cancellation & Detached Tasks

### Q22: How do `Task.cancel()` and `Task.detached` work?

**Answer:**

```swift
// CANCELLATION — cooperative, not forced
class SearchViewModel {
    private var searchTask: Task<Void, Never>?
    
    func search(query: String) {
        searchTask?.cancel()  // Cancel previous search
        
        searchTask = Task {
            try? await Task.sleep(for: .milliseconds(300))  // debounce
            
            guard !Task.isCancelled else { return }  // Check before API call
            
            let results = try? await APIService.search(query)
            
            guard !Task.isCancelled else { return }  // Check after API call
            
            await MainActor.run { self.results = results ?? [] }
        }
    }
}

// DETACHED — no inherited context
class ImageProcessor {
    @MainActor
    func process(image: UIImage) {
        // Task {} inherits @MainActor — still on main thread
        Task {
            // This is on MainActor — DON'T do heavy work here
        }
        
        // Task.detached — truly background, no MainActor
        Task.detached(priority: .userInitiated) {
            let processed = await self.applyFilters(image)  // background thread ✅
            await MainActor.run {
                self.displayImage(processed)  // back to main for UI
            }
        }
    }
}
```

---

## 15. NavigationStack & NavigationPath

### Q23: How do NavigationStack and NavigationPath work together?

**Answer:**

```swift
// Define routes as a type-safe enum
enum AppRoute: Hashable {
    case deviceDetail(deviceId: UUID)
    case deviceSettings(deviceId: UUID)
    case roomList
    case addDevice
}

@Observable
class Router {
    var path = NavigationPath()
    
    func push(_ route: AppRoute) { path.append(route) }
    func popToRoot() { path = NavigationPath() }
    func pop() { if !path.isEmpty { path.removeLast() } }
}

struct HomeView: View {
    @State var router = Router()
    
    var body: some View {
        NavigationStack(path: $router.path) {
            DeviceListView()
                .navigationDestination(for: AppRoute.self) { route in
                    switch route {
                    case .deviceDetail(let id):
                        DeviceDetailView(deviceId: id)
                    case .deviceSettings(let id):
                        DeviceSettingsView(deviceId: id)
                    case .roomList:
                        RoomListView()
                    case .addDevice:
                        AddDeviceView()
                    }
                }
        }
        .environment(router)
    }
}

// Deep linking — push multiple screens at once:
func handleDeepLink(deviceId: UUID) {
    router.popToRoot()
    router.push(.deviceDetail(deviceId: deviceId))
    router.push(.deviceSettings(deviceId: deviceId))
}
```

---

## 16. Offline Storage Management

### Q24: How do you design offline storage for an iOS app?

**Answer:**

| Storage | Best For | Size Limit | Persistence |
|---|---|---|---|
| `UserDefaults` | Small settings, flags | ~1 MB | Until app deleted |
| `Keychain` | Tokens, passwords | Small | Survives app delete |
| `Core Data` / `SwiftData` | Structured data with relationships | Large | Until app deleted |
| `FileManager` | Images, PDFs, large files | Device storage | Until app deleted |
| `NSCache` | Temporary in-memory cache | RAM limited | Until memory pressure |
| `URLCache` | HTTP response cache | Configurable | Configurable |

**Real-World Example — Offline-first IoT App:**
```swift
// Strategy: Cache API data locally, sync when online
actor OfflineStorageManager {
    private let coreDataStack = CoreDataStack.shared
    private var pendingChanges: [PendingChange] = []
    
    // Save data locally whether online or offline
    func saveDeviceReading(_ reading: DeviceReading) async {
        // 1. Save to Core Data immediately
        await coreDataStack.save(reading)
        
        // 2. Queue for server sync
        pendingChanges.append(PendingChange(type: .create, data: reading))
        
        // 3. Try to sync if online
        if NetworkMonitor.shared.isConnected {
            await syncPendingChanges()
        }
    }
    
    func syncPendingChanges() async {
        for change in pendingChanges {
            do {
                try await APIService.sync(change)
                pendingChanges.removeAll { $0.id == change.id }
            } catch {
                break  // Stop on first failure, retry later
            }
        }
    }
}

// Network monitor to detect online/offline:
import Network

class NetworkMonitor: ObservableObject {
    static let shared = NetworkMonitor()
    @Published var isConnected = true
    private let monitor = NWPathMonitor()
    
    init() {
        monitor.pathUpdateHandler = { [weak self] path in
            DispatchQueue.main.async {
                self?.isConnected = path.status == .satisfied
            }
        }
        monitor.start(queue: DispatchQueue.global())
    }
}
```

---

## 17. TCA — Central API Call Example

### Q25: Show a TCA (The Composable Architecture) example for fetching a user profile.

**Answer:**

```swift
import ComposableArchitecture

// 1. STATE — what the screen shows
@Reducer
struct UserProfileFeature {
    @ObservableState
    struct State: Equatable {
        var user: UserProfile?
        var isLoading = false
        var errorMessage: String?
    }
    
    // 2. ACTION — what can happen
    enum Action {
        case fetchProfile
        case profileResponse(Result<UserProfile, Error>)
        case retryTapped
        case logoutTapped
    }
    
    // 3. DEPENDENCY — injected API client
    @Dependency(\.apiClient) var apiClient
    
    // 4. REDUCER — business logic
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .fetchProfile:
                state.isLoading = true
                state.errorMessage = nil
                return .run { send in
                    do {
                        let user = try await apiClient.fetchUserProfile()
                        await send(.profileResponse(.success(user)))
                    } catch {
                        await send(.profileResponse(.failure(error)))
                    }
                }
                
            case .profileResponse(.success(let user)):
                state.isLoading = false
                state.user = user
                return .none
                
            case .profileResponse(.failure(let error)):
                state.isLoading = false
                state.errorMessage = error.localizedDescription
                return .none
                
            case .retryTapped:
                return .send(.fetchProfile)
                
            case .logoutTapped:
                state.user = nil
                return .none
            }
        }
    }
}

// 5. VIEW
struct UserProfileView: View {
    let store: StoreOf<UserProfileFeature>
    
    var body: some View {
        WithViewStore(store, observe: { $0 }) { viewStore in
            VStack {
                if viewStore.isLoading {
                    ProgressView()
                } else if let user = viewStore.user {
                    Text(user.name).font(.title)
                    Text(user.email).foregroundColor(.gray)
                } else if let error = viewStore.errorMessage {
                    Text(error).foregroundColor(.red)
                    Button("Retry") { viewStore.send(.retryTapped) }
                }
            }
            .onAppear { viewStore.send(.fetchProfile) }
        }
    }
}
```

---

## 18. VIPER Architecture

### Q26: Explain VIPER with a code example.

**Answer:**
```
V - View       → Displays UI, receives user input
I - Interactor  → Business logic, data fetching
P - Presenter   → Formats data for View, handles View actions
E - Entity      → Data models (structs)
R - Router      → Navigation logic
```

```swift
// ENTITY
struct User {
    let id: String
    let name: String
    let email: String
}

// PROTOCOLS (contracts between layers)
protocol UserListViewProtocol: AnyObject {
    func showUsers(_ users: [UserViewModel])
    func showError(_ message: String)
    func showLoading()
}

protocol UserListPresenterProtocol {
    func viewDidLoad()
    func didSelectUser(at index: Int)
}

protocol UserListInteractorProtocol {
    func fetchUsers() async throws -> [User]
}

protocol UserListRouterProtocol {
    func navigateToDetail(user: User)
}

// INTERACTOR — business logic
class UserListInteractor: UserListInteractorProtocol {
    func fetchUsers() async throws -> [User] {
        let (data, _) = try await URLSession.shared.data(from: apiURL)
        return try JSONDecoder().decode([User].self, from: data)
    }
}

// PRESENTER — coordinates everything
class UserListPresenter: UserListPresenterProtocol {
    weak var view: UserListViewProtocol?
    var interactor: UserListInteractorProtocol
    var router: UserListRouterProtocol
    private var users: [User] = []
    
    init(view: UserListViewProtocol, interactor: UserListInteractorProtocol, router: UserListRouterProtocol) {
        self.view = view
        self.interactor = interactor
        self.router = router
    }
    
    func viewDidLoad() {
        view?.showLoading()
        Task {
            do {
                users = try await interactor.fetchUsers()
                let viewModels = users.map { UserViewModel(name: $0.name, email: $0.email) }
                await MainActor.run { view?.showUsers(viewModels) }
            } catch {
                await MainActor.run { view?.showError(error.localizedDescription) }
            }
        }
    }
    
    func didSelectUser(at index: Int) {
        router.navigateToDetail(user: users[index])
    }
}

// ROUTER — navigation
class UserListRouter: UserListRouterProtocol {
    weak var viewController: UIViewController?
    
    func navigateToDetail(user: User) {
        let detailVC = UserDetailRouter.createModule(user: user)
        viewController?.navigationController?.pushViewController(detailVC, animated: true)
    }
    
    static func createModule() -> UIViewController {
        let view = UserListViewController()
        let interactor = UserListInteractor()
        let router = UserListRouter()
        let presenter = UserListPresenter(view: view, interactor: interactor, router: router)
        
        view.presenter = presenter
        router.viewController = view
        return view
    }
}
```

---

## 19. App Store Submission Process

### Q27: What is the complete App Store submission process?

**Answer:**

```
1. PREPARE
   ├── Archive build (Product → Archive)
   ├── Validate with App Store Connect
   ├── Prepare screenshots (all device sizes)
   ├── Write app description, keywords
   └── Set pricing & availability

2. UPLOAD
   ├── Upload via Xcode Organizer or Transporter
   ├── Or automate via Fastlane / GitHub Actions
   └── Wait for processing (~15-30 minutes)

3. TESTFLIGHT (Optional but recommended)
   ├── Internal Testing (up to 100 testers, no review needed)
   ├── External Testing (up to 10,000 testers, needs Beta Review)
   └── Gather feedback, fix bugs

4. SUBMIT FOR REVIEW
   ├── Select build in App Store Connect
   ├── Fill in review notes
   ├── Submit
   └── Review takes 24-48 hours typically

5. RELEASE
   ├── Manual Release — you choose when to publish
   ├── Automatic Release — goes live after approval
   └── Phased Release — 1% → 2% → 5% → 10% → 20% → 50% → 100% over 7 days
```

**Common Rejection Reasons:**
1. **Crashes on launch** — always test on a real device
2. **Placeholder content** — lorem ipsum text or missing images
3. **Missing privacy policy** — required for all apps
4. **Incomplete metadata** — missing screenshots or descriptions
5. **Guideline 4.3 (Spam)** — app too similar to existing apps
6. **Guideline 2.1 (Performance)** — app doesn't work as described
7. **Missing login credentials** — reviewer can't test login-required features

---

## 20. Caching Strategies

### Q28: How do you implement caching in an iOS app?

**Answer:**

```swift
// LAYER 1: In-Memory Cache (fastest, cleared on app kill)
actor InMemoryCache {
    private let cache = NSCache<NSString, CacheEntry>()
    
    init() {
        cache.countLimit = 100
        cache.totalCostLimit = 50_000_000  // 50 MB
    }
    
    func get<T: Codable>(_ key: String) -> T? {
        guard let entry = cache.object(forKey: key as NSString),
              !entry.isExpired else { return nil }
        return try? JSONDecoder().decode(T.self, from: entry.data)
    }
    
    func set<T: Codable>(_ value: T, for key: String, ttl: TimeInterval = 300) {
        guard let data = try? JSONEncoder().encode(value) else { return }
        let entry = CacheEntry(data: data, expiry: Date().addingTimeInterval(ttl))
        cache.setObject(entry, forKey: key as NSString)
    }
}

class CacheEntry: NSObject {
    let data: Data
    let expiry: Date
    var isExpired: Bool { Date() > expiry }
    
    init(data: Data, expiry: Date) {
        self.data = data
        self.expiry = expiry
    }
}

// LAYER 2: Disk Cache (survives app restart)
actor DiskCache {
    private let directory: URL
    
    init(name: String) {
        directory = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
            .appendingPathComponent(name)
        try? FileManager.default.createDirectory(at: directory, withIntermediateDirectories: true)
    }
    
    func save<T: Codable>(_ value: T, key: String) throws {
        let data = try JSONEncoder().encode(value)
        let fileURL = directory.appendingPathComponent(key.md5Hash)
        try data.write(to: fileURL)
    }
    
    func load<T: Codable>(key: String) throws -> T? {
        let fileURL = directory.appendingPathComponent(key.md5Hash)
        guard FileManager.default.fileExists(atPath: fileURL.path) else { return nil }
        let data = try Data(contentsOf: fileURL)
        return try JSONDecoder().decode(T.self, from: data)
    }
}

// COMBINED — check memory first, then disk, then network
class CacheManager {
    private let memory = InMemoryCache()
    private let disk = DiskCache(name: "api_cache")
    
    func fetch<T: Codable>(key: String, loader: () async throws -> T) async throws -> T {
        // 1. Check memory
        if let cached: T = await memory.get(key) { return cached }
        
        // 2. Check disk
        if let cached: T = try await disk.load(key: key) {
            await memory.set(cached, for: key)
            return cached
        }
        
        // 3. Fetch from network
        let value = try await loader()
        await memory.set(value, for: key)
        try? await disk.save(value, key: key)
        return value
    }
}
```

---

## 21. Behavioral & System Design Questions

### Q29: Tell me about a challenging technical problem you solved. (STAR Method)

**Answer Template:**
```
SITUATION: "At Blaze Automation, we were building the B.One smart home app 
that controlled IoT devices via BLE and MQTT."

TASK: "Users reported that BLE connections would randomly drop when they 
moved between rooms, and devices would show as 'offline' even though they 
were working fine."

ACTION: "I implemented a multi-layer connection recovery system:
1. Added CoreBluetooth state restoration so the app could resume BLE 
   connections after being killed by iOS
2. Implemented an exponential backoff retry mechanism for MQTT reconnection
3. Created a local cache of last-known device states so the UI wouldn't 
   show 'offline' during brief disconnections
4. Added a heartbeat system that pinged devices every 30 seconds and only 
   marked them offline after 3 consecutive failures"

RESULT: "Connection reliability improved from ~70% to ~98%. User complaints 
about 'offline devices' dropped by 85%. The solution was reused across all 
our IoT apps (Vida Genio, Raycon)."
```

---

### Q30: How do you handle disagreements with team members about technical decisions?

**Answer:**
```
"I approach technical disagreements with data, not opinions:

1. LISTEN FIRST — I make sure I fully understand their perspective before 
   responding. Often they have context I'm missing.

2. WRITE IT DOWN — I propose we both document our approaches with pros/cons
   in a short ADR (Architecture Decision Record). This removes emotion.

3. PROTOTYPE — If the decision is significant (like choosing MVVM vs VIPER),
   I suggest we spike both approaches for 2-4 hours and compare.

EXAMPLE: At Cognizant, a colleague wanted to use RxSwift for the Express 
Scripts app. I preferred Combine + async/await. Instead of arguing, I built 
a small networking layer using both approaches. We compared:
- Combine: No third-party dependency, Apple-supported, smaller binary size
- RxSwift: More operators, larger community, but adds 2MB to binary

The team chose Combine because the client (Cigna) had strict policies about 
third-party dependencies in healthcare apps."
```

---

### Q31: How do you prioritize tasks when you have multiple urgent bugs AND a feature deadline?

**Answer:**
```
"I use a priority matrix:

┌─────────────────────┬──────────────────────┐
│  URGENT + IMPORTANT │  IMPORTANT, NOT URGENT│
│  Production crash    │  Feature deadline     │
│  Data loss bug      │  Architecture refactor│
│  Security vuln      │  Tech debt            │
│  → DO IMMEDIATELY   │  → SCHEDULE           │
├─────────────────────┼──────────────────────┤
│  URGENT, NOT        │  NEITHER              │
│  IMPORTANT          │                       │
│  UI glitch in       │  Nice-to-have polish  │
│  non-critical screen│  Minor code cleanup   │
│  → DELEGATE/LATER   │  → DROP               │
└─────────────────────┴──────────────────────┘

EXAMPLE: During the Royal Caribbean app release, we had:
1. Crash on booking confirmation (P0 — fixed immediately)
2. Feature deadline for room selection (P1 — continued after P0 fix)
3. Font size too small on iPad (P2 — logged for next sprint)

I fixed the crash within 2 hours, pushed a hotfix via TestFlight, 
then returned to the feature work. I communicated the delay to the 
PM with a revised timeline."
```

---

### Q32: Design an image caching system from scratch. (System Design)

**Answer:**
```swift
// Two-tier cache: Memory (fast) + Disk (persistent)
actor ImageCacheSystem {
    static let shared = ImageCacheSystem()
    
    // Tier 1: Memory cache
    private let memoryCache = NSCache<NSURL, UIImage>()
    
    // Tier 2: Disk cache
    private let diskCachePath: URL
    
    // Track in-flight downloads to avoid duplicate requests
    private var inFlightTasks: [URL: Task<UIImage, Error>] = [:]
    
    init() {
        memoryCache.countLimit = 200
        memoryCache.totalCostLimit = 100_000_000  // 100 MB
        
        diskCachePath = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
            .appendingPathComponent("ImageCache")
        try? FileManager.default.createDirectory(at: diskCachePath, withIntermediateDirectories: true)
    }
    
    func image(for url: URL) async throws -> UIImage {
        // 1. Check memory cache
        if let cached = memoryCache.object(forKey: url as NSURL) {
            return cached
        }
        
        // 2. Check disk cache
        let diskPath = diskCachePath.appendingPathComponent(url.absoluteString.md5Hash)
        if let data = try? Data(contentsOf: diskPath),
           let image = UIImage(data: data) {
            memoryCache.setObject(image, forKey: url as NSURL)
            return image
        }
        
        // 3. Check if already downloading
        if let existingTask = inFlightTasks[url] {
            return try await existingTask.value
        }
        
        // 4. Download from network
        let task = Task<UIImage, Error> {
            let (data, _) = try await URLSession.shared.data(from: url)
            guard let image = UIImage(data: data) else { throw ImageError.invalidData }
            
            // Save to both caches
            memoryCache.setObject(image, forKey: url as NSURL)
            try? data.write(to: diskPath)
            
            return image
        }
        
        inFlightTasks[url] = task
        let image = try await task.value
        inFlightTasks[url] = nil
        return image
    }
    
    func clearMemoryCache() {
        memoryCache.removeAllObjects()
    }
}

// SwiftUI usage:
struct AsyncCachedImage: View {
    let url: URL
    @State private var image: UIImage?
    
    var body: some View {
        Group {
            if let image {
                Image(uiImage: image).resizable().aspectRatio(contentMode: .fill)
            } else {
                ProgressView()
            }
        }
        .task {
            image = try? await ImageCacheSystem.shared.image(for: url)
        }
    }
}
```

---

### Q33: How would you architect a multi-module iOS app for a team of 10 developers?

**Answer:**
```
MyApp/
├── App/                          ← Main app target (thin, just composition)
│   ├── AppDelegate.swift
│   └── ContentView.swift
│
├── Packages/
│   ├── CoreKit/                  ← Shared utilities (Team: Everyone)
│   │   ├── Extensions/
│   │   ├── Logger/
│   │   └── Constants/
│   │
│   ├── NetworkKit/               ← API client (Team: Backend Integration)
│   │   ├── APIClient.swift
│   │   ├── Endpoints/
│   │   └── DTOs/
│   │
│   ├── DesignKit/                ← UI components (Team: Design System)
│   │   ├── Buttons/
│   │   ├── Colors/
│   │   └── Typography/
│   │
│   ├── AuthKit/                  ← Auth flow (Team: Auth)
│   │   ├── LoginView.swift
│   │   ├── AuthManager.swift
│   │   └── KeychainManager.swift
│   │
│   ├── FeatureHome/              ← Home tab (Team: Home)
│   ├── FeatureProfile/           ← Profile tab (Team: Profile)
│   ├── FeatureDevices/           ← Device mgmt (Team: IoT)
│   └── FeatureSettings/          ← Settings (Team: Settings)

Benefits:
✅ Teams work independently on separate packages
✅ Faster builds — only changed modules recompile
✅ Enforced boundaries — modules can't access each other's internals
✅ Each module has its own unit tests
✅ Feature flags control which modules are active
```
# 📱 Senior iOS Developer — Interview Q&A Guide (Part 3)

> Covers: UIKit, Generics, Error Handling, SwiftUI Advanced, Security, CI/CD, WebSockets, SwiftData, MVVM-C, and more.

---

## 22. UIKit — View Controller Lifecycle

### Q34: Explain the complete UIViewController lifecycle in order.

**Answer:**

```
init → loadView → viewDidLoad → viewWillAppear → viewDidAppear
                                                       ↓
                                            (user interacts)
                                                       ↓
viewWillDisappear → viewDidDisappear → deinit
```

```swift
class ProfileViewController: UIViewController {
    
    // 1. INIT — allocate memory, set initial properties
    init(userId: String) {
        self.userId = userId
        super.init(nibName: nil, bundle: nil)
        print("1️⃣ init — object created in memory")
    }
    
    // 2. loadView — create the view hierarchy (programmatic UI only)
    override func loadView() {
        super.loadView()
        print("2️⃣ loadView — view hierarchy created")
        // Only override if building 100% programmatic UI
    }
    
    // 3. viewDidLoad — CALLED ONCE — setup that happens once
    override func viewDidLoad() {
        super.viewDidLoad()
        print("3️⃣ viewDidLoad — view loaded into memory (ONCE)")
        // ✅ Setup UI, add subviews, register cells
        // ✅ Setup constraints
        // ✅ Start initial data fetch
        // ❌ Don't use frame-based layout here (frame not final)
        setupUI()
        fetchData()
    }
    
    // 4. viewWillAppear — CALLED EVERY TIME view is about to show
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        print("4️⃣ viewWillAppear — about to appear")
        // ✅ Refresh data, start animations
        // ✅ Update navigation bar style
        navigationController?.setNavigationBarHidden(false, animated: true)
    }
    
    // 5. viewDidAppear — view is now visible on screen
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        print("5️⃣ viewDidAppear — now visible")
        // ✅ Start animations, timers, tracking
        // ✅ Show onboarding tooltips
        Analytics.trackScreen("profile")
    }
    
    // 6. viewWillDisappear — about to leave
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        print("6️⃣ viewWillDisappear — about to leave")
        // ✅ Pause video/audio, save draft
        // ✅ Cancel pending network requests
    }
    
    // 7. viewDidDisappear — fully gone
    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        print("7️⃣ viewDidDisappear — no longer visible")
        // ✅ Stop timers, invalidate resources
    }
    
    // 8. deinit — object destroyed
    deinit {
        print("8️⃣ deinit — memory freed")
    }
}
```

**Real-World Example — When navigating A → B → A:**
```
A: viewDidLoad (once)
A: viewWillAppear
A: viewDidAppear
  → Push B
A: viewWillDisappear
A: viewDidDisappear
B: viewDidLoad (once)
B: viewWillAppear
B: viewDidAppear
  → Pop back to A
B: viewWillDisappear
B: viewDidDisappear
A: viewWillAppear      ← called AGAIN
A: viewDidAppear       ← called AGAIN
B: deinit              ← B is freed
```

---

## 23. Auto Layout

### Q35: How do you create constraints programmatically?

**Answer:**

```swift
class LoginViewController: UIViewController {
    
    private let emailField: UITextField = {
        let tf = UITextField()
        tf.placeholder = "Email"
        tf.borderStyle = .roundedRect
        tf.translatesAutoresizingMaskIntoConstraints = false  // ⚠️ REQUIRED
        return tf
    }()
    
    private let loginButton: UIButton = {
        let btn = UIButton(type: .system)
        btn.setTitle("Log In", for: .normal)
        btn.translatesAutoresizingMaskIntoConstraints = false
        return btn
    }()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(emailField)
        view.addSubview(loginButton)
        
        // Anchors API (preferred modern approach):
        NSLayoutConstraint.activate([
            emailField.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            emailField.centerYAnchor.constraint(equalTo: view.centerYAnchor),
            emailField.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 32),
            emailField.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -32),
            emailField.heightAnchor.constraint(equalToConstant: 44),
            
            loginButton.topAnchor.constraint(equalTo: emailField.bottomAnchor, constant: 16),
            loginButton.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            loginButton.widthAnchor.constraint(equalTo: emailField.widthAnchor),
            loginButton.heightAnchor.constraint(equalToConstant: 50),
        ])
    }
}
```

**Content Hugging & Compression Resistance:**
```swift
// Content Hugging — "I don't want to grow bigger than my content"
// Higher priority = resists stretching more
label.setContentHuggingPriority(.required, for: .horizontal)  // 1000 = never stretch

// Compression Resistance — "I don't want to shrink smaller than my content"
// Higher priority = resists compression more
label.setContentCompressionResistancePriority(.required, for: .horizontal)  // 1000 = never shrink

// Real-world: Label + TextField side by side
// Label should NOT stretch, TextField should fill remaining space
nameLabel.setContentHuggingPriority(.defaultHigh, for: .horizontal)     // 750
nameTextField.setContentHuggingPriority(.defaultLow, for: .horizontal)  // 250
```

---

## 24. Diffable Data Source

### Q36: What is Diffable Data Source and how does it replace traditional data sources?

**Answer:**

```swift
// ✅ MODERN: Diffable Data Source — automatic animations, no crashes
class UserListViewController: UIViewController {
    
    enum Section { case main }
    
    var collectionView: UICollectionView!
    var dataSource: UICollectionViewDiffableDataSource<Section, User>!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        configureCollectionView()
        configureDataSource()
        applyInitialSnapshot()
    }
    
    func configureDataSource() {
        let cellRegistration = UICollectionView.CellRegistration<UICollectionViewListCell, User> { cell, indexPath, user in
            var config = cell.defaultContentConfiguration()
            config.text = user.name
            config.secondaryText = user.email
            cell.contentConfiguration = config
        }
        
        dataSource = UICollectionViewDiffableDataSource<Section, User>(
            collectionView: collectionView
        ) { collectionView, indexPath, user in
            collectionView.dequeueConfiguredReusableCell(using: cellRegistration, for: indexPath, item: user)
        }
    }
    
    func updateList(with users: [User]) {
        var snapshot = NSDiffableDataSourceSnapshot<Section, User>()
        snapshot.appendSections([.main])
        snapshot.appendItems(users)
        dataSource.apply(snapshot, animatingDifferences: true)  // ✅ Auto-animated!
    }
    
    func filterUsers(query: String) {
        let filtered = allUsers.filter { $0.name.localizedCaseInsensitiveContains(query) }
        var snapshot = NSDiffableDataSourceSnapshot<Section, User>()
        snapshot.appendSections([.main])
        snapshot.appendItems(filtered)
        dataSource.apply(snapshot, animatingDifferences: true)
        // ✅ No reloadData(), no crashes, smooth animations automatically
    }
}
```

**Why Diffable is better than traditional:**
| Old Way | Diffable Data Source |
|---|---|
| `numberOfRowsInSection` + `cellForRowAt` | Single `apply(snapshot)` call |
| Manual `reloadData()` — no animations | Automatic diffing + animations |
| `performBatchUpdates` crash-prone | ✅ Thread-safe, crash-free |
| Must track indexPaths carefully | Uses unique Hashable identifiers |

---

## 25. Generics

### Q37: What are Generics and how do you use type constraints?

**Answer:**

```swift
// GENERIC FUNCTION — works with any type
func swapValues<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

// GENERIC TYPE — reusable data structure
struct Stack<Element> {
    private var items: [Element] = []
    
    mutating func push(_ item: Element) { items.append(item) }
    mutating func pop() -> Element? { items.popLast() }
    var peek: Element? { items.last }
    var isEmpty: Bool { items.isEmpty }
}

var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)

var stringStack = Stack<String>()
stringStack.push("Hello")

// TYPE CONSTRAINTS — restrict what types can be used
func findIndex<T: Equatable>(of value: T, in array: [T]) -> Int? {
    array.firstIndex(of: value)  // Equatable needed for ==
}

// GENERIC with multiple constraints
func fetch<T: Codable & Sendable>(endpoint: String) async throws -> T {
    let (data, _) = try await URLSession.shared.data(from: URL(string: endpoint)!)
    return try JSONDecoder().decode(T.self, from: data)
}

// `some` (Opaque type) vs `any` (Existential type)
func makeView() -> some View { Text("Hello") }       // compiler knows exact type — fast
func makeViews() -> [any View] { [Text("A"), Image("B")] }  // type-erased — slower

// Real-world: Generic network layer
class APIClient {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        let (data, _) = try await URLSession.shared.data(for: endpoint.urlRequest)
        return try JSONDecoder().decode(T.self, from: data)
    }
}

// Usage — compiler infers T from context:
let users: [User] = try await apiClient.request(.getUsers)
let order: Order = try await apiClient.request(.getOrder(id: "123"))
```

---

## 26. Error Handling

### Q38: How does error handling work in Swift?

**Answer:**

```swift
// 1. Define custom errors
enum NetworkError: Error, LocalizedError {
    case noConnection
    case serverError(statusCode: Int)
    case decodingFailed(underlying: Error)
    case unauthorized
    case timeout
    
    var errorDescription: String? {
        switch self {
        case .noConnection: return "No internet connection"
        case .serverError(let code): return "Server error (\(code))"
        case .decodingFailed: return "Failed to parse response"
        case .unauthorized: return "Session expired. Please log in again."
        case .timeout: return "Request timed out"
        }
    }
}

// 2. Function that throws
func fetchUser(id: String) async throws -> User {
    guard NetworkMonitor.shared.isConnected else {
        throw NetworkError.noConnection
    }
    
    let (data, response) = try await URLSession.shared.data(from: url)
    
    guard let httpResponse = response as? HTTPURLResponse else {
        throw NetworkError.serverError(statusCode: 0)
    }
    
    switch httpResponse.statusCode {
    case 200...299:
        do {
            return try JSONDecoder().decode(User.self, from: data)
        } catch {
            throw NetworkError.decodingFailed(underlying: error)
        }
    case 401:
        throw NetworkError.unauthorized
    default:
        throw NetworkError.serverError(statusCode: httpResponse.statusCode)
    }
}

// 3. Handling errors
func loadUser() async {
    do {
        let user = try await fetchUser(id: "123")
        self.user = user
    } catch let error as NetworkError {
        switch error {
        case .unauthorized:
            router.navigate(to: .login)
        case .noConnection:
            showOfflineMode()
        default:
            showAlert(error.localizedDescription)
        }
    } catch {
        showAlert("Unexpected error: \(error)")
    }
}

// 4. try? — returns nil on failure (no error info)
let user = try? await fetchUser(id: "123")  // User? — nil if any error

// 5. try! — crashes on failure (only use when SURE it won't fail)
let config = try! JSONDecoder().decode(Config.self, from: bundledJSON)

// 6. Result type — for callbacks
func fetchUser(completion: @escaping (Result<User, NetworkError>) -> Void) {
    // ...
    completion(.success(user))
    // or
    completion(.failure(.noConnection))
}
```

---

## 27. SwiftUI — ViewBuilder, PreferenceKey, GeometryReader

### Q39: What is `@ViewBuilder` and how do you create custom container views?

**Answer:**

```swift
// @ViewBuilder lets you build views using SwiftUI's declarative syntax
struct Card<Content: View>: View {
    let title: String
    @ViewBuilder let content: Content
    
    init(title: String, @ViewBuilder content: () -> Content) {
        self.title = title
        self.content = content()
    }
    
    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            Text(title).font(.headline)
            content
        }
        .padding()
        .background(.ultraThinMaterial)
        .cornerRadius(16)
    }
}

// Usage — pass any views as content:
Card(title: "Device Status") {
    HStack {
        Image(systemName: "lightbulb.fill")
        Text("Living Room Light")
        Spacer()
        Toggle("", isOn: $isOn)
    }
    Text("Last updated: 2 min ago")
        .foregroundColor(.secondary)
}
```

### Q40: What is `PreferenceKey`? (Child-to-Parent communication)

**Answer:**

```swift
// PreferenceKey sends data from CHILD views UP to PARENT views
// (opposite of Environment which flows DOWN)

struct SizePreferenceKey: PreferenceKey {
    static var defaultValue: CGSize = .zero
    static func reduce(value: inout CGSize, nextValue: () -> CGSize) {
        value = nextValue()
    }
}

// Child reads its own size and sends it up:
struct ChildView: View {
    var body: some View {
        Text("Hello World")
            .background(
                GeometryReader { proxy in
                    Color.clear.preference(
                        key: SizePreferenceKey.self,
                        value: proxy.size
                    )
                }
            )
    }
}

// Parent receives the child's size:
struct ParentView: View {
    @State private var childSize: CGSize = .zero
    
    var body: some View {
        VStack {
            ChildView()
            Text("Child size: \(childSize.width) × \(childSize.height)")
        }
        .onPreferenceChange(SizePreferenceKey.self) { size in
            childSize = size
        }
    }
}
```

### Q41: What is `GeometryReader` and what are its pitfalls?

**Answer:**

```swift
// GeometryReader reads the available size and position of its container
struct ResponsiveGrid: View {
    var body: some View {
        GeometryReader { proxy in
            let width = proxy.size.width
            let columns = width > 600 ? 3 : 2
            let itemWidth = (width - CGFloat(columns + 1) * 16) / CGFloat(columns)
            
            LazyVGrid(columns: Array(repeating: GridItem(.fixed(itemWidth)), count: columns)) {
                ForEach(items) { item in
                    ItemCard(item: item)
                }
            }
        }
    }
}

// ⚠️ PITFALL: GeometryReader takes ALL available space
// It will push siblings out of the way and expand to fill its container
// FIX: Use .frame() to constrain it, or use it inside .background/.overlay
```

---

## 28. SSL Pinning & Security

### Q42: How do you implement SSL Certificate Pinning?

**Answer:**

```swift
// SSL Pinning prevents man-in-the-middle attacks by verifying
// the server's certificate against a known copy in the app bundle

class PinnedSessionDelegate: NSObject, URLSessionDelegate {
    
    func urlSession(_ session: URLSession,
                    didReceive challenge: URLAuthenticationChallenge,
                    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        
        guard let serverTrust = challenge.protectionSpace.serverTrust,
              let serverCert = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Get server's public key
        let serverPublicKey = SecCertificateCopyKey(serverCert)
        let serverKeyData = SecKeyCopyExternalRepresentation(serverPublicKey!, nil)! as Data
        
        // Compare with pinned key hash
        let serverKeyHash = SHA256.hash(data: serverKeyData)
            .compactMap { String(format: "%02x", $0) }.joined()
        
        let pinnedHashes = [
            "abc123def456..."  // Your server's public key hash
        ]
        
        if pinnedHashes.contains(serverKeyHash) {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)  // BLOCK connection
        }
    }
}

// Usage:
let session = URLSession(
    configuration: .default,
    delegate: PinnedSessionDelegate(),
    delegateQueue: nil
)
```

### Q43: How do you encrypt data on iOS using CryptoKit?

**Answer:**

```swift
import CryptoKit

// AES-GCM Encryption (symmetric — same key to encrypt & decrypt)
struct EncryptionManager {
    
    // Generate a random encryption key
    static func generateKey() -> SymmetricKey {
        SymmetricKey(size: .bits256)
    }
    
    // Encrypt
    static func encrypt(data: Data, key: SymmetricKey) throws -> Data {
        let sealedBox = try AES.GCM.seal(data, using: key)
        return sealedBox.combined!  // nonce + ciphertext + tag
    }
    
    // Decrypt
    static func decrypt(data: Data, key: SymmetricKey) throws -> Data {
        let sealedBox = try AES.GCM.SealedBox(combined: data)
        return try AES.GCM.open(sealedBox, using: key)
    }
    
    // Hash (one-way, for passwords/verification)
    static func hash(_ string: String) -> String {
        let data = Data(string.utf8)
        let digest = SHA256.hash(data: data)
        return digest.compactMap { String(format: "%02x", $0) }.joined()
    }
}

// Real-world: Encrypt sensitive health data before saving to disk
let healthData = "Blood pressure: 120/80".data(using: .utf8)!
let key = EncryptionManager.generateKey()

let encrypted = try EncryptionManager.encrypt(data: healthData, key: key)
// Save `encrypted` to file — unreadable without the key

let decrypted = try EncryptionManager.decrypt(data: encrypted, key: key)
let original = String(data: decrypted, encoding: .utf8)!  // "Blood pressure: 120/80"

// Store the key in Keychain (never in UserDefaults!)
try KeychainManager.save(key: "encryption_key", data: key.withUnsafeBytes { Data($0) })
```

---

## 29. WebSockets

### Q44: How do you implement WebSockets in iOS?

**Answer:**

```swift
class WebSocketManager: NSObject, URLSessionWebSocketDelegate {
    private var webSocket: URLSessionWebSocketTask?
    private var session: URLSession!
    var onMessage: ((String) -> Void)?
    
    func connect(to url: URL) {
        session = URLSession(configuration: .default, delegate: self, delegateQueue: nil)
        webSocket = session.webSocketTask(with: url)
        webSocket?.resume()
        receiveMessage()
    }
    
    func send(message: String) {
        webSocket?.send(.string(message)) { error in
            if let error { print("Send error: \(error)") }
        }
    }
    
    private func receiveMessage() {
        webSocket?.receive { [weak self] result in
            switch result {
            case .success(let message):
                switch message {
                case .string(let text):
                    self?.onMessage?(text)
                case .data(let data):
                    print("Received binary: \(data.count) bytes")
                @unknown default:
                    break
                }
                self?.receiveMessage()  // Continue listening
                
            case .failure(let error):
                print("Receive error: \(error)")
                self?.reconnect()
            }
        }
    }
    
    func disconnect() {
        webSocket?.cancel(with: .goingAway, reason: nil)
    }
    
    private func reconnect() {
        // Exponential backoff reconnection
        DispatchQueue.global().asyncAfter(deadline: .now() + 5) { [weak self] in
            self?.connect(to: URL(string: "wss://api.example.com/ws")!)
        }
    }
    
    // URLSessionWebSocketDelegate
    func urlSession(_ session: URLSession, webSocketTask: URLSessionWebSocketTask,
                    didOpenWithProtocol protocol: String?) {
        print("✅ WebSocket connected")
    }
    
    func urlSession(_ session: URLSession, webSocketTask: URLSessionWebSocketTask,
                    didCloseWith closeCode: URLSessionWebSocketTask.CloseCode, reason: Data?) {
        print("❌ WebSocket closed: \(closeCode)")
    }
}

// Real-world: Live chat
let ws = WebSocketManager()
ws.connect(to: URL(string: "wss://chat.example.com/ws")!)
ws.onMessage = { message in
    let chatMessage = try? JSONDecoder().decode(ChatMessage.self, from: Data(message.utf8))
    DispatchQueue.main.async {
        self.messages.append(chatMessage!)
    }
}
ws.send(message: "{\"type\":\"message\",\"text\":\"Hello!\"}")
```

---

## 30. SwiftData (iOS 17+)

### Q45: What is SwiftData and how does it compare to Core Data?

**Answer:**

| Feature | Core Data | SwiftData |
|---|---|---|
| API Style | Imperative, verbose | Declarative, Swift-native |
| Model definition | `.xcdatamodeld` + NSManagedObject | `@Model` macro on Swift class |
| Query | NSFetchRequest + NSPredicate | `@Query` macro + `#Predicate` |
| Context | NSManagedObjectContext | ModelContext |
| SwiftUI integration | `@FetchRequest` | `@Query` (simpler) |
| Minimum iOS | iOS 3+ | **iOS 17+** |
| Undo/Redo | ✅ Built-in | ✅ Built-in |

```swift
import SwiftData

// MODEL — just add @Model
@Model
class Task {
    var title: String
    var isCompleted: Bool
    var dueDate: Date?
    var priority: Int
    
    @Relationship(deleteRule: .cascade)
    var subtasks: [Subtask] = []
    
    init(title: String, isCompleted: Bool = false, dueDate: Date? = nil, priority: Int = 0) {
        self.title = title
        self.isCompleted = isCompleted
        self.dueDate = dueDate
        self.priority = priority
    }
}

// APP SETUP
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [Task.self, Subtask.self])
    }
}

// VIEW WITH @Query
struct TaskListView: View {
    @Query(filter: #Predicate<Task> { !$0.isCompleted },
           sort: \Task.priority, order: .reverse)
    var tasks: [Task]
    
    @Environment(\.modelContext) var context
    
    var body: some View {
        List(tasks) { task in
            Text(task.title)
        }
    }
    
    func addTask() {
        let task = Task(title: "New Task")
        context.insert(task)
        // No save() needed — auto-saves! ✅
    }
    
    func deleteTask(_ task: Task) {
        context.delete(task)
    }
}
```

---

## 31. Fastlane

### Q46: How do you use Fastlane for iOS CI/CD?

**Answer:**

```ruby
# Fastfile

default_platform(:ios)

platform :ios do
  
  # Run all tests
  lane :test do
    run_tests(
      scheme: "MyApp",
      devices: ["iPhone 15 Pro"],
      code_coverage: true
    )
  end
  
  # Build and upload to TestFlight
  lane :beta do
    # Auto-increment build number
    increment_build_number(
      build_number: latest_testflight_build_number + 1
    )
    
    # Code signing via match
    match(type: "appstore", readonly: true)
    
    # Build the app
    build_app(
      scheme: "MyApp",
      export_method: "app-store"
    )
    
    # Upload to TestFlight
    upload_to_testflight(
      skip_waiting_for_build_processing: true
    )
    
    # Notify team via Slack
    slack(
      message: "New build uploaded to TestFlight! 🚀",
      channel: "#ios-releases"
    )
  end
  
  # Submit to App Store
  lane :release do
    build_app(scheme: "MyApp", export_method: "app-store")
    upload_to_app_store(
      submit_for_review: true,
      automatic_release: false,  # manual release after approval
      phased_release: true       # gradual rollout
    )
  end
end
```

**`match` for code signing:**
```ruby
# Matchfile
git_url("https://github.com/team/certificates.git")
type("appstore")
app_identifier("com.company.myapp")
username("dev@company.com")

# match stores certificates & profiles in a private git repo
# All team members run `fastlane match appstore` to get the same certs
```

---

## 32. MVVM-C (Coordinator Pattern)

### Q47: How do you implement the Coordinator pattern with MVVM?

**Answer:**

```swift
// COORDINATOR PROTOCOL
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    func start()
}

// APP COORDINATOR — manages the whole app flow
class AppCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    private let window: UIWindow
    private let navigationController = UINavigationController()
    
    init(window: UIWindow) {
        self.window = window
    }
    
    func start() {
        window.rootViewController = navigationController
        window.makeKeyAndVisible()
        
        if AuthManager.shared.isLoggedIn {
            showHome()
        } else {
            showLogin()
        }
    }
    
    private func showLogin() {
        let loginCoordinator = LoginCoordinator(nav: navigationController)
        loginCoordinator.onLoginSuccess = { [weak self] in
            self?.childCoordinators.removeAll { $0 is LoginCoordinator }
            self?.showHome()
        }
        childCoordinators.append(loginCoordinator)
        loginCoordinator.start()
    }
    
    private func showHome() {
        let homeCoordinator = HomeCoordinator(nav: navigationController)
        childCoordinators.append(homeCoordinator)
        homeCoordinator.start()
    }
}

// LOGIN COORDINATOR
class LoginCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    var onLoginSuccess: (() -> Void)?
    private let nav: UINavigationController
    
    init(nav: UINavigationController) { self.nav = nav }
    
    func start() {
        let viewModel = LoginViewModel()
        viewModel.onLoginSuccess = { [weak self] in
            self?.onLoginSuccess?()  // Tell parent coordinator
        }
        viewModel.onForgotPassword = { [weak self] in
            self?.showForgotPassword()
        }
        let vc = LoginViewController(viewModel: viewModel)
        nav.setViewControllers([vc], animated: true)
    }
    
    private func showForgotPassword() {
        let vc = ForgotPasswordViewController()
        nav.pushViewController(vc, animated: true)
    }
}

// Benefits:
// ✅ ViewControllers don't know about navigation
// ✅ Easy to change navigation flow without touching VCs
// ✅ Each coordinator manages a specific flow (login, onboarding, checkout)
// ✅ Easily testable — mock the coordinator
```

---

## 33. MQTT for IoT

### Q48: What is MQTT and how do you use it in iOS?

**Answer:**

```
MQTT = Message Queuing Telemetry Transport

Publisher ──→ Broker ──→ Subscriber
  (IoT device)  (server)  (iOS app)

Topics: "home/livingroom/light/status"
        "home/bedroom/thermostat/temperature"
        
QoS Levels:
  0 = At most once (fire & forget)
  1 = At least once (acknowledged)
  2 = Exactly once (guaranteed, slowest)
```

```swift
import CocoaMQTT

class MQTTManager {
    private var mqtt: CocoaMQTT?
    var onMessage: ((String, String) -> Void)?  // (topic, message)
    
    func connect() {
        mqtt = CocoaMQTT(clientID: "ios_\(UUID().uuidString)", host: "mqtt.example.com", port: 1883)
        mqtt?.username = "user"
        mqtt?.password = "pass"
        mqtt?.keepAlive = 60
        mqtt?.delegate = self
        mqtt?.connect()
    }
    
    func subscribe(topic: String) {
        mqtt?.subscribe(topic, qos: .qos1)
    }
    
    func publish(topic: String, message: String) {
        mqtt?.publish(topic, withString: message, qos: .qos1, retained: false)
    }
}

extension MQTTManager: CocoaMQTTDelegate {
    func mqtt(_ mqtt: CocoaMQTT, didReceiveMessage message: CocoaMQTTMessage, id: UInt16) {
        if let payload = message.string {
            onMessage?(message.topic, payload)
        }
    }
    
    func mqtt(_ mqtt: CocoaMQTT, didConnectAck ack: CocoaMQTTConnAck) {
        if ack == .accept {
            subscribe(topic: "home/+/+/status")  // wildcard subscription
        }
    }
}

// Real-world: Smart home app
let mqtt = MQTTManager()
mqtt.connect()
mqtt.onMessage = { topic, message in
    // topic: "home/livingroom/light/status"
    // message: "{\"on\": true, \"brightness\": 80}"
    DispatchQueue.main.async {
        updateDeviceUI(topic: topic, status: message)
    }
}

// Turn on a light:
mqtt.publish(topic: "home/livingroom/light/command", message: "{\"on\": true}")
```

---

## 34. Smart Home Protocols

### Q49: What are the major smart home protocols?

**Answer:**

| Protocol | Range | Power | Speed | iOS Support |
|---|---|---|---|---|
| **Matter** | Local network | Low-Med | Fast | ✅ HomeKit + Matter |
| **HomeKit** | WiFi/BLE | Varies | Fast | ✅ Native Apple |
| **Zigbee** | 10-30m mesh | Very low | Medium | Via hub only |
| **Z-Wave** | 30m mesh | Very low | Medium | Via hub only |
| **Thread** | Mesh network | Low | Fast | ✅ Apple supports |
| **BLE** | 10-30m | Very low | Medium | ✅ CoreBluetooth |
| **WiFi** | Router range | High | Fast | ✅ NetworkExtension |
| **MQTT** | Internet | N/A | Fast | Via libraries |

```swift
// HomeKit Example — controlling a smart light
import HomeKit

class HomeKitManager: NSObject, HMHomeManagerDelegate {
    let homeManager = HMHomeManager()
    
    func turnOnLight(named name: String) {
        guard let home = homeManager.primaryHome,
              let accessory = home.accessories.first(where: { $0.name == name }),
              let lightService = accessory.services.first(where: { $0.serviceType == HMServiceTypeLightbulb }),
              let powerChar = lightService.characteristics.first(where: { 
                  $0.characteristicType == HMCharacteristicTypePowerState 
              }) else { return }
        
        powerChar.writeValue(true) { error in
            if let error { print("Error: \(error)") }
            else { print("✅ Light turned on") }
        }
    }
}
```

