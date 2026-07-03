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

