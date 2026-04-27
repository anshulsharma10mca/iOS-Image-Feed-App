# Building an iOS Image Feed App with CoreData, Remote Sync & Swift Concurrency

---

## Table of Contents

1. [Project Overview & Architecture](#1-project-overview--architecture)
2. [Xcode Project Setup](#2-xcode-project-setup)
3. [CoreData Model Design](#3-coredata-model-design)
4. [Networking Layer](#4-networking-layer)
5. [CoreData Stack & Repository Layer](#5-coredata-stack--repository-layer)
6. [Background Sync Engine](#6-background-sync-engine)
7. [Feed Screen with Pagination](#7-feed-screen-with-pagination)
8. [Recent Selections Tab](#8-recent-selections-tab)
9. [Entry Screen](#9-entry-screen)
10. [Linked Image List Component](#10-linked-image-list-component)
11. [Image Picker from Feed](#11-image-picker-from-feed)
12. [App Entry Point & Tab Navigation](#12-app-entry-point--tab-navigation)
13. [Dependency Injection & Service Locator](#13-dependency-injection--service-locator)
14. [Error Handling & Edge Cases](#14-error-handling--edge-cases)
15. [Testing Strategy](#15-testing-strategy)

---

## 1. Project Overview & Architecture

### What We're Building

A photo-journaling iOS app where users:
- Browse a **paginated image feed** (15 images/page)
- **Select an image** and navigate to an **Entry screen**
- On the Entry screen, add **description**, **date**, **note**, and a **linked image list**
- Linked images are referenced by **image ID** and are tappable (navigates to that image's entry)
- Linked images can be **added** (by typing an ID) and **deleted**
- All data is **persisted locally in SQLite via CoreData**
- All entries are **uploaded to a remote server asynchronously**
- A **background sync engine** reconciles local and remote state
- A **Recent Selections** tab shows recently accessed images

### Architecture: MVVM + Repository Pattern

```
┌─────────────────────────────────────────────┐
│                   UI Layer                  │
│  FeedView  EntryView  RecentSelectionsView  │
└────────────────────┬────────────────────────┘
                     │ observes
┌────────────────────▼────────────────────────┐
│              ViewModel Layer                │
│  FeedViewModel  EntryViewModel              │
└────────────────────┬────────────────────────┘
                     │ calls
┌────────────────────▼────────────────────────┐
│             Repository Layer                │
│  ImageRepository  EntryRepository          │
└──────────┬─────────────────────┬────────────┘
           │                     │
┌──────────▼──────┐   ┌──────────▼──────────┐
│  CoreData Stack │   │   Network Service   │
│   (SQLite)      │   │   (Remote API)      │
└─────────────────┘   └─────────────────────┘
           ▲                     │
           └──── SyncEngine ─────┘
                (Background)
```

### Folder Structure

```
ImageFeedApp/
├── App/
│   ├── ImageFeedApp.swift
│   └── AppDependencies.swift
├── CoreData/
│   ├── ImageFeedApp.xcdatamodeld
│   ├── CoreDataStack.swift
│   └── NSManagedObject+Extensions/
│       ├── ImageEntity+CoreDataClass.swift
│       └── EntryEntity+CoreDataClass.swift
├── Models/
│   ├── ImageItem.swift
│   ├── Entry.swift
│   └── LinkedImage.swift
├── Networking/
│   ├── APIClient.swift
│   ├── APIEndpoints.swift
│   └── NetworkModels.swift
├── Repositories/
│   ├── ImageRepository.swift
│   └── EntryRepository.swift
├── Sync/
│   └── SyncEngine.swift
├── Features/
│   ├── Feed/
│   │   ├── FeedViewModel.swift
│   │   └── FeedView.swift
│   ├── Entry/
│   │   ├── EntryViewModel.swift
│   │   └── EntryView.swift
│   └── RecentSelections/
│       ├── RecentSelectionsViewModel.swift
│       └── RecentSelectionsView.swift
└── Shared/
    ├── ImageCardView.swift
    ├── LinkedImageRowView.swift
    └── Extensions/
```

---

## 2. Xcode Project Setup

### Step 1 — Create the Project

1. Open Xcode → **File → New → Project**
2. Select **iOS → App**
3. Set:
   - **Product Name**: `ImageFeedApp`
   - **Interface**: `SwiftUI`
   - **Language**: `Swift`
   - ✅ **Use Core Data**
   - ✅ **Include Tests**
4. Click **Next** and choose your save location.

### Step 2 — Add Swift Package Dependencies

Go to **File → Add Package Dependencies** and add:

```
https://github.com/onevcat/Kingfisher   (async image loading & caching)
```

In `Package.swift` (or via SPM UI), pin to version `≥ 7.0.0`.

### Step 3 — Configure Background Modes

In `Info.plist` (or via Signing & Capabilities), add:

- **Background Modes** → ✅ Background fetch
- **Background Modes** → ✅ Background processing

Add to `Info.plist`:
```xml
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.yourapp.sync</string>
</array>
```

---

## 3. CoreData Model Design

### Step 1 — Open the `.xcdatamodeld` File

In Xcode, click `ImageFeedApp.xcdatamodeld`. Add two entities.

### Entity 1: `ImageEntity`

| Attribute       | Type    | Notes                          |
|-----------------|---------|--------------------------------|
| `id`            | String  | Remote image ID (primary key)  |
| `imageURL`      | String  | Remote URL of the image        |
| `thumbnailURL`  | String  | Thumbnail URL                  |
| `accessedAt`    | Date    | Last accessed timestamp        |
| `createdAt`     | Date    | When first seen                |

**Relationship**: `entry` → to-one → `EntryEntity` (optional, inverse: `image`)

### Entity 2: `EntryEntity`

| Attribute          | Type    | Notes                              |
|--------------------|---------|------------------------------------|
| `imageID`          | String  | Foreign key to ImageEntity         |
| `entryDescription` | String  | User's description text            |
| `note`             | String  | User's note                        |
| `entryDate`        | Date    | User-selected date                 |
| `linkedImageIDs`   | String  | JSON-encoded `[String]`            |
| `isSynced`         | Boolean | False until confirmed by server    |
| `updatedAt`        | Date    | Last local modification timestamp  |
| `serverID`         | String  | Optional, set after server save    |

**Relationship**: `image` → to-one → `ImageEntity` (inverse: `entry`)

### Step 2 — Generate NSManagedObject Subclasses

Select each entity → Editor menu → **Create NSManagedObject Subclass**.
Set **Codegen** to `Manual/None` for full control.

### `ImageEntity+CoreDataClass.swift`

```swift
import CoreData

@objc(ImageEntity)
public class ImageEntity: NSManagedObject {

    @NSManaged public var id: String
    @NSManaged public var imageURL: String
    @NSManaged public var thumbnailURL: String
    @NSManaged public var accessedAt: Date?
    @NSManaged public var createdAt: Date
    @NSManaged public var entry: EntryEntity?

    // Convert to domain model
    func toDomain() -> ImageItem {
        ImageItem(
            id: id,
            imageURL: URL(string: imageURL)!,
            thumbnailURL: URL(string: thumbnailURL)!,
            accessedAt: accessedAt,
            createdAt: createdAt
        )
    }
}

extension ImageEntity {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<ImageEntity> {
        NSFetchRequest<ImageEntity>(entityName: "ImageEntity")
    }
}
```

### `EntryEntity+CoreDataClass.swift`

```swift
import CoreData

@objc(EntryEntity)
public class EntryEntity: NSManagedObject {

    @NSManaged public var imageID: String
    @NSManaged public var entryDescription: String
    @NSManaged public var note: String
    @NSManaged public var entryDate: Date
    @NSManaged public var linkedImageIDsJSON: String  // stored as JSON
    @NSManaged public var isSynced: Bool
    @NSManaged public var updatedAt: Date
    @NSManaged public var serverID: String?
    @NSManaged public var image: ImageEntity?

    // Decode/encode linked image IDs
    var linkedImageIDs: [String] {
        get {
            (try? JSONDecoder().decode([String].self,
              from: linkedImageIDsJSON.data(using: .utf8) ?? Data())) ?? []
        }
        set {
            linkedImageIDsJSON = (try? String(
                data: JSONEncoder().encode(newValue), encoding: .utf8)) ?? "[]"
        }
    }

    func toDomain() -> Entry {
        Entry(
            imageID: imageID,
            description: entryDescription,
            note: note,
            date: entryDate,
            linkedImageIDs: linkedImageIDs,
            isSynced: isSynced,
            updatedAt: updatedAt,
            serverID: serverID
        )
    }
}

extension EntryEntity {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<EntryEntity> {
        NSFetchRequest<EntryEntity>(entityName: "EntryEntity")
    }
}
```

---

## 4. Networking Layer

### `Models/ImageItem.swift`

```swift
import Foundation

struct ImageItem: Identifiable, Hashable {
    let id: String
    let imageURL: URL
    let thumbnailURL: URL
    var accessedAt: Date?
    let createdAt: Date
}

struct Entry: Identifiable {
    var id: String { imageID }
    let imageID: String
    var description: String
    var note: String
    var date: Date
    var linkedImageIDs: [String]
    var isSynced: Bool
    var updatedAt: Date
    var serverID: String?
}
```

### `Networking/APIEndpoints.swift`

```swift
import Foundation

enum APIEndpoints {
    static let baseURL = URL(string: "https://api.yourserver.com/v1")!

    static func feed(page: Int, pageSize: Int = 15) -> URL {
        baseURL
            .appendingPathComponent("images")
            .appending(queryItems: [
                URLQueryItem(name: "page", value: "\(page)"),
                URLQueryItem(name: "pageSize", value: "\(pageSize)")
            ])
    }

    static func image(id: String) -> URL {
        baseURL.appendingPathComponent("images/\(id)")
    }

    static var entries: URL {
        baseURL.appendingPathComponent("entries")
    }

    static func entry(id: String) -> URL {
        baseURL.appendingPathComponent("entries/\(id)")
    }
}

// Helper for query items on URL
private extension URL {
    func appending(queryItems: [URLQueryItem]) -> URL {
        var components = URLComponents(url: self, resolvingAgainstBaseURL: false)!
        components.queryItems = (components.queryItems ?? []) + queryItems
        return components.url!
    }
}
```

### `Networking/NetworkModels.swift`

```swift
import Foundation

// MARK: - Remote DTOs

struct RemoteImageDTO: Codable {
    let id: String
    let imageURL: String
    let thumbnailURL: String
    let createdAt: String          // ISO8601

    enum CodingKeys: String, CodingKey {
        case id
        case imageURL     = "image_url"
        case thumbnailURL = "thumbnail_url"
        case createdAt    = "created_at"
    }
}

struct RemoteFeedResponse: Codable {
    let images: [RemoteImageDTO]
    let page: Int
    let pageSize: Int
    let totalCount: Int
    let hasNextPage: Bool
}

struct RemoteEntryDTO: Codable {
    let id: String?
    let imageID: String
    let description: String
    let note: String
    let date: String               // ISO8601
    let linkedImageIDs: [String]
    let updatedAt: String

    enum CodingKeys: String, CodingKey {
        case id
        case imageID       = "image_id"
        case description
        case note
        case date
        case linkedImageIDs = "linked_image_ids"
        case updatedAt     = "updated_at"
    }
}

struct RemoteEntryResponse: Codable {
    let entry: RemoteEntryDTO
}
```

### `Networking/APIClient.swift`

```swift
import Foundation

// MARK: - Errors

enum APIError: Error, LocalizedError {
    case invalidResponse
    case httpError(statusCode: Int, data: Data)
    case decodingError(Error)
    case networkError(Error)

    var errorDescription: String? {
        switch self {
        case .invalidResponse:         return "Invalid server response."
        case .httpError(let code, _):  return "HTTP error \(code)."
        case .decodingError(let e):    return "Decoding failed: \(e.localizedDescription)"
        case .networkError(let e):     return "Network error: \(e.localizedDescription)"
        }
    }
}

// MARK: - Protocol for testability

protocol APIClientProtocol {
    func fetchFeed(page: Int) async throws -> RemoteFeedResponse
    func fetchImage(id: String) async throws -> RemoteImageDTO
    func uploadEntry(_ entry: Entry) async throws -> RemoteEntryDTO
    func updateEntry(_ entry: Entry) async throws -> RemoteEntryDTO
}

// MARK: - Implementation

final class APIClient: APIClientProtocol {

    private let session: URLSession
    private let decoder: JSONDecoder = {
        let d = JSONDecoder()
        d.dateDecodingStrategy = .iso8601
        return d
    }()

    init(session: URLSession = .shared) {
        self.session = session
    }

    // MARK: Feed

    func fetchFeed(page: Int) async throws -> RemoteFeedResponse {
        let url = APIEndpoints.feed(page: page)
        return try await get(url: url)
    }

    // MARK: Single Image

    func fetchImage(id: String) async throws -> RemoteImageDTO {
        let url = APIEndpoints.image(id: id)
        return try await get(url: url)
    }

    // MARK: Entries

    func uploadEntry(_ entry: Entry) async throws -> RemoteEntryDTO {
        let payload = makeEntryPayload(from: entry)
        return try await post(url: APIEndpoints.entries, body: payload)
    }

    func updateEntry(_ entry: Entry) async throws -> RemoteEntryDTO {
        guard let serverID = entry.serverID else {
            return try await uploadEntry(entry)
        }
        let payload = makeEntryPayload(from: entry)
        return try await put(url: APIEndpoints.entry(id: serverID), body: payload)
    }

    // MARK: - Helpers

    private func makeEntryPayload(from entry: Entry) -> RemoteEntryDTO {
        let formatter = ISO8601DateFormatter()
        return RemoteEntryDTO(
            id: entry.serverID,
            imageID: entry.imageID,
            description: entry.description,
            note: entry.note,
            date: formatter.string(from: entry.date),
            linkedImageIDs: entry.linkedImageIDs,
            updatedAt: formatter.string(from: entry.updatedAt)
        )
    }

    private func get<T: Decodable>(url: URL) async throws -> T {
        var request = URLRequest(url: url)
        request.httpMethod = "GET"
        request.setValue("Bearer \(AuthStore.shared.token)", forHTTPHeaderField: "Authorization")
        return try await perform(request)
    }

    private func post<T: Decodable>(url: URL, body: some Encodable) async throws -> T {
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(AuthStore.shared.token)", forHTTPHeaderField: "Authorization")
        request.httpBody = try JSONEncoder().encode(body)
        return try await perform(request)
    }

    private func put<T: Decodable>(url: URL, body: some Encodable) async throws -> T {
        var request = URLRequest(url: url)
        request.httpMethod = "PUT"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(AuthStore.shared.token)", forHTTPHeaderField: "Authorization")
        request.httpBody = try JSONEncoder().encode(body)
        return try await perform(request)
    }

    private func perform<T: Decodable>(_ request: URLRequest) async throws -> T {
        let (data, response) = try await session.data(for: request)
        guard let http = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }
        guard (200...299).contains(http.statusCode) else {
            throw APIError.httpError(statusCode: http.statusCode, data: data)
        }
        do {
            return try decoder.decode(T.self, from: data)
        } catch {
            throw APIError.decodingError(error)
        }
    }
}

// MARK: - Auth Store (replace with your real implementation)

final class AuthStore {
    static let shared = AuthStore()
    var token: String = "your-auth-token"
    private init() {}
}
```

---

## 5. CoreData Stack & Repository Layer

### `CoreData/CoreDataStack.swift`

```swift
import CoreData
import Combine

final class CoreDataStack {

    static let shared = CoreDataStack()

    // The persistent container — backed by SQLite
    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "ImageFeedApp")
        container.loadPersistentStores { _, error in
            if let error { fatalError("CoreData load failed: \(error)") }
        }
        // Merge changes from background contexts automatically
        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
        return container
    }()

    var viewContext: NSManagedObjectContext {
        persistentContainer.viewContext
    }

    /// Creates a background context for heavy reads/writes
    func newBackgroundContext() -> NSManagedObjectContext {
        let ctx = persistentContainer.newBackgroundContext()
        ctx.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
        return ctx
    }

    /// Saves a context safely — logs errors instead of crashing
    func save(context: NSManagedObjectContext) {
        guard context.hasChanges else { return }
        do {
            try context.save()
        } catch {
            print("CoreData save error: \(error)")
        }
    }

    private init() {}
}
```

### `Repositories/ImageRepository.swift`

```swift
import CoreData
import Foundation

protocol ImageRepositoryProtocol {
    func saveImages(_ images: [RemoteImageDTO]) async
    func fetchCachedImages(page: Int, pageSize: Int) async -> [ImageItem]
    func fetchRecentSelections(limit: Int) async -> [ImageItem]
    func markAccessed(imageID: String) async
    func fetchImage(id: String) async -> ImageItem?
}

final class ImageRepository: ImageRepositoryProtocol {

    private let stack: CoreDataStack

    init(stack: CoreDataStack = .shared) {
        self.stack = stack
    }

    // MARK: - Save remote images locally

    func saveImages(_ images: [RemoteImageDTO]) async {
        let context = stack.newBackgroundContext()
        await context.perform {
            let formatter = ISO8601DateFormatter()
            for dto in images {
                // Upsert: fetch existing or create new
                let request = ImageEntity.fetchRequest()
                request.predicate = NSPredicate(format: "id == %@", dto.id)
                request.fetchLimit = 1
                let existing = (try? context.fetch(request))?.first
                let entity = existing ?? ImageEntity(context: context)
                entity.id           = dto.id
                entity.imageURL     = dto.imageURL
                entity.thumbnailURL = dto.thumbnailURL
                entity.createdAt    = formatter.date(from: dto.createdAt) ?? Date()
            }
            self.stack.save(context: context)
        }
    }

    // MARK: - Fetch cached images for feed (paginated)

    func fetchCachedImages(page: Int, pageSize: Int) async -> [ImageItem] {
        let context = stack.newBackgroundContext()
        return await context.perform {
            let request = ImageEntity.fetchRequest()
            request.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
            request.fetchLimit  = pageSize
            request.fetchOffset = (page - 1) * pageSize
            let entities = (try? context.fetch(request)) ?? []
            return entities.map { $0.toDomain() }
        }
    }

    // MARK: - Recent selections

    func fetchRecentSelections(limit: Int = 50) async -> [ImageItem] {
        let context = stack.newBackgroundContext()
        return await context.perform {
            let request = ImageEntity.fetchRequest()
            request.predicate = NSPredicate(format: "accessedAt != nil")
            request.sortDescriptors = [NSSortDescriptor(key: "accessedAt", ascending: false)]
            request.fetchLimit = limit
            let entities = (try? context.fetch(request)) ?? []
            return entities.map { $0.toDomain() }
        }
    }

    // MARK: - Mark as accessed (updates timestamp)

    func markAccessed(imageID: String) async {
        let context = stack.newBackgroundContext()
        await context.perform {
            let request = ImageEntity.fetchRequest()
            request.predicate = NSPredicate(format: "id == %@", imageID)
            request.fetchLimit = 1
            if let entity = (try? context.fetch(request))?.first {
                entity.accessedAt = Date()
                self.stack.save(context: context)
            }
        }
    }

    // MARK: - Fetch single image by ID

    func fetchImage(id: String) async -> ImageItem? {
        let context = stack.newBackgroundContext()
        return await context.perform {
            let request = ImageEntity.fetchRequest()
            request.predicate = NSPredicate(format: "id == %@", id)
            request.fetchLimit = 1
            return (try? context.fetch(request))?.first?.toDomain()
        }
    }
}
```

### `Repositories/EntryRepository.swift`

```swift
import CoreData
import Foundation

protocol EntryRepositoryProtocol {
    func saveEntry(_ entry: Entry) async
    func fetchEntry(imageID: String) async -> Entry?
    func fetchUnsyncedEntries() async -> [Entry]
    func markSynced(imageID: String, serverID: String) async
}

final class EntryRepository: EntryRepositoryProtocol {

    private let stack: CoreDataStack

    init(stack: CoreDataStack = .shared) {
        self.stack = stack
    }

    // MARK: - Save or update an entry

    func saveEntry(_ entry: Entry) async {
        let context = stack.newBackgroundContext()
        await context.perform {
            let request = EntryEntity.fetchRequest()
            request.predicate = NSPredicate(format: "imageID == %@", entry.imageID)
            request.fetchLimit = 1
            let existing = (try? context.fetch(request))?.first
            let entity = existing ?? EntryEntity(context: context)

            entity.imageID            = entry.imageID
            entity.entryDescription   = entry.description
            entity.note               = entry.note
            entity.entryDate          = entry.date
            entity.linkedImageIDs     = entry.linkedImageIDs  // uses computed property
            entity.isSynced           = false                 // mark dirty
            entity.updatedAt          = Date()
            if let serverID = entry.serverID {
                entity.serverID = serverID
            }
            self.stack.save(context: context)
        }
    }

    // MARK: - Fetch entry for a given image ID

    func fetchEntry(imageID: String) async -> Entry? {
        let context = stack.newBackgroundContext()
        return await context.perform {
            let request = EntryEntity.fetchRequest()
            request.predicate = NSPredicate(format: "imageID == %@", imageID)
            request.fetchLimit = 1
            return (try? context.fetch(request))?.first?.toDomain()
        }
    }

    // MARK: - Fetch all unsynced entries (for background sync)

    func fetchUnsyncedEntries() async -> [Entry] {
        let context = stack.newBackgroundContext()
        return await context.perform {
            let request = EntryEntity.fetchRequest()
            request.predicate = NSPredicate(format: "isSynced == NO")
            request.sortDescriptors = [NSSortDescriptor(key: "updatedAt", ascending: true)]
            return ((try? context.fetch(request)) ?? []).map { $0.toDomain() }
        }
    }

    // MARK: - Mark entry as synced after server confirms

    func markSynced(imageID: String, serverID: String) async {
        let context = stack.newBackgroundContext()
        await context.perform {
            let request = EntryEntity.fetchRequest()
            request.predicate = NSPredicate(format: "imageID == %@", imageID)
            request.fetchLimit = 1
            if let entity = (try? context.fetch(request))?.first {
                entity.isSynced  = true
                entity.serverID  = serverID
                self.stack.save(context: context)
            }
        }
    }
}
```

---

## 6. Background Sync Engine

### `Sync/SyncEngine.swift`

```swift
import Foundation
import BackgroundTasks

// MARK: - Sync Engine

@MainActor
final class SyncEngine: ObservableObject {

    @Published private(set) var isSyncing = false
    @Published private(set) var lastSyncDate: Date?
    @Published private(set) var pendingCount: Int = 0

    private let entryRepository: EntryRepositoryProtocol
    private let apiClient: APIClientProtocol

    static let backgroundTaskID = "com.yourapp.sync"

    init(
        entryRepository: EntryRepositoryProtocol,
        apiClient: APIClientProtocol
    ) {
        self.entryRepository = entryRepository
        self.apiClient       = apiClient
    }

    // MARK: - Register background task

    static func registerBackgroundTask() {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: backgroundTaskID,
            using: nil
        ) { task in
            guard let processingTask = task as? BGProcessingTask else { return }
            Task {
                // Create a temporary sync engine for background use
                let engine = SyncEngine(
                    entryRepository: EntryRepository(),
                    apiClient: APIClient()
                )
                await engine.syncNow()
                processingTask.setTaskCompleted(success: true)
                await SyncEngine.scheduleNextBackgroundSync()
            }

            processingTask.expirationHandler = {
                task.setTaskCompleted(success: false)
            }
        }
    }

    // MARK: - Schedule next background sync

    static func scheduleNextBackgroundSync() async {
        let request = BGProcessingTaskRequest(identifier: backgroundTaskID)
        request.requiresNetworkConnectivity = true
        request.requiresExternalPower       = false
        request.earliestBeginDate           = Date(timeIntervalSinceNow: 15 * 60) // 15 min

        try? BGTaskScheduler.shared.submit(request)
    }

    // MARK: - Main sync method

    func syncNow() async {
        guard !isSyncing else { return }
        isSyncing = true
        defer { isSyncing = false }

        let unsynced = await entryRepository.fetchUnsyncedEntries()
        pendingCount = unsynced.count

        await withTaskGroup(of: Void.self) { group in
            for entry in unsynced {
                group.addTask {
                    await self.syncEntry(entry)
                }
            }
        }

        await MainActor.run {
            self.lastSyncDate = Date()
            self.pendingCount = 0
        }
    }

    // MARK: - Sync a single entry with retry

    private func syncEntry(_ entry: Entry, retryCount: Int = 3) async {
        for attempt in 1...retryCount {
            do {
                let remote: RemoteEntryDTO
                if entry.serverID != nil {
                    remote = try await apiClient.updateEntry(entry)
                } else {
                    remote = try await apiClient.uploadEntry(entry)
                }
                await entryRepository.markSynced(
                    imageID: entry.imageID,
                    serverID: remote.id ?? entry.imageID
                )
                return
            } catch {
                if attempt == retryCount {
                    print("Sync failed permanently for imageID \(entry.imageID): \(error)")
                } else {
                    // Exponential backoff
                    try? await Task.sleep(nanoseconds: UInt64(pow(2.0, Double(attempt))) * 1_000_000_000)
                }
            }
        }
    }
}
```

### Register the Sync Engine in App Startup

In your app's `@main` struct (shown fully in Section 12), add:

```swift
init() {
    SyncEngine.registerBackgroundTask()
    Task {
        await SyncEngine.scheduleNextBackgroundSync()
    }
}
```

---

## 7. Feed Screen with Pagination

### `Features/Feed/FeedViewModel.swift`

```swift
import Foundation
import Combine

@MainActor
final class FeedViewModel: ObservableObject {

    // MARK: - State

    @Published private(set) var images: [ImageItem] = []
    @Published private(set) var isLoading = false
    @Published private(set) var isLoadingMore = false
    @Published private(set) var errorMessage: String?
    @Published private(set) var hasNextPage = true

    private let pageSize = 15
    private var currentPage = 1
    private var isFetching = false

    // MARK: - Dependencies

    private let apiClient: APIClientProtocol
    private let imageRepository: ImageRepositoryProtocol

    init(
        apiClient: APIClientProtocol,
        imageRepository: ImageRepositoryProtocol
    ) {
        self.apiClient       = apiClient
        self.imageRepository = imageRepository
    }

    // MARK: - Load first page

    func loadInitialFeed() async {
        guard !isFetching else { return }
        isLoading = true
        currentPage = 1
        hasNextPage = true
        await fetchPage(page: 1, isRefresh: true)
        isLoading = false
    }

    // MARK: - Load next page (called as user scrolls near bottom)

    func loadNextPageIfNeeded(currentItem: ImageItem) async {
        guard hasNextPage, !isFetching else { return }
        // Trigger when within last 5 items
        guard let index = images.firstIndex(where: { $0.id == currentItem.id }),
              index >= images.count - 5 else { return }

        isLoadingMore = true
        currentPage += 1
        await fetchPage(page: currentPage, isRefresh: false)
        isLoadingMore = false
    }

    // MARK: - Pull to refresh

    func refresh() async {
        await loadInitialFeed()
    }

    // MARK: - Core fetch logic

    private func fetchPage(page: Int, isRefresh: Bool) async {
        isFetching = true
        defer { isFetching = false }

        do {
            let response = try await apiClient.fetchFeed(page: page)
            await imageRepository.saveImages(response.images)

            let newItems = response.images.compactMap { dto -> ImageItem? in
                guard let imageURL = URL(string: dto.imageURL),
                      let thumbURL = URL(string: dto.thumbnailURL) else { return nil }
                return ImageItem(
                    id: dto.id,
                    imageURL: imageURL,
                    thumbnailURL: thumbURL,
                    accessedAt: nil,
                    createdAt: ISO8601DateFormatter().date(from: dto.createdAt) ?? Date()
                )
            }

            if isRefresh {
                images = newItems
            } else {
                // Avoid duplicates
                let existingIDs = Set(images.map { $0.id })
                images += newItems.filter { !existingIDs.contains($0.id) }
            }

            hasNextPage = response.hasNextPage
            errorMessage = nil

        } catch {
            // On network error, fall back to local cache
            errorMessage = error.localizedDescription
            if isRefresh {
                images = await imageRepository.fetchCachedImages(
                    page: page,
                    pageSize: pageSize
                )
            }
        }
    }
}
```

### `Features/Feed/FeedView.swift`

```swift
import SwiftUI
import Kingfisher

struct FeedView: View {

    @StateObject private var viewModel: FeedViewModel
    @State private var selectedImage: ImageItem?

    // Injected via environment or init
    private let entryRepository: EntryRepositoryProtocol
    private let apiClient: APIClientProtocol
    private let imageRepository: ImageRepositoryProtocol

    init(dependencies: AppDependencies) {
        _viewModel = StateObject(wrappedValue: FeedViewModel(
            apiClient: dependencies.apiClient,
            imageRepository: dependencies.imageRepository
        ))
        self.entryRepository  = dependencies.entryRepository
        self.apiClient        = dependencies.apiClient
        self.imageRepository  = dependencies.imageRepository
    }

    // MARK: - 2-column grid layout

    private let columns = [
        GridItem(.flexible(), spacing: 2),
        GridItem(.flexible(), spacing: 2)
    ]

    var body: some View {
        NavigationStack {
            Group {
                if viewModel.isLoading {
                    ProgressView("Loading feed…")
                        .frame(maxWidth: .infinity, maxHeight: .infinity)
                } else {
                    ScrollView {
                        LazyVGrid(columns: columns, spacing: 2) {
                            ForEach(viewModel.images) { image in
                                ImageThumbnailCell(image: image)
                                    .onTapGesture {
                                        selectedImage = image
                                    }
                                    .task {
                                        await viewModel.loadNextPageIfNeeded(currentItem: image)
                                    }
                            }
                        }

                        if viewModel.isLoadingMore {
                            ProgressView()
                                .padding()
                        }
                    }
                    .refreshable {
                        await viewModel.refresh()
                    }
                }
            }
            .navigationTitle("Feed")
            .alert("Error", isPresented: .constant(viewModel.errorMessage != nil)) {
                Button("OK") {}
            } message: {
                Text(viewModel.errorMessage ?? "")
            }
            .navigationDestination(item: $selectedImage) { image in
                EntryView(
                    imageItem: image,
                    dependencies: AppDependencies.shared
                )
            }
        }
        .task {
            await viewModel.loadInitialFeed()
        }
    }
}

// MARK: - Thumbnail Cell

struct ImageThumbnailCell: View {
    let image: ImageItem

    var body: some View {
        KFImage(image.thumbnailURL)
            .placeholder {
                Color.gray.opacity(0.3)
            }
            .resizable()
            .scaledToFill()
            .frame(
                width: (UIScreen.main.bounds.width / 2) - 2,
                height: (UIScreen.main.bounds.width / 2) - 2
            )
            .clipped()
            .contentShape(Rectangle())
    }
}
```

---

## 8. Recent Selections Tab

### `Features/RecentSelections/RecentSelectionsViewModel.swift`

```swift
import Foundation

@MainActor
final class RecentSelectionsViewModel: ObservableObject {

    @Published private(set) var recentImages: [ImageItem] = []
    @Published private(set) var isLoading = false

    private let imageRepository: ImageRepositoryProtocol

    init(imageRepository: ImageRepositoryProtocol) {
        self.imageRepository = imageRepository
    }

    func load() async {
        isLoading = true
        recentImages = await imageRepository.fetchRecentSelections(limit: 100)
        isLoading = false
    }

    func refresh() async {
        await load()
    }
}
```

### `Features/RecentSelections/RecentSelectionsView.swift`

```swift
import SwiftUI
import Kingfisher

struct RecentSelectionsView: View {

    @StateObject private var viewModel: RecentSelectionsViewModel
    @State private var selectedImage: ImageItem?

    private let dependencies: AppDependencies

    init(dependencies: AppDependencies) {
        _viewModel = StateObject(wrappedValue: RecentSelectionsViewModel(
            imageRepository: dependencies.imageRepository
        ))
        self.dependencies = dependencies
    }

    var body: some View {
        NavigationStack {
            Group {
                if viewModel.isLoading {
                    ProgressView()
                        .frame(maxWidth: .infinity, maxHeight: .infinity)
                } else if viewModel.recentImages.isEmpty {
                    ContentUnavailableView(
                        "No Recent Selections",
                        systemImage: "photo.on.rectangle.angled",
                        description: Text("Images you've opened will appear here.")
                    )
                } else {
                    List(viewModel.recentImages) { image in
                        RecentImageRow(image: image)
                            .contentShape(Rectangle())
                            .onTapGesture {
                                selectedImage = image
                            }
                            .listRowInsets(EdgeInsets(top: 4, leading: 0, bottom: 4, trailing: 16))
                    }
                    .listStyle(.plain)
                    .refreshable {
                        await viewModel.refresh()
                    }
                }
            }
            .navigationTitle("Recent")
            .navigationDestination(item: $selectedImage) { image in
                EntryView(imageItem: image, dependencies: dependencies)
            }
        }
        .task {
            await viewModel.load()
        }
    }
}

// MARK: - Row

struct RecentImageRow: View {
    let image: ImageItem

    var body: some View {
        HStack(spacing: 12) {
            KFImage(image.thumbnailURL)
                .resizable()
                .scaledToFill()
                .frame(width: 64, height: 64)
                .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(image.id)
                    .font(.headline)
                    .lineLimit(1)
                if let accessedAt = image.accessedAt {
                    Text("Viewed \(accessedAt, style: .relative) ago")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }

            Spacer()
        }
        .padding(.leading, 16)
    }
}
```

---

## 9. Entry Screen

### `Features/Entry/EntryViewModel.swift`

```swift
import Foundation
import Combine

@MainActor
final class EntryViewModel: ObservableObject {

    // MARK: - State

    let imageItem: ImageItem

    @Published var description: String = ""
    @Published var note: String = ""
    @Published var date: Date = Date()
    @Published var linkedImageIDs: [String] = []

    @Published private(set) var isSaving = false
    @Published private(set) var isSynced = false
    @Published var errorMessage: String?

    // For adding a new linked image ID
    @Published var newLinkedID: String = ""
    @Published var showAddLinkedIDSheet = false

    // MARK: - Dependencies

    private let entryRepository: EntryRepositoryProtocol
    private let apiClient: APIClientProtocol
    private let imageRepository: ImageRepositoryProtocol

    init(
        imageItem: ImageItem,
        entryRepository: EntryRepositoryProtocol,
        apiClient: APIClientProtocol,
        imageRepository: ImageRepositoryProtocol
    ) {
        self.imageItem        = imageItem
        self.entryRepository  = entryRepository
        self.apiClient        = apiClient
        self.imageRepository  = imageRepository
    }

    // MARK: - Load existing entry

    func loadEntry() async {
        // Mark image as recently accessed
        await imageRepository.markAccessed(imageID: imageItem.id)

        if let existing = await entryRepository.fetchEntry(imageID: imageItem.id) {
            description    = existing.description
            note           = existing.note
            date           = existing.date
            linkedImageIDs = existing.linkedImageIDs
            isSynced       = existing.isSynced
        }
    }

    // MARK: - Save locally + trigger async upload

    func saveEntry() async {
        isSaving = true
        defer { isSaving = false }

        let entry = Entry(
            imageID: imageItem.id,
            description: description,
            note: note,
            date: date,
            linkedImageIDs: linkedImageIDs,
            isSynced: false,
            updatedAt: Date(),
            serverID: nil
        )

        // Save locally first (fast, offline-safe)
        await entryRepository.saveEntry(entry)

        // Upload asynchronously — don't await, let it happen in background
        Task.detached(priority: .background) {
            do {
                let remote: RemoteEntryDTO
                if entry.serverID != nil {
                    remote = try await self.apiClient.updateEntry(entry)
                } else {
                    remote = try await self.apiClient.uploadEntry(entry)
                }
                await self.entryRepository.markSynced(
                    imageID: entry.imageID,
                    serverID: remote.id ?? entry.imageID
                )
                await MainActor.run { self.isSynced = true }
            } catch {
                // Will be picked up by background sync engine later
                print("Immediate upload failed, will retry via SyncEngine: \(error)")
            }
        }
    }

    // MARK: - Linked Image Management

    func addLinkedImageID() {
        let trimmed = newLinkedID.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !trimmed.isEmpty, !linkedImageIDs.contains(trimmed) else { return }
        linkedImageIDs.append(trimmed)
        newLinkedID = ""
    }

    func removeLinkedImageIDs(at offsets: IndexSet) {
        linkedImageIDs.remove(atOffsets: offsets)
    }
}
```

### `Features/Entry/EntryView.swift`

```swift
import SwiftUI
import Kingfisher

struct EntryView: View {

    @StateObject private var viewModel: EntryViewModel
    @State private var linkedImageToOpen: ImageItem?

    private let dependencies: AppDependencies

    init(imageItem: ImageItem, dependencies: AppDependencies) {
        _viewModel = StateObject(wrappedValue: EntryViewModel(
            imageItem: imageItem,
            entryRepository: dependencies.entryRepository,
            apiClient: dependencies.apiClient,
            imageRepository: dependencies.imageRepository
        ))
        self.dependencies = dependencies
    }

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 20) {

                // MARK: Hero Image
                KFImage(viewModel.imageItem.imageURL)
                    .resizable()
                    .scaledToFill()
                    .frame(maxWidth: .infinity)
                    .frame(height: 280)
                    .clipped()
                    .overlay(alignment: .bottomLeading) {
                        Text("ID: \(viewModel.imageItem.id)")
                            .font(.caption)
                            .padding(6)
                            .background(.ultraThinMaterial)
                            .clipShape(RoundedRectangle(cornerRadius: 6))
                            .padding(8)
                    }

                Group {
                    // MARK: Description
                    SectionHeader(title: "Description")
                    TextField("Add a description…", text: $viewModel.description, axis: .vertical)
                        .lineLimit(3...6)
                        .textFieldStyle(.roundedBorder)

                    // MARK: Date
                    SectionHeader(title: "Date")
                    DatePicker("", selection: $viewModel.date, displayedComponents: .date)
                        .datePickerStyle(.compact)
                        .labelsHidden()

                    // MARK: Note
                    SectionHeader(title: "Note")
                    TextField("Add a note…", text: $viewModel.note, axis: .vertical)
                        .lineLimit(3...8)
                        .textFieldStyle(.roundedBorder)
                }
                .padding(.horizontal)

                // MARK: Linked Images
                LinkedImageListSection(
                    linkedImageIDs: viewModel.linkedImageIDs,
                    newLinkedID: $viewModel.newLinkedID,
                    onAdd: { viewModel.addLinkedImageID() },
                    onDelete: { offsets in viewModel.removeLinkedImageIDs(at: offsets) },
                    onTapItem: { id in
                        Task {
                            if let item = await dependencies.imageRepository.fetchImage(id: id) {
                                linkedImageToOpen = item
                            }
                        }
                    }
                )

                // MARK: Sync Status
                HStack {
                    Image(systemName: viewModel.isSynced ? "checkmark.icloud" : "icloud.slash")
                        .foregroundStyle(viewModel.isSynced ? .green : .orange)
                    Text(viewModel.isSynced ? "Synced" : "Pending sync")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
                .padding(.horizontal)
                .padding(.bottom, 8)
            }
        }
        .navigationTitle("Entry")
        .navigationBarTitleDisplayMode(.inline)
        .toolbar {
            ToolbarItem(placement: .primaryAction) {
                if viewModel.isSaving {
                    ProgressView()
                } else {
                    Button("Save") {
                        Task { await viewModel.saveEntry() }
                    }
                    .bold()
                }
            }
        }
        .navigationDestination(item: $linkedImageToOpen) { item in
            EntryView(imageItem: item, dependencies: dependencies)
        }
        .task {
            await viewModel.loadEntry()
        }
    }
}

// MARK: - Section Header

struct SectionHeader: View {
    let title: String
    var body: some View {
        Text(title)
            .font(.headline)
            .foregroundStyle(.primary)
    }
}
```

---

## 10. Linked Image List Component

```swift
import SwiftUI

struct LinkedImageListSection: View {

    let linkedImageIDs: [String]
    @Binding var newLinkedID: String

    let onAdd: () -> Void
    let onDelete: (IndexSet) -> Void
    let onTapItem: (String) -> Void

    var body: some View {
        VStack(alignment: .leading, spacing: 0) {
            HStack {
                Text("Linked Images")
                    .font(.headline)
                    .padding(.horizontal)
                Spacer()
                Text("\(linkedImageIDs.count)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .padding(.horizontal)
            }
            .padding(.vertical, 8)

            // Add new linked ID row
            HStack(spacing: 8) {
                TextField("Enter image ID…", text: $newLinkedID)
                    .textFieldStyle(.roundedBorder)
                    .autocorrectionDisabled()
                    .textInputAutocapitalization(.never)
                    .onSubmit { onAdd() }

                Button {
                    onAdd()
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.title2)
                        .foregroundStyle(.blue)
                }
                .disabled(newLinkedID.trimmingCharacters(in: .whitespaces).isEmpty)
            }
            .padding(.horizontal)
            .padding(.bottom, 8)

            if linkedImageIDs.isEmpty {
                Text("No linked images yet.")
                    .font(.caption)
                    .foregroundStyle(.tertiary)
                    .padding(.horizontal)
                    .padding(.bottom, 8)
            } else {
                // Swipe-to-delete list
                ForEach(Array(linkedImageIDs.enumerated()), id: \.offset) { index, id in
                    LinkedImageRow(imageID: id, onTap: { onTapItem(id) })
                        .swipeActions(edge: .trailing, allowsFullSwipe: true) {
                            Button(role: .destructive) {
                                onDelete(IndexSet(integer: index))
                            } label: {
                                Label("Delete", systemImage: "trash")
                            }
                        }

                    if index < linkedImageIDs.count - 1 {
                        Divider().padding(.leading, 60)
                    }
                }
            }
        }
        .background(Color(.secondarySystemBackground))
        .clipShape(RoundedRectangle(cornerRadius: 12))
        .padding(.horizontal)
    }
}

// MARK: - Individual linked image row

struct LinkedImageRow: View {
    let imageID: String
    let onTap: () -> Void

    var body: some View {
        Button(action: onTap) {
            HStack(spacing: 12) {
                Image(systemName: "photo.fill")
                    .font(.title3)
                    .foregroundStyle(.blue)
                    .frame(width: 36, height: 36)
                    .background(Color.blue.opacity(0.1))
                    .clipShape(RoundedRectangle(cornerRadius: 8))

                Text(imageID)
                    .font(.system(.body, design: .monospaced))
                    .foregroundStyle(.primary)
                    .lineLimit(1)

                Spacer()

                Image(systemName: "chevron.right")
                    .font(.caption)
                    .foregroundStyle(.tertiary)
            }
            .padding(.horizontal)
            .padding(.vertical, 10)
        }
        .buttonStyle(.plain)
    }
}
```

---

## 11. Image Picker from Feed

The Feed IS the image picker — tapping any image in the feed navigates to its Entry screen. However, if you need a dedicated modal sheet picker (e.g. from within the Entry screen), here's the component:

```swift
import SwiftUI
import Kingfisher

// Present this as a .sheet() when you want the user to pick an image
struct ImagePickerSheet: View {

    @Environment(\.dismiss) private var dismiss
    let onSelect: (ImageItem) -> Void
    private let dependencies: AppDependencies

    @StateObject private var viewModel: FeedViewModel

    init(dependencies: AppDependencies, onSelect: @escaping (ImageItem) -> Void) {
        self.dependencies = dependencies
        self.onSelect = onSelect
        _viewModel = StateObject(wrappedValue: FeedViewModel(
            apiClient: dependencies.apiClient,
            imageRepository: dependencies.imageRepository
        ))
    }

    private let columns = [
        GridItem(.flexible(), spacing: 2),
        GridItem(.flexible(), spacing: 2),
        GridItem(.flexible(), spacing: 2)
    ]

    var body: some View {
        NavigationStack {
            ScrollView {
                LazyVGrid(columns: columns, spacing: 2) {
                    ForEach(viewModel.images) { image in
                        KFImage(image.thumbnailURL)
                            .resizable()
                            .scaledToFill()
                            .frame(
                                width: UIScreen.main.bounds.width / 3 - 2,
                                height: UIScreen.main.bounds.width / 3 - 2
                            )
                            .clipped()
                            .contentShape(Rectangle())
                            .onTapGesture {
                                onSelect(image)
                                dismiss()
                            }
                            .task {
                                await viewModel.loadNextPageIfNeeded(currentItem: image)
                            }
                    }
                }
                if viewModel.isLoadingMore {
                    ProgressView().padding()
                }
            }
            .navigationTitle("Select Image")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") { dismiss() }
                }
            }
        }
        .task {
            await viewModel.loadInitialFeed()
        }
    }
}
```

---

## 12. App Entry Point & Tab Navigation

### `App/AppDependencies.swift`

```swift
import Foundation

// Single place to create and share all dependencies
final class AppDependencies {

    static let shared = AppDependencies()

    let apiClient: APIClientProtocol
    let imageRepository: ImageRepositoryProtocol
    let entryRepository: EntryRepositoryProtocol
    let syncEngine: SyncEngine

    init(
        apiClient: APIClientProtocol = APIClient(),
        stack: CoreDataStack = .shared
    ) {
        self.apiClient        = apiClient
        self.imageRepository  = ImageRepository(stack: stack)
        self.entryRepository  = EntryRepository(stack: stack)
        self.syncEngine       = SyncEngine(
            entryRepository: EntryRepository(stack: stack),
            apiClient: apiClient
        )
    }
}
```

### `App/ImageFeedApp.swift`

```swift
import SwiftUI
import BackgroundTasks

@main
struct ImageFeedApp: App {

    private let dependencies = AppDependencies.shared

    init() {
        // Register background sync task
        SyncEngine.registerBackgroundTask()
        // Schedule the first background sync
        Task {
            await SyncEngine.scheduleNextBackgroundSync()
        }
    }

    var body: some Scene {
        WindowGroup {
            MainTabView(dependencies: dependencies)
                .task {
                    // Trigger sync on app launch
                    await dependencies.syncEngine.syncNow()
                }
        }
    }
}

// MARK: - Main Tab View

struct MainTabView: View {

    let dependencies: AppDependencies
    @StateObject private var syncEngine: SyncEngine

    init(dependencies: AppDependencies) {
        self.dependencies = dependencies
        _syncEngine = StateObject(wrappedValue: dependencies.syncEngine)
    }

    var body: some View {
        TabView {
            // Tab 1: Feed
            FeedView(dependencies: dependencies)
                .tabItem {
                    Label("Feed", systemImage: "photo.stack")
                }

            // Tab 2: Recent Selections
            RecentSelectionsView(dependencies: dependencies)
                .tabItem {
                    Label("Recent", systemImage: "clock.arrow.circlepath")
                }
        }
        // Show pending sync badge on Recent tab
        .overlay(alignment: .bottomTrailing) {
            if syncEngine.pendingCount > 0 {
                SyncStatusBanner(count: syncEngine.pendingCount, isSyncing: syncEngine.isSyncing)
                    .padding(.bottom, 60)
                    .padding(.trailing, 16)
            }
        }
    }
}

// MARK: - Sync Status Banner

struct SyncStatusBanner: View {
    let count: Int
    let isSyncing: Bool

    var body: some View {
        HStack(spacing: 6) {
            if isSyncing {
                ProgressView().scaleEffect(0.7)
            } else {
                Image(systemName: "arrow.triangle.2.circlepath")
                    .font(.caption)
            }
            Text(isSyncing ? "Syncing…" : "\(count) pending")
                .font(.caption2)
                .bold()
        }
        .foregroundStyle(.white)
        .padding(.horizontal, 10)
        .padding(.vertical, 6)
        .background(.blue, in: Capsule())
        .shadow(radius: 4)
    }
}
```

---

## 13. Dependency Injection & Service Locator

The `AppDependencies` class (Section 12) is the service locator. All ViewModels receive their dependencies through initializers — making them fully testable.

For **SwiftUI previews**, create a mock dependencies object:

```swift
// For Previews & Tests
final class MockAPIClient: APIClientProtocol {
    func fetchFeed(page: Int) async throws -> RemoteFeedResponse {
        RemoteFeedResponse(
            images: (1...15).map { i in
                RemoteImageDTO(
                    id: "img-\(i)",
                    imageURL: "https://picsum.photos/seed/\(i)/800/800",
                    thumbnailURL: "https://picsum.photos/seed/\(i)/200/200",
                    createdAt: ISO8601DateFormatter().string(from: Date())
                )
            },
            page: page, pageSize: 15, totalCount: 60, hasNextPage: page < 4
        )
    }

    func fetchImage(id: String) async throws -> RemoteImageDTO {
        RemoteImageDTO(id: id,
                       imageURL: "https://picsum.photos/seed/\(id)/800/800",
                       thumbnailURL: "https://picsum.photos/seed/\(id)/200/200",
                       createdAt: ISO8601DateFormatter().string(from: Date()))
    }

    func uploadEntry(_ entry: Entry) async throws -> RemoteEntryDTO {
        RemoteEntryDTO(id: UUID().uuidString, imageID: entry.imageID,
                       description: entry.description, note: entry.note,
                       date: ISO8601DateFormatter().string(from: entry.date),
                       linkedImageIDs: entry.linkedImageIDs,
                       updatedAt: ISO8601DateFormatter().string(from: Date()))
    }

    func updateEntry(_ entry: Entry) async throws -> RemoteEntryDTO {
        try await uploadEntry(entry)
    }
}

// Usage in previews:
#Preview {
    FeedView(dependencies: AppDependencies(apiClient: MockAPIClient()))
}
```

---

## 14. Error Handling & Edge Cases

### Offline Support

The `FeedViewModel.fetchPage` already falls back to local CoreData cache when network fails. The `EntryViewModel.saveEntry` always saves locally first before attempting upload.

### Preventing Duplicate Linked IDs

In `EntryViewModel.addLinkedImageID`:
```swift
guard !trimmed.isEmpty, !linkedImageIDs.contains(trimmed) else { return }
```

### Handling Unknown Image IDs in Linked List

When a user taps a linked image ID, fetch it from the remote if not cached:

```swift
// In EntryView, extend the onTapItem handler:
onTapItem: { id in
    Task {
        // Try local first
        if let item = await dependencies.imageRepository.fetchImage(id: id) {
            linkedImageToOpen = item
        } else {
            // Fetch from remote and cache
            do {
                let dto = try await dependencies.apiClient.fetchImage(id: id)
                await dependencies.imageRepository.saveImages([dto])
                if let item = await dependencies.imageRepository.fetchImage(id: id) {
                    linkedImageToOpen = item
                }
            } catch {
                // Show error: "Image ID not found"
            }
        }
    }
}
```

### Sync Conflict Resolution

The server is the source of truth for synced data. The `mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy` ensures the most recent write wins locally. For server conflicts, implement a `updatedAt`-based last-write-wins strategy in your API.

---

## 15. Testing Strategy

### Unit Test: FeedViewModel

```swift
import XCTest
@testable import ImageFeedApp

@MainActor
final class FeedViewModelTests: XCTestCase {

    func testLoadInitialFeedPopulatesImages() async {
        let mockAPI  = MockAPIClient()
        let mockRepo = MockImageRepository()
        let vm       = FeedViewModel(apiClient: mockAPI, imageRepository: mockRepo)

        await vm.loadInitialFeed()

        XCTAssertEqual(vm.images.count, 15)
        XCTAssertFalse(vm.isLoading)
    }

    func testPaginationTriggersOnLastFiveItems() async {
        let mockAPI  = MockAPIClient()
        let mockRepo = MockImageRepository()
        let vm       = FeedViewModel(apiClient: mockAPI, imageRepository: mockRepo)

        await vm.loadInitialFeed()

        // Trigger pagination from the 12th item (15 - 3 = within last 5)
        let triggerItem = vm.images[11]
        await vm.loadNextPageIfNeeded(currentItem: triggerItem)

        XCTAssertEqual(vm.images.count, 30)
    }
}
```

### Unit Test: EntryViewModel - Linked Image IDs

```swift
@MainActor
final class EntryViewModelTests: XCTestCase {

    func testAddLinkedImageIDAppends() async {
        let vm = makeViewModel()
        vm.newLinkedID = "img-123"
        vm.addLinkedImageID()
        XCTAssertEqual(vm.linkedImageIDs, ["img-123"])
        XCTAssertEqual(vm.newLinkedID, "")
    }

    func testDuplicateLinkedIDNotAdded() async {
        let vm = makeViewModel()
        vm.newLinkedID = "img-123"
        vm.addLinkedImageID()
        vm.newLinkedID = "img-123"
        vm.addLinkedImageID()
        XCTAssertEqual(vm.linkedImageIDs.count, 1)
    }

    func testRemoveLinkedImageID() async {
        let vm = makeViewModel()
        vm.newLinkedID = "img-1"; vm.addLinkedImageID()
        vm.newLinkedID = "img-2"; vm.addLinkedImageID()
        vm.removeLinkedImageIDs(at: IndexSet(integer: 0))
        XCTAssertEqual(vm.linkedImageIDs, ["img-2"])
    }

    private func makeViewModel() -> EntryViewModel {
        EntryViewModel(
            imageItem: ImageItem(id: "test", imageURL: URL(string: "https://a.com")!,
                                 thumbnailURL: URL(string: "https://a.com")!,
                                 accessedAt: nil, createdAt: Date()),
            entryRepository: MockEntryRepository(),
            apiClient: MockAPIClient(),
            imageRepository: MockImageRepository()
        )
    }
}
```

### Unit Test: SyncEngine

```swift
@MainActor
final class SyncEngineTests: XCTestCase {

    func testSyncNowCallsUploadForUnsyncedEntries() async {
        let mockRepo = MockEntryRepository()
        let mockAPI  = MockAPIClient()
        let engine   = SyncEngine(entryRepository: mockRepo, apiClient: mockAPI)

        // Pre-populate unsynced entry
        mockRepo.stubbedUnsynced = [
            Entry(imageID: "img-1", description: "test", note: "",
                  date: Date(), linkedImageIDs: [], isSynced: false,
                  updatedAt: Date(), serverID: nil)
        ]

        await engine.syncNow()

        XCTAssertTrue(mockRepo.markedSyncedIDs.contains("img-1"))
        XCTAssertEqual(engine.pendingCount, 0)
    }
}
```

---

## Summary

Here's the complete data and navigation flow in one glance:

```
App Launch
    ↓
MainTabView (2 tabs)
    ├── [Tab 1] FeedView
    │       ↓ paginated grid (15/page)
    │       ↓ tap image
    │       └── EntryView
    │               ↓ hero image + fields
    │               ↓ linked image row tap
    │               └── EntryView (recursive navigation)
    │
    └── [Tab 2] RecentSelectionsView
            ↓ list of accessed images
            ↓ tap image
            └── EntryView

Data Flow:
    Save → CoreData (immediate) → isSynced = false
    Background / launch sync → APIClient.uploadEntry/updateEntry → isSynced = true
    Load → CoreData (offline cache) + remote (live data merged in)
```

This architecture gives you offline-first reliability, clean separation of concerns, and full Swift Concurrency throughout. Every layer is independently testable via protocol abstractions, and the sync engine handles retries with exponential backoff automatically.

