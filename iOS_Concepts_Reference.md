# iOS Development — Core Concepts Reference

> A practical reference for five foundational iOS topics: Navigation, Sensors, GPS, List/Table Views, and Core Data. Each section explains what it is, when to use it, key APIs, and ready-to-run code snippets.

---

## Table of Contents

1. [Navigation](#1-navigation)
2. [Sensors (CoreMotion)](#2-sensors-coremotion)
3. [GPS & Location (CoreLocation)](#3-gps--location-corelocation)
4. [List / Table View](#4-list--table-view)
5. [Core Data](#5-core-data)
6. [Where to Use Each Concept](#6-where-to-use-each-concept)
7. [Master Keyword Glossary](#7-master-keyword-glossary)

---

## 1. Navigation

### What it is
Navigation is how users move between screens in your app. iOS offers two main patterns:
- **Push navigation** — screens stack on top of each other; back button returns to previous screen.
- **Modal presentation** — a new screen slides up and covers the current one.

SwiftUI uses `NavigationStack` (iOS 16+) or the older `NavigationView`. UIKit uses `UINavigationController`.

### When to use it
Use push navigation for hierarchical flows (Home → Detail → Edit).
Use modal presentation for self-contained tasks (compose message, settings sheet).

### Key APIs

| API | Purpose |
|---|---|
| `NavigationStack` | Root container for push-navigation in SwiftUI (iOS 16+) |
| `NavigationLink` | Tappable element that pushes a destination view |
| `NavigationView` | Older SwiftUI navigation container (iOS 13–15) |
| `.navigationTitle(_:)` | Sets the title shown in the navigation bar |
| `.navigationBarTitleDisplayMode` | `.large` (big title) or `.inline` (small title) |
| `.sheet(isPresented:)` | Presents a view modally as a bottom sheet |
| `.fullScreenCover(isPresented:)` | Full-screen modal presentation |
| `@State var path` | Programmatic navigation — control the stack from code |
| `UINavigationController` | UIKit equivalent, manages a stack of `UIViewController`s |

### Code Snippets

#### Basic two-screen push navigation (SwiftUI)

```swift
import SwiftUI

// Screen 1 — wraps everything in NavigationStack
struct HomeView: View {
    var body: some View {
        NavigationStack {
            VStack {
                Text("Home Screen")
                    .font(.title)

                // NavigationLink pushes DetailView onto the stack
                NavigationLink("Go to Detail") {
                    DetailView(name: "Alice")
                }
                .buttonStyle(.borderedProminent)
            }
            .navigationTitle("Home")          // large title by default
        }
    }
}

// Screen 2 — receives data from Screen 1
struct DetailView: View {
    let name: String

    var body: some View {
        Text("Hello, \(name)!")
            .font(.largeTitle)
            .navigationTitle("Detail")
            .navigationBarTitleDisplayMode(.inline)  // small title
    }
}
```

#### Programmatic navigation using a state flag

```swift
struct HomeView: View {
    @State private var goToDetail = false
    @State private var result: String = ""

    var body: some View {
        NavigationStack {
            VStack {
                Button("Fetch and Navigate") {
                    result = "Data loaded"
                    goToDetail = true          // trigger navigation in code
                }

                NavigationLink(
                    destination: DetailView(name: result),
                    isActive: $goToDetail      // bound to the state flag
                ) { EmptyView() }              // invisible link
            }
            .navigationTitle("Home")
        }
    }
}
```

#### Modal sheet presentation

```swift
struct ParentView: View {
    @State private var showSheet = false

    var body: some View {
        Button("Open Settings") {
            showSheet = true
        }
        .sheet(isPresented: $showSheet) {
            SettingsView()                     // presented modally
        }
    }
}
```

#### UIKit — push with UINavigationController

```swift
// Inside a UIViewController
@IBAction func goNextTapped(_ sender: UIButton) {
    let next = DetailViewController()
    // Push onto the navigation stack (requires UINavigationController as root)
    navigationController?.pushViewController(next, animated: true)
}

// To go back programmatically
navigationController?.popViewController(animated: true)
```

**Keywords:** `NavigationStack` · `NavigationLink` · `NavigationView` · `isActive` · `navigationTitle` · `sheet(isPresented:)` · `UINavigationController` · `pushViewController` · `popViewController`

---

## 2. Sensors (CoreMotion)

### What it is
`CoreMotion` is Apple's framework for reading data from hardware sensors built into iPhone and Apple Watch:

| Sensor | Measures |
|---|---|
| Accelerometer | Linear acceleration (g-force) on X/Y/Z axes |
| Gyroscope | Rotation rate around X/Y/Z axes (radians/sec) |
| Magnetometer | Magnetic field strength — used for compass heading |
| Barometer | Atmospheric pressure and relative altitude |
| Pedometer | Step count, distance, pace, floors climbed |
| Device Motion | Fused attitude (roll/pitch/yaw), gravity, user acceleration |

### When to use it
- Count steps in a fitness app → **CMPedometer**
- Detect shaking, tilting, or gestures → **CMMotionManager (accelerometer)**
- Measure altitude gain on a hike → **CMAltimeter**
- Build an AR or compass feature → **CMDeviceMotion (attitude)**

### Key APIs

| API | Purpose |
|---|---|
| `CMMotionManager` | Central class — start/stop accelerometer, gyroscope, magnetometer, device motion |
| `CMAccelerometerData` | Struct with `.acceleration` (x, y, z) |
| `CMGyroData` | Struct with `.rotationRate` (x, y, z) |
| `CMDeviceMotion` | Fused data — attitude (roll, pitch, yaw), gravity, userAcceleration |
| `CMPedometer` | Step counter; also distance, pace, floors |
| `CMAltimeter` | Barometric pressure and relative altitude |
| `startAccelerometerUpdates(to:withHandler:)` | Stream accelerometer data on a given queue |
| `startDeviceMotionUpdates(to:withHandler:)` | Stream fused device-motion data |
| `CMPedometer.isStepCountingAvailable()` | Check hardware support before using |
| `NSMotionUsageDescription` | Required Info.plist key for motion permission |

### Code Snippets

#### Read accelerometer data in real time

```swift
import CoreMotion

class MotionManager: ObservableObject {

    private let motion = CMMotionManager()

    @Published var x: Double = 0
    @Published var y: Double = 0
    @Published var z: Double = 0

    func startAccelerometer() {
        guard motion.isAccelerometerAvailable else { return }

        motion.accelerometerUpdateInterval = 0.1   // 10 updates per second
        motion.startAccelerometerUpdates(to: .main) { [weak self] data, error in
            guard let data = data else { return }
            self?.x = data.acceleration.x   // in g (1 g = 9.8 m/s²)
            self?.y = data.acceleration.y
            self?.z = data.acceleration.z
        }
    }

    func stop() {
        motion.stopAccelerometerUpdates()
    }
}

// SwiftUI view using the manager
struct AccelView: View {
    @StateObject private var manager = MotionManager()

    var body: some View {
        VStack {
            Text("X: \(manager.x, specifier: "%.2f") g")
            Text("Y: \(manager.y, specifier: "%.2f") g")
            Text("Z: \(manager.z, specifier: "%.2f") g")
        }
        .onAppear { manager.startAccelerometer() }
        .onDisappear { manager.stop() }
    }
}
```

#### Count steps with CMPedometer

```swift
import CoreMotion

class PedometerManager: ObservableObject {

    private let pedometer = CMPedometer()
    @Published var steps: Int = 0

    func startCounting() {
        guard CMPedometer.isStepCountingAvailable() else { return }

        // Count steps from now onwards
        pedometer.startUpdates(from: Date()) { [weak self] data, error in
            guard let data = data else { return }
            DispatchQueue.main.async {
                self?.steps = data.numberOfSteps.intValue
            }
        }
    }

    func stop() {
        pedometer.stopUpdates()
    }
}
```

#### Read roll/pitch/yaw with DeviceMotion

```swift
import CoreMotion

let motion = CMMotionManager()
motion.deviceMotionUpdateInterval = 0.05

motion.startDeviceMotionUpdates(to: .main) { data, error in
    guard let attitude = data?.attitude else { return }
    print("Roll:  \(attitude.roll)  radians")   // rotation around front-to-back axis
    print("Pitch: \(attitude.pitch) radians")   // tilt forward/back
    print("Yaw:   \(attitude.yaw)   radians")   // rotation around vertical axis
}
```

**Info.plist key required:**
```xml
<key>NSMotionUsageDescription</key>
<string>This app uses motion sensors to count your steps and detect activity.</string>
```

**Keywords:** `CMMotionManager` · `CMPedometer` · `CMAltimeter` · `CMDeviceMotion` · `startAccelerometerUpdates` · `startDeviceMotionUpdates` · `isStepCountingAvailable` · `acceleration.x/y/z` · `attitude.roll/pitch/yaw` · `NSMotionUsageDescription`

---

## 3. GPS & Location (CoreLocation)

### What it is
`CoreLocation` gives your app access to the device's position using GPS, Wi-Fi triangulation, and cell towers. It also provides heading (compass), region monitoring, and geofencing.

### When to use it
- Show user's position on a map
- Record a running/hiking route
- Trigger an action when user enters/leaves a geographic area (geofence)
- Get a single position fix when a button is pressed

### Key APIs

| API | Purpose |
|---|---|
| `CLLocationManager` | Main class — request permission, start/stop location updates |
| `CLLocationManagerDelegate` | Protocol for receiving location and error callbacks |
| `requestWhenInUseAuthorization()` | Ask permission to use location while app is open |
| `requestAlwaysAuthorization()` | Ask permission to use location in background |
| `requestLocation()` | One-shot fix — best for on-demand use (calls delegate once) |
| `startUpdatingLocation()` | Continuous stream of location updates |
| `stopUpdatingLocation()` | Stop the continuous stream |
| `CLLocation` | Represents a position — holds coordinate, altitude, speed, course |
| `CLLocationCoordinate2D` | Struct with `.latitude` and `.longitude` (Double, degrees) |
| `CLLocationAccuracy` | Constants: `kCLLocationAccuracyBest`, `kCLLocationAccuracyHundredMeters` |
| `CLGeocoder` | Convert coordinates ↔ human-readable addresses (reverse geocoding) |
| `CLRegion` / `startMonitoring(for:)` | Geofencing — enter/exit region events |
| `NSLocationWhenInUseUsageDescription` | Required Info.plist key |
| `NSLocationAlwaysAndWhenInUseUsageDescription` | Required for background location |

### requestLocation() vs startUpdatingLocation()

| | `requestLocation()` | `startUpdatingLocation()` |
|---|---|---|
| Updates | One, then stops | Continuous stream |
| Battery | Efficient | Drains battery |
| Best for | On-demand button press | Live tracking, route recording |
| Delegate called | Once | Repeatedly |

### Code Snippets

#### LocationManager as ObservableObject (SwiftUI pattern)

```swift
import CoreLocation
import Foundation

class LocationManager: NSObject, ObservableObject, CLLocationManagerDelegate {

    private let manager = CLLocationManager()

    @Published var coordinate: CLLocationCoordinate2D? = nil
    @Published var altitude: Double? = nil
    @Published var speed: Double? = nil
    @Published var authorizationStatus: CLAuthorizationStatus = .notDetermined
    @Published var errorMessage: String? = nil

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()     // triggers permission popup
    }

    // MARK: - Public interface

    /// One-shot fix — call this when a button is tapped.
    func fetchOnce() {
        errorMessage = nil
        manager.requestLocation()
    }

    /// Continuous tracking — call this to start a live route recording.
    func startTracking() {
        errorMessage = nil
        manager.startUpdatingLocation()
    }

    func stopTracking() {
        manager.stopUpdatingLocation()
    }

    // MARK: - Delegate

    func locationManager(_ manager: CLLocationManager,
                         didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        coordinate = location.coordinate         // CLLocationCoordinate2D
        altitude   = location.altitude           // metres above sea level
        speed      = location.speed              // m/s (-1 if unavailable)
    }

    func locationManager(_ manager: CLLocationManager,
                         didFailWithError error: Error) {
        errorMessage = error.localizedDescription
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus
    }
}
```

#### Reverse geocoding — coordinate to street address

```swift
import CoreLocation

let geocoder = CLGeocoder()
let location = CLLocation(latitude: 43.70, longitude: -79.42)

geocoder.reverseGeocodeLocation(location) { placemarks, error in
    if let place = placemarks?.first {
        let address = [place.name,
                       place.locality,
                       place.administrativeArea,
                       place.country]
            .compactMap { $0 }
            .joined(separator: ", ")
        print(address)   // e.g. "150 King St W, Toronto, Ontario, Canada"
    }
}
```

#### Display location on a Map (SwiftUI + MapKit)

```swift
import SwiftUI
import MapKit

struct MapView: View {
    @StateObject private var locationManager = LocationManager()

    @State private var region = MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 43.70, longitude: -79.42),
        span: MKCoordinateSpan(latitudeDelta: 0.05, longitudeDelta: 0.05)
    )

    var body: some View {
        Map(coordinateRegion: $region, showsUserLocation: true)
            .ignoresSafeArea()
            .onChange(of: locationManager.coordinate) { coord in
                if let coord = coord {
                    region.center = coord      // pan map to user's position
                }
            }
            .onAppear { locationManager.startTracking() }
            .onDisappear { locationManager.stopTracking() }
    }
}
```

**Info.plist keys required:**
```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>This app uses your location to show your position on the map.</string>

<!-- Only if you need background location: -->
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>This app tracks your route in the background.</string>
```

**Keywords:** `CLLocationManager` · `CLLocationManagerDelegate` · `requestLocation` · `startUpdatingLocation` · `didUpdateLocations` · `CLLocationCoordinate2D` · `kCLLocationAccuracyBest` · `CLGeocoder` · `reverseGeocodeLocation` · `NSLocationWhenInUseUsageDescription` · `showsUserLocation` · `MKCoordinateRegion`

---

## 4. List / Table View

### What it is
A scrollable, vertically arranged list of rows — the most common UI pattern in iOS (think Settings, Contacts, Mail). In SwiftUI it's `List`; in UIKit it's `UITableView`.

### When to use it
- Show any collection of items: saved records, search results, history, menu options
- Allow swipe-to-delete, swipe actions, or reordering
- Group items under section headers

### Key APIs — SwiftUI List

| API | Purpose |
|---|---|
| `List` | SwiftUI list container — automatically styled and scrollable |
| `ForEach` | Iterates over a collection to produce rows |
| `Identifiable` | Protocol — give each item a unique `id` so List can track changes |
| `.listStyle(.plain)` / `.grouped` / `.insetGrouped` | Visual styles |
| `.onDelete(perform:)` | Enables swipe-to-delete with the red trash action |
| `.onMove(perform:)` | Enables drag-to-reorder rows |
| `.swipeActions` | Custom swipe actions (iOS 15+) |
| `Section(header:)` | Groups rows under a section header |
| `@State var items` | Mutable array that drives the list |
| `EditButton()` | Toggles edit mode (shows delete/reorder controls) |

### Key APIs — UIKit UITableView

| API | Purpose |
|---|---|
| `UITableView` | The table view itself |
| `UITableViewDataSource` | Protocol — provide number of rows and cell content |
| `UITableViewDelegate` | Protocol — respond to taps, configure row height |
| `UITableViewCell` | A single row; reuse with `dequeueReusableCell(withIdentifier:)` |
| `numberOfRowsInSection` | Tell the table how many rows to show |
| `cellForRowAt` | Configure and return a cell for a given row |
| `didSelectRowAt` | Handle row tap |
| `tableView.deleteRows(at:with:)` | Animate deletion of rows |

### Code Snippets

#### Simple SwiftUI List from an array

```swift
import SwiftUI

struct City: Identifiable {        // Identifiable provides the .id List needs
    let id = UUID()
    let name: String
    let country: String
}

struct CityListView: View {

    let cities: [City] = [
        City(name: "Toronto",  country: "Canada"),
        City(name: "London",   country: "UK"),
        City(name: "Tokyo",    country: "Japan"),
    ]

    var body: some View {
        NavigationStack {
            List(cities) { city in               // List takes any Identifiable collection
                VStack(alignment: .leading) {
                    Text(city.name).font(.headline)
                    Text(city.country).font(.subheadline).foregroundColor(.secondary)
                }
            }
            .navigationTitle("Cities")
            .listStyle(.insetGrouped)
        }
    }
}
```

#### List with add, delete, and navigation to detail

```swift
import SwiftUI

struct Item: Identifiable {
    let id = UUID()
    var title: String
}

struct ItemListView: View {

    @State private var items: [Item] = [
        Item(title: "First item"),
        Item(title: "Second item"),
    ]

    var body: some View {
        NavigationStack {
            List {
                ForEach(items) { item in
                    NavigationLink(item.title) {
                        DetailView(item: item)   // push detail on tap
                    }
                }
                .onDelete(perform: deleteItems)  // swipe-to-delete
                .onMove(perform: moveItems)      // drag-to-reorder
            }
            .navigationTitle("Items")
            .toolbar {
                ToolbarItem(placement: .navigationBarLeading) {
                    EditButton()                 // toggles delete/reorder UI
                }
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("Add") {
                        items.append(Item(title: "New item \(items.count + 1)"))
                    }
                }
            }
        }
    }

    private func deleteItems(at offsets: IndexSet) {
        items.remove(atOffsets: offsets)         // IndexSet tells us which rows to remove
    }

    private func moveItems(from source: IndexSet, to destination: Int) {
        items.move(fromOffsets: source, toOffset: destination)
    }
}

struct DetailView: View {
    let item: Item
    var body: some View {
        Text(item.title).font(.title)
            .navigationTitle("Detail")
    }
}
```

#### List with Section headers

```swift
struct GroupedListView: View {

    let fruits = ["Apple", "Banana", "Cherry"]
    let veggies = ["Carrot", "Broccoli", "Spinach"]

    var body: some View {
        List {
            Section(header: Text("Fruits")) {
                ForEach(fruits, id: \.self) { Text($0) }
            }
            Section(header: Text("Vegetables")) {
                ForEach(veggies, id: \.self) { Text($0) }
            }
        }
        .listStyle(.grouped)
    }
}
```

#### UIKit UITableView (programmatic)

```swift
import UIKit

class MyTableViewController: UITableViewController {

    let data = ["Row 1", "Row 2", "Row 3"]

    // MARK: - UITableViewDataSource

    override func tableView(_ tableView: UITableView,
                            numberOfRowsInSection section: Int) -> Int {
        return data.count
    }

    override func tableView(_ tableView: UITableView,
                            cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        // Reuse cells for performance — dequeue instead of allocating new ones
        let cell = tableView.dequeueReusableCell(withIdentifier: "cell",
                                                 for: indexPath)
        cell.textLabel?.text = data[indexPath.row]
        return cell
    }

    // MARK: - UITableViewDelegate

    override func tableView(_ tableView: UITableView,
                            didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)
        print("Tapped: \(data[indexPath.row])")
    }
}
```

**Keywords:** `List` · `ForEach` · `Identifiable` · `.onDelete` · `.onMove` · `IndexSet` · `Section` · `EditButton` · `UITableView` · `UITableViewDataSource` · `UITableViewDelegate` · `cellForRowAt` · `dequeueReusableCell` · `didSelectRowAt` · `listStyle`

---

## 5. Core Data

### What it is
Core Data is Apple's on-device object persistence framework. It acts as an ORM (Object Relational Mapper) on top of SQLite. You define a data model (entities with attributes and relationships), and Core Data handles saving, loading, querying, and migrating that data.

### When to use it
- Save user-generated data that must survive app restarts
- Store lists of records (journal entries, workouts, to-do items)
- Need complex queries, relationships, or filtering
- Want undo/redo support

### Key Concepts

| Concept | Meaning |
|---|---|
| **NSPersistentContainer** | Sets up the whole stack (model + store + context) in one call |
| **NSManagedObjectContext** | The "scratch pad" — create, edit, and delete objects here before saving |
| **NSManagedObject** | Base class for every Core Data entity |
| **NSFetchRequest** | Query — defines what entities and predicates to retrieve |
| **NSPredicate** | Filter condition for a fetch request (like SQL WHERE) |
| **NSSortDescriptor** | Defines sort order for fetch results |
| **`context.save()`** | Persists all pending changes to disk |
| **`@FetchRequest`** | SwiftUI property wrapper — automatically runs and updates on changes |
| **`@Environment(\.managedObjectContext)`** | Injects the context into a SwiftUI view |
| **.xcdatamodeld file** | Visual data model editor in Xcode — define entities here |

### Key APIs

| API | Purpose |
|---|---|
| `NSPersistentContainer(name:)` | Creates the stack; name must match the `.xcdatamodeld` file |
| `container.loadPersistentStores` | Opens or creates the SQLite store file |
| `container.viewContext` | The main-thread managed object context |
| `NSEntityDescription.insertNewObject(forEntityName:into:)` | Create a new managed object |
| `NSFetchRequest<EntityName>` | Generic fetch request for a specific entity |
| `context.fetch(_:)` | Execute a fetch request and return results |
| `context.delete(_:)` | Mark an object for deletion (call `save()` to commit) |
| `try context.save()` | Write all changes to the persistent store |
| `@FetchRequest(sortDescriptors:)` | SwiftUI property wrapper for live-updating queries |

### Code Snippets

#### Step 1 — Define entities in the .xcdatamodeld file

In Xcode, open `YourApp.xcdatamodeld`:
1. Click **Add Entity** → name it `Note`
2. Add attributes:
   - `id` → UUID
   - `title` → String
   - `body` → String
   - `createdAt` → Date

Xcode auto-generates `Note+CoreDataClass.swift` and `Note+CoreDataProperties.swift`.

#### Step 2 — Set up PersistenceController

```swift
import CoreData

struct PersistenceController {

    static let shared = PersistenceController()   // singleton

    let container: NSPersistentContainer

    init() {
        container = NSPersistentContainer(name: "YourApp")  // must match .xcdatamodeld name
        container.loadPersistentStores { _, error in
            if let error = error {
                fatalError("Core Data store failed to load: \(error)")
            }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
    }
}
```

#### Step 3 — Inject context into SwiftUI

```swift
// In YourApp.swift (@main)
import SwiftUI

@main
struct YourApp: App {

    let persistence = PersistenceController.shared

    var body: some Scene {
        WindowGroup {
            ContentView()
                // Make the context available to all child views via environment
                .environment(\.managedObjectContext, persistence.container.viewContext)
        }
    }
}
```

#### Step 4 — Create (insert) a new record

```swift
import CoreData
import SwiftUI

struct AddNoteView: View {

    @Environment(\.managedObjectContext) private var context  // injected context
    @State private var title = ""

    var body: some View {
        VStack {
            TextField("Title", text: $title)
                .textFieldStyle(.roundedBorder)

            Button("Save Note") {
                saveNote()
            }
            .buttonStyle(.borderedProminent)
        }
        .padding()
    }

    private func saveNote() {
        // 1. Create a new managed object in the context
        let note = Note(context: context)
        note.id        = UUID()
        note.title     = title
        note.body      = ""
        note.createdAt = Date()

        // 2. Persist to disk
        do {
            try context.save()
        } catch {
            print("Save failed: \(error)")
        }
    }
}
```

#### Step 5 — Fetch and display records with @FetchRequest

```swift
import SwiftUI
import CoreData

struct NoteListView: View {

    // @FetchRequest automatically re-runs when the store changes — live updating
    @FetchRequest(
        sortDescriptors: [NSSortDescriptor(keyPath: \Note.createdAt, ascending: false)],
        animation: .default
    ) private var notes: FetchedResults<Note>

    @Environment(\.managedObjectContext) private var context

    var body: some View {
        List {
            ForEach(notes) { note in
                VStack(alignment: .leading) {
                    Text(note.title ?? "Untitled").font(.headline)
                    Text(note.createdAt ?? Date(), style: .date).font(.caption)
                }
            }
            .onDelete(perform: deleteNotes)
        }
        .navigationTitle("Notes")
    }

    private func deleteNotes(at offsets: IndexSet) {
        offsets.map { notes[$0] }.forEach(context.delete)  // mark for deletion
        try? context.save()                                 // commit to disk
    }
}
```

#### Manual fetch with NSFetchRequest and NSPredicate

```swift
func fetchNotes(containing keyword: String,
                context: NSManagedObjectContext) -> [Note] {
    let request = NSFetchRequest<Note>(entityName: "Note")

    // NSPredicate — like SQL WHERE title CONTAINS[c] keyword ([c] = case insensitive)
    request.predicate = NSPredicate(format: "title CONTAINS[c] %@", keyword)

    // Sort results newest first
    request.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]

    do {
        return try context.fetch(request)
    } catch {
        print("Fetch failed: \(error)")
        return []
    }
}
```

#### Update an existing record

```swift
func updateNote(_ note: Note, newTitle: String, context: NSManagedObjectContext) {
    note.title = newTitle           // modify the managed object directly
    try? context.save()             // save commits the change to disk
}
```

**Keywords:** `NSPersistentContainer` · `NSManagedObjectContext` · `NSManagedObject` · `NSFetchRequest` · `NSPredicate` · `NSSortDescriptor` · `@FetchRequest` · `FetchedResults` · `@Environment(\.managedObjectContext)` · `context.save()` · `context.delete()` · `loadPersistentStores` · `.xcdatamodeld` · `viewContext` · `automaticallyMergesChangesFromParent`

---

## 6. Where to Use Each Concept

The table below maps each concept to typical app features so you know which tool to reach for.

| App Feature | Use These Concepts |
|---|---|
| Moving between screens | **Navigation** — `NavigationStack`, `NavigationLink`, `.sheet` |
| Show a list of saved records | **List/Table View** + **Core Data** (`@FetchRequest`) |
| Save data across app restarts | **Core Data** (`NSPersistentContainer`, `context.save()`) |
| Show the user's current position | **GPS** (`CLLocationManager.requestLocation()`) |
| Draw the user's route on a map | **GPS** (`startUpdatingLocation()`) + **MapKit** |
| Step counter / fitness tracker | **Sensors** (`CMPedometer`) |
| Detect device tilt or shake | **Sensors** (`CMMotionManager` accelerometer) |
| Measure altitude gain | **Sensors** (`CMAltimeter`) |
| Filter or search a list | **Core Data** `NSPredicate` or `List` + `.searchable` |
| Swipe to delete list items | **List/Table View** (`.onDelete` / `UITableView` editing) |
| Navigate after GPS fix arrives | **GPS** + **Navigation** (`.onChange` → `NavigationLink isActive`) |
| Settings / options screen | **Navigation** `.sheet` or push + **List** for options rows |
| Compose / add new item form | **Navigation** `.sheet` + **Core Data** insert |

---

## 7. Master Keyword Glossary

| Keyword | Category | Quick definition |
|---|---|---|
| `NavigationStack` | Navigation | SwiftUI push-navigation container (iOS 16+) |
| `NavigationLink` | Navigation | Tappable link that pushes a destination view |
| `NavigationView` | Navigation | Older SwiftUI navigation container (iOS 13–15) |
| `.sheet(isPresented:)` | Navigation | Present a view as a bottom modal sheet |
| `.fullScreenCover` | Navigation | Present a view full-screen modally |
| `pushViewController` | Navigation | UIKit — push a new screen onto the nav stack |
| `popViewController` | Navigation | UIKit — go back one screen |
| `CMMotionManager` | Sensors | Start/stop accelerometer, gyroscope, device motion |
| `CMPedometer` | Sensors | Step counting, distance, pace |
| `CMAltimeter` | Sensors | Barometric pressure and relative altitude |
| `CMDeviceMotion` | Sensors | Fused attitude: roll, pitch, yaw |
| `startAccelerometerUpdates` | Sensors | Begin streaming raw accelerometer data |
| `NSMotionUsageDescription` | Sensors | Info.plist key for motion permission |
| `CLLocationManager` | GPS | Manages location hardware |
| `requestWhenInUseAuthorization` | GPS | Shows permission popup |
| `requestLocation()` | GPS | One-shot GPS fix — fires delegate once |
| `startUpdatingLocation()` | GPS | Continuous GPS stream |
| `didUpdateLocations` | GPS | Delegate callback with new position |
| `CLLocationCoordinate2D` | GPS | Latitude + longitude struct |
| `CLGeocoder` | GPS | Coordinate ↔ address conversion |
| `NSLocationWhenInUseUsageDescription` | GPS | Info.plist key shown in permission popup |
| `List` | List/Table | SwiftUI scrollable list |
| `ForEach` | List/Table | Iterates collection to produce rows |
| `Identifiable` | List/Table | Protocol — gives each item a unique id |
| `.onDelete(perform:)` | List/Table | Swipe-to-delete modifier |
| `.onMove(perform:)` | List/Table | Drag-to-reorder modifier |
| `EditButton()` | List/Table | Toggles edit mode in a List |
| `Section` | List/Table | Groups rows under a header |
| `UITableView` | List/Table | UIKit table view class |
| `dequeueReusableCell` | List/Table | Recycle cells for performance |
| `cellForRowAt` | List/Table | UIKit — configure a cell for a given row |
| `NSPersistentContainer` | Core Data | Sets up the full Core Data stack |
| `NSManagedObjectContext` | Core Data | In-memory scratch pad for changes |
| `NSFetchRequest` | Core Data | Query object — what to retrieve |
| `NSPredicate` | Core Data | Filter condition (like SQL WHERE) |
| `NSSortDescriptor` | Core Data | Sort order for fetch results |
| `@FetchRequest` | Core Data | SwiftUI live-updating fetch |
| `FetchedResults` | Core Data | Type returned by `@FetchRequest` |
| `context.save()` | Core Data | Persist all pending changes to disk |
| `context.delete(_:)` | Core Data | Mark an object for deletion |
| `@Environment(\.managedObjectContext)` | Core Data | Inject context into a SwiftUI view |
| `.xcdatamodeld` | Core Data | Xcode data model editor file |
