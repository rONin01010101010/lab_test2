# TrailTracker — iOS App

> A production-style iOS app that tracks outdoor trails using GPS, reads device sensors, and persists data with Core Data. Built with Swift and SwiftUI.

---

## Table of Contents

1. [What We're Building](#what-were-building)
2. [Prerequisites](#prerequisites)
3. [Project Setup](#project-setup)
4. [Architecture & File Structure](#architecture--file-structure)
5. [Part 1 — Navigation](#part-1--navigation)
6. [Part 2 — Sensors](#part-2--sensors)
7. [Part 3 — GPS & Location](#part-3--gps--location)
8. [Part 4 — List / Table View](#part-4--list--table-view)
9. [Part 5 — Core Data Persistence](#part-5--core-data-persistence)
10. [Putting It All Together](#putting-it-all-together)
11. [Build & Run](#build--run)
12. [Keyword Reference](#keyword-reference)

---

## What We're Building

TrailTracker is a real iOS application that lets users:

- **Record GPS trails** with live map visualization
- **Track motion data** — steps, pace, acceleration, rotation rate
- **Browse and manage** saved trails in a searchable list
- **Persist everything** locally using Core Data so data survives app restarts

By the end of this guide you will have a fully functional app you can run on a real device or the simulator. Every section is self-contained — skip to the part you need.

**Keywords:** `SwiftUI app tutorial` · `iOS trail tracker` · `GPS route recorder` · `CoreMotion pedometer` · `Core Data iOS 16` · `MVVM Swift`

---

## Prerequisites

| Requirement | Version |
|---|---|
| macOS | 13.0 Ventura or later |
| Xcode | 15.0 or later |
| Swift | 5.9 or later |
| iOS Deployment Target | iOS 16.0 or later |
| Apple Developer Account | Free (Simulator) · Paid (physical device) |

Install Xcode from the Mac App Store. Open it, accept the license agreement, and install additional components if prompted.

**Keywords:** `Xcode setup` · `Swift 5.9` · `iOS 16 minimum deployment` · `Apple Developer account` · `Simulator vs device`

---

## Project Setup

### Step 1 — Create the Xcode Project

1. Open **Xcode**.
2. Click **Create New Project** (`File > New > Project`).
3. Select **iOS** tab → **App** → **Next**.
4. Fill in project details:
   - **Product Name:** `TrailTracker`
   - **Team:** Your Apple ID / Team
   - **Organization Identifier:** `com.yourname`
   - **Interface:** `SwiftUI`
   - **Language:** `Swift`
   - **Storage:** `Core Data` ← check this
   - **Include Tests:** optional
5. Click **Next**, pick a save location, click **Create**.

> Checking "Core Data" at creation auto-generates `Persistence.swift` and wires the managed object context into `@main`. You can add it manually later, but this saves boilerplate.

### Step 2 — Configure Info.plist Permissions

Open `Info.plist` (or the target's **Info** tab in Xcode 14+) and add:

```xml
<!-- Location permissions -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>TrailTracker uses your location to record GPS paths.</string>

<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>TrailTracker uses your location in the background to continue tracking your route.</string>

<!-- Motion/sensor permissions -->
<key>NSMotionUsageDescription</key>
<string>TrailTracker uses motion sensors to count steps and detect activity.</string>
```

### Step 3 — Add Required Frameworks

In **Target > General > Frameworks, Libraries, and Embedded Content**, confirm these are present (modern Xcode includes them automatically):

- `CoreData.framework`
- `CoreLocation.framework`
- `CoreMotion.framework`
- `MapKit.framework`

**Keywords:** `Info.plist permissions` · `NSLocationWhenInUseUsageDescription` · `NSMotionUsageDescription` · `Xcode frameworks` · `CoreLocation setup` · `CoreMotion setup`

---

## Architecture & File Structure

TrailTracker uses **MVVM** (Model-View-ViewModel). Views are passive — they observe ViewModels. ViewModels own Services. Services own system APIs.

```
TrailTracker/
├── TrailTrackerApp.swift              # @main entry point, Core Data stack injection
├── ContentView.swift                  # Root view — hosts NavigationStack
│
├── Navigation/
│   └── AppTabView.swift               # Tab bar root (TabView + NavigationStack)
│
├── Models/
│   ├── TrailTracker.xcdatamodeld      # Core Data schema
│   ├── Trail+CoreDataClass.swift      # NSManagedObject subclass
│   ├── Trail+CoreDataProperties.swift
│   └── TrailPoint+CoreDataClass.swift
│
├── ViewModels/
│   ├── TrailListViewModel.swift       # Drives list view — fetch, delete, filter
│   ├── TrailDetailViewModel.swift     # Drives detail view — map rendering
│   └── SensorViewModel.swift         # Drives sensors view — formatted readings
│
├── Views/
│   ├── List/
│   │   ├── TrailListView.swift        # Searchable list of saved trails
│   │   └── TrailRowView.swift         # Row subview
│   ├── Detail/
│   │   ├── TrailDetailView.swift      # Detail screen for one trail
│   │   └── TrailMapView.swift         # MapKit polyline subview
│   ├── Sensors/
│   │   └── SensorsView.swift          # Accelerometer, gyroscope, pedometer
│   ├── GPS/
│   │   └── LiveTrackingView.swift     # Real-time GPS tracking + live map
│   └── Shared/
│       └── StatBadgeView.swift        # Reusable stat badge component
│
├── Services/
│   ├── LocationService.swift          # CLLocationManager wrapper
│   ├── MotionService.swift            # CMMotionManager + CMPedometer wrapper
│   └── PersistenceController.swift    # Core Data stack (singleton)
│
└── Extensions/
    ├── Color+Extensions.swift
    └── Date+Extensions.swift
```

**Design decisions:**
- Services are `ObservableObject` singletons injected into ViewModels — keeps system APIs isolated and easy to mock.
- ViewModels own the business logic and publish formatted data — Views contain zero formatting code.
- Core Data context flows down the view hierarchy via `.environment(\.managedObjectContext, ...)`.

**Keywords:** `MVVM architecture iOS` · `SwiftUI MVVM` · `ObservableObject` · `service layer Swift` · `Core Data environment` · `separation of concerns`

---

## Part 1 — Navigation

iOS navigation in SwiftUI is built around `NavigationStack` for push/pop flows and `TabView` for top-level sections. TrailTracker uses both: a `TabView` root with three tabs, each containing its own `NavigationStack`.

### 1.1 — Tab Bar Root

**File:** `Navigation/AppTabView.swift`

```swift
import SwiftUI

struct AppTabView: View {
    var body: some View {
        TabView {
            NavigationStack {
                TrailListView()
                    .navigationTitle("My Trails")
            }
            .tabItem { Label("Trails", systemImage: "list.bullet") }

            NavigationStack {
                LiveTrackingView()
                    .navigationTitle("Track")
            }
            .tabItem { Label("Track", systemImage: "location.fill") }

            NavigationStack {
                SensorsView()
                    .navigationTitle("Sensors")
            }
            .tabItem { Label("Sensors", systemImage: "waveform.path") }
        }
    }
}
```

### 1.2 — App Entry Point

**File:** `TrailTrackerApp.swift`

```swift
import SwiftUI

@main
struct TrailTrackerApp: App {
    let persistenceController = PersistenceController.shared

    var body: some Scene {
        WindowGroup {
            AppTabView()
                .environment(
                    \.managedObjectContext,
                    persistenceController.container.viewContext
                )
        }
    }
}
```

### 1.3 — Push Navigation (List → Detail)

```swift
NavigationLink(destination: TrailDetailView(trail: trail)) {
    TrailRowView(trail: trail)
}
```

### 1.4 — Programmatic Navigation

Use a `@State` path variable when you need to navigate after an async action (e.g., after a save completes):

```swift
struct TrailListView: View {
    @State private var navigationPath = NavigationPath()

    var body: some View {
        NavigationStack(path: $navigationPath) {
            List { ... }
                .navigationDestination(for: Trail.self) { trail in
                    TrailDetailView(trail: trail)
                }
        }
    }

    func navigateToNewTrail(_ trail: Trail) {
        navigationPath.append(trail)
    }
}
```

### 1.5 — Sheets and Alerts

```swift
struct TrailListView: View {
    @State private var showAddSheet = false
    @State private var showDeleteAlert = false

    var body: some View {
        List { ... }
        .toolbar {
            ToolbarItem(placement: .navigationBarTrailing) {
                Button { showAddSheet = true } label: {
                    Image(systemName: "plus")
                }
            }
        }
        .sheet(isPresented: $showAddSheet) { AddTrailView() }
        .alert("Delete Trail?", isPresented: $showDeleteAlert) {
            Button("Delete", role: .destructive) { /* delete */ }
            Button("Cancel", role: .cancel) {}
        }
    }
}
```

**Keywords:** `NavigationStack SwiftUI` · `TabView iOS` · `NavigationLink` · `programmatic navigation NavigationPath` · `sheet modifier` · `toolbar SwiftUI` · `push navigation` · `modal presentation`

---

## Part 2 — Sensors

Device sensors are accessed through the `CoreMotion` framework. `CMMotionManager` provides raw accelerometer and gyroscope data. `CMPedometer` provides higher-level step counting and pace.

> **Simulator note:** Accelerometer and gyroscope return no data in the iOS Simulator. Use a real device to test sensor output. Step counting is also unavailable in Simulator.

### 2.1 — MotionService

Wraps `CMMotionManager` and `CMPedometer` in a single service class that publishes state to the UI.

**File:** `Services/MotionService.swift`

```swift
import CoreMotion
import Combine

final class MotionService: ObservableObject {
    @Published var accelerometerData: CMAccelerometerData?
    @Published var gyroscopeData: CMGyroData?
    @Published var stepCount: Int = 0
    @Published var currentPace: Double = 0.0   // seconds per meter
    @Published var isAvailable: Bool = false

    private let motionManager = CMMotionManager()
    private let pedometer = CMPedometer()
    private let updateInterval: TimeInterval = 0.1  // 10 Hz

    init() {
        isAvailable = motionManager.isAccelerometerAvailable
    }

    // MARK: - Accelerometer

    func startAccelerometer() {
        guard motionManager.isAccelerometerAvailable else { return }
        motionManager.accelerometerUpdateInterval = updateInterval
        motionManager.startAccelerometerUpdates(to: .main) { [weak self] data, error in
            guard let data, error == nil else { return }
            self?.accelerometerData = data
        }
    }

    func stopAccelerometer() {
        motionManager.stopAccelerometerUpdates()
    }

    // MARK: - Gyroscope

    func startGyroscope() {
        guard motionManager.isGyroAvailable else { return }
        motionManager.gyroUpdateInterval = updateInterval
        motionManager.startGyroUpdates(to: .main) { [weak self] data, error in
            guard let data, error == nil else { return }
            self?.gyroscopeData = data
        }
    }

    func stopGyroscope() {
        motionManager.stopGyroUpdates()
    }

    // MARK: - Pedometer

    func startPedometer() {
        guard CMPedometer.isStepCountingAvailable() else { return }
        pedometer.startUpdates(from: Date()) { [weak self] data, error in
            guard let data, error == nil else { return }
            DispatchQueue.main.async {
                self?.stepCount = data.numberOfSteps.intValue
                if let pace = data.currentPace {
                    self?.currentPace = pace.doubleValue
                }
            }
        }
    }

    func stopPedometer() {
        pedometer.stopUpdates()
    }

    func startAll() { startAccelerometer(); startGyroscope(); startPedometer() }
    func stopAll()  { stopAccelerometer(); stopGyroscope(); stopPedometer() }
}
```

### 2.2 — SensorViewModel

Transforms raw sensor data into display-ready strings.

**File:** `ViewModels/SensorViewModel.swift`

```swift
import Foundation
import Combine

final class SensorViewModel: ObservableObject {
    private let motionService = MotionService()
    private var cancellables = Set<AnyCancellable>()

    @Published var formattedAccel: String = "—"
    @Published var stepCount: Int = 0

    init() {
        motionService.$accelerometerData
            .compactMap { $0 }
            .map { data in
                let mag = sqrt(
                    pow(data.acceleration.x, 2) +
                    pow(data.acceleration.y, 2) +
                    pow(data.acceleration.z, 2)
                )
                return String(format: "%.3f g", mag)
            }
            .assign(to: &$formattedAccel)

        motionService.$stepCount
            .assign(to: &$stepCount)
    }

    func start() { motionService.startAll() }
    func stop()  { motionService.stopAll() }
}
```

### 2.3 — SensorsView

**File:** `Views/Sensors/SensorsView.swift`

```swift
import SwiftUI

struct SensorsView: View {
    @StateObject private var motionService = MotionService()

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                if !motionService.isAvailable {
                    Label("Motion sensors unavailable on this device/simulator",
                          systemImage: "exclamationmark.triangle")
                        .foregroundStyle(.orange)
                        .padding()
                }

                SensorCard(title: "Accelerometer", systemImage: "gyroscope") {
                    if let accel = motionService.accelerometerData {
                        SensorRow(axis: "X", value: accel.acceleration.x)
                        SensorRow(axis: "Y", value: accel.acceleration.y)
                        SensorRow(axis: "Z", value: accel.acceleration.z)
                    } else {
                        Text("No data").foregroundStyle(.secondary)
                    }
                }

                SensorCard(title: "Gyroscope", systemImage: "arrow.3.trianglepath") {
                    if let gyro = motionService.gyroscopeData {
                        SensorRow(axis: "X", value: gyro.rotationRate.x)
                        SensorRow(axis: "Y", value: gyro.rotationRate.y)
                        SensorRow(axis: "Z", value: gyro.rotationRate.z)
                    } else {
                        Text("No data").foregroundStyle(.secondary)
                    }
                }

                SensorCard(title: "Pedometer", systemImage: "figure.walk") {
                    LabeledContent("Steps", value: "\(motionService.stepCount)")
                    LabeledContent("Pace",
                        value: String(format: "%.1f s/m", motionService.currentPace))
                }
            }
            .padding()
        }
        .onAppear { motionService.startAll() }
        .onDisappear { motionService.stopAll() }
    }
}

// MARK: - Reusable Subviews

struct SensorCard<Content: View>: View {
    let title: String
    let systemImage: String
    @ViewBuilder let content: () -> Content

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            Label(title, systemImage: systemImage).font(.headline)
            Divider()
            content()
        }
        .padding()
        .background(Color(.secondarySystemBackground))
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

struct SensorRow: View {
    let axis: String
    let value: Double

    var body: some View {
        HStack {
            Text(axis).bold().frame(width: 20)
            Spacer()
            Text(String(format: "%.4f", value))
                .monospacedDigit()
                .foregroundStyle(.secondary)
        }
    }
}
```

**Keywords:** `CoreMotion Swift` · `CMMotionManager` · `CMPedometer` · `accelerometer SwiftUI` · `gyroscope iOS` · `step counter Swift` · `@Published Combine` · `weak self closure` · `10 Hz sensor update`

---

## Part 3 — GPS & Location

GPS access is provided by `CoreLocation`. `CLLocationManager` requests permissions and delivers location updates via its delegate. Accuracy filtering is critical — raw GPS can be noisy, so we reject readings with `horizontalAccuracy >= 20m`.

### 3.1 — LocationService

**File:** `Services/LocationService.swift`

```swift
import CoreLocation
import Combine
import MapKit

final class LocationService: NSObject, ObservableObject, CLLocationManagerDelegate {
    @Published var authorizationStatus: CLAuthorizationStatus = .notDetermined
    @Published var currentLocation: CLLocation?
    @Published var currentRegion: MKCoordinateRegion = MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 37.3318, longitude: -122.0312),
        span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
    )
    @Published var recordedPath: [CLLocationCoordinate2D] = []
    @Published var isTracking: Bool = false
    @Published var totalDistance: Double = 0.0  // meters

    private let locationManager = CLLocationManager()
    private var lastLocation: CLLocation?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.distanceFilter = 5  // update every 5 meters
    }

    // MARK: - Permissions

    func requestPermission() {
        locationManager.requestWhenInUseAuthorization()
    }

    func requestAlwaysPermission() {
        locationManager.requestAlwaysAuthorization()
    }

    // MARK: - Tracking

    func startTracking() {
        recordedPath.removeAll()
        totalDistance = 0.0
        lastLocation = nil
        isTracking = true
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        locationManager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus
        switch manager.authorizationStatus {
        case .authorizedWhenInUse, .authorizedAlways:
            locationManager.startUpdatingLocation()
        case .denied, .restricted:
            break
        case .notDetermined:
            requestPermission()
        @unknown default:
            break
        }
    }

    func locationManager(_ manager: CLLocationManager,
                         didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        guard location.horizontalAccuracy < 20 else { return }  // filter noise

        currentLocation = location
        currentRegion = MKCoordinateRegion(
            center: location.coordinate,
            span: MKCoordinateSpan(latitudeDelta: 0.005, longitudeDelta: 0.005)
        )

        if isTracking {
            recordedPath.append(location.coordinate)
            if let last = lastLocation {
                totalDistance += location.distance(from: last)
            }
            lastLocation = location
        }
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        print("[LocationService] Error: \(error.localizedDescription)")
    }
}
```

### 3.2 — LiveTrackingView

Displays a live map with a polyline drawn as the user walks.

**File:** `Views/GPS/LiveTrackingView.swift`

```swift
import SwiftUI
import MapKit

struct LiveTrackingView: View {
    @StateObject private var locationService = LocationService()

    var body: some View {
        VStack {
            // Live map
            Map(coordinateRegion: $locationService.currentRegion,
                showsUserLocation: true,
                annotationItems: []) { _ in MapMarker(coordinate: .init()) }
                .ignoresSafeArea(edges: .top)
                .frame(maxHeight: .infinity)

            // Stats bar
            HStack {
                StatBadgeView(
                    label: "Distance",
                    value: String(format: "%.0f m", locationService.totalDistance)
                )
                Spacer()
                StatBadgeView(
                    label: "Points",
                    value: "\(locationService.recordedPath.count)"
                )
            }
            .padding()

            // Controls
            Button(locationService.isTracking ? "Stop" : "Start Tracking") {
                locationService.isTracking
                    ? locationService.stopTracking()
                    : locationService.startTracking()
            }
            .buttonStyle(.borderedProminent)
            .tint(locationService.isTracking ? .red : .green)
            .padding(.bottom)
        }
        .onAppear { locationService.requestPermission() }
    }
}
```

**Keywords:** `CoreLocation Swift` · `CLLocationManager delegate` · `GPS tracking iOS` · `horizontalAccuracy filter` · `MKCoordinateRegion` · `MapKit SwiftUI` · `requestWhenInUseAuthorization` · `distanceFilter` · `kCLLocationAccuracyBest` · `polyline map`

---

## Part 4 — List / Table View

`List` in SwiftUI is the equivalent of `UITableView`. It supports swipe-to-delete, pull-to-refresh, sectioning, and search natively.

### 4.1 — TrailListViewModel

Owns the `@FetchRequest` logic and delete/filter operations.

**File:** `ViewModels/TrailListViewModel.swift`

```swift
import Foundation
import CoreData
import Combine

final class TrailListViewModel: ObservableObject {
    @Published var searchText: String = ""

    private let viewContext: NSManagedObjectContext

    init(context: NSManagedObjectContext) {
        self.viewContext = context
    }

    func delete(_ trail: Trail) {
        viewContext.delete(trail)
        try? viewContext.save()
    }
}
```

### 4.2 — TrailListView

**File:** `Views/List/TrailListView.swift`

```swift
import SwiftUI
import CoreData

struct TrailListView: View {
    @Environment(\.managedObjectContext) private var viewContext

    @FetchRequest(
        sortDescriptors: [NSSortDescriptor(keyPath: \Trail.date, ascending: false)],
        animation: .default
    )
    private var trails: FetchedResults<Trail>

    @StateObject private var viewModel: TrailListViewModel
    @State private var searchText = ""

    init() {
        // viewModel is injected lazily; context arrives via @Environment
        _viewModel = StateObject(wrappedValue: TrailListViewModel(
            context: PersistenceController.shared.container.viewContext
        ))
    }

    var body: some View {
        List {
            ForEach(filteredTrails) { trail in
                NavigationLink(destination: TrailDetailView(trail: trail)) {
                    TrailRowView(trail: trail)
                }
            }
            .onDelete(perform: deleteTrails)
        }
        .searchable(text: $searchText, prompt: "Search trails")
        .navigationTitle("My Trails")
        .toolbar {
            EditButton()
        }
    }

    private var filteredTrails: [Trail] {
        guard !searchText.isEmpty else { return Array(trails) }
        return trails.filter {
            ($0.name ?? "").localizedCaseInsensitiveContains(searchText)
        }
    }

    private func deleteTrails(at offsets: IndexSet) {
        offsets.map { filteredTrails[$0] }.forEach(viewModel.delete)
    }
}
```

### 4.3 — TrailRowView

**File:** `Views/List/TrailRowView.swift`

```swift
import SwiftUI

struct TrailRowView: View {
    let trail: Trail

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            Text(trail.name ?? "Unnamed Trail")
                .font(.headline)
            HStack {
                Label(
                    String(format: "%.1f km", (trail.distanceMeters / 1000)),
                    systemImage: "location"
                )
                Spacer()
                Text(trail.date ?? Date(), style: .date)
            }
            .font(.caption)
            .foregroundStyle(.secondary)
        }
        .padding(.vertical, 4)
    }
}
```

**Keywords:** `SwiftUI List` · `FetchRequest Core Data` · `searchable modifier` · `onDelete swipe` · `ForEach CoreData` · `NSSortDescriptor` · `filteredResults` · `EditButton` · `NavigationLink list row`

---

## Part 5 — Core Data Persistence

Core Data is the on-device persistence layer. The stack consists of: `NSPersistentContainer` (owns the store) → `NSManagedObjectContext` (scratch pad for objects) → `NSManagedObject` subclasses (your model types).

### 5.1 — PersistenceController

**File:** `Services/PersistenceController.swift`

```swift
import CoreData

struct PersistenceController {
    static let shared = PersistenceController()

    // In-memory store for SwiftUI previews and unit tests
    static var preview: PersistenceController = {
        let controller = PersistenceController(inMemory: true)
        let context = controller.container.viewContext
        // Seed preview data
        let trail = Trail(context: context)
        trail.id = UUID()
        trail.name = "Sample Trail"
        trail.date = Date()
        trail.distanceMeters = 3420
        try? context.save()
        return controller
    }()

    let container: NSPersistentContainer

    init(inMemory: Bool = false) {
        container = NSPersistentContainer(name: "TrailTracker")
        if inMemory {
            container.persistentStoreDescriptions.first?.url =
                URL(fileURLWithPath: "/dev/null")
        }
        container.loadPersistentStores { _, error in
            if let error {
                fatalError("Core Data failed to load: \(error)")
            }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
    }
}
```

### 5.2 — Data Model (Trail entity)

In `TrailTracker.xcdatamodeld`, create a `Trail` entity with these attributes:

| Attribute | Type | Notes |
|---|---|---|
| `id` | UUID | Set as default on creation |
| `name` | String | User-provided trail name |
| `date` | Date | Recording start time |
| `distanceMeters` | Double | Total distance in meters |
| `durationSeconds` | Double | Total recording duration |
| `encodedPath` | String | Polyline encoded coordinate string |

Create a second entity `TrailPoint` with `latitude` (Double), `longitude` (Double), `timestamp` (Date), and a **to-many** relationship `points` on `Trail` (inverse: `trail` on `TrailPoint`).

### 5.3 — Saving a Trail

```swift
func saveTrail(name: String,
               distance: Double,
               duration: Double,
               path: [CLLocationCoordinate2D],
               context: NSManagedObjectContext) {
    let trail = Trail(context: context)
    trail.id = UUID()
    trail.name = name
    trail.date = Date()
    trail.distanceMeters = distance
    trail.durationSeconds = duration

    for coord in path {
        let point = TrailPoint(context: context)
        point.latitude = coord.latitude
        point.longitude = coord.longitude
        point.timestamp = Date()
        point.trail = trail
    }

    do {
        try context.save()
    } catch {
        print("[Persistence] Save failed: \(error)")
        context.rollback()
    }
}
```

### 5.4 — Fetching Trails

Use `@FetchRequest` directly in SwiftUI views:

```swift
@FetchRequest(
    sortDescriptors: [NSSortDescriptor(keyPath: \Trail.date, ascending: false)],
    animation: .default
)
private var trails: FetchedResults<Trail>
```

Or fetch programmatically from a ViewModel:

```swift
func fetchRecentTrails(limit: Int = 10) -> [Trail] {
    let request = Trail.fetchRequest()
    request.sortDescriptors = [NSSortDescriptor(keyPath: \Trail.date, ascending: false)]
    request.fetchLimit = limit
    return (try? viewContext.fetch(request)) ?? []
}
```

**Keywords:** `NSPersistentContainer` · `NSManagedObjectContext` · `Core Data stack` · `@FetchRequest SwiftUI` · `NSManagedObject subclass` · `save context` · `rollback` · `in-memory store preview` · `automaticallyMergesChangesFromParent` · `Core Data relationships`

---

## Putting It All Together

With all five parts implemented, wire everything up:

1. `TrailTrackerApp` injects the Core Data context via `.environment`.
2. `AppTabView` provides three tabs — each tab gets its own `NavigationStack`.
3. **Trails tab:** `TrailListView` fetches and displays saved trails, navigates to `TrailDetailView` on tap.
4. **Track tab:** `LiveTrackingView` uses `LocationService` to record a route. On stop, calls `saveTrail(...)` to persist to Core Data.
5. **Sensors tab:** `SensorsView` uses `MotionService` to display live accelerometer, gyroscope, and pedometer readings.

### Data Flow Diagram

```
User taps "Stop Tracking"
        │
        ▼
LiveTrackingView
        │  calls
        ▼
LocationService.stopTracking()
        │  returns path + distance
        ▼
saveTrail(..., context: viewContext)
        │  saves to
        ▼
NSPersistentContainer (SQLite on disk)
        │  triggers
        ▼
@FetchRequest in TrailListView refreshes
        │
        ▼
New trail appears in list
```

**Keywords:** `data flow SwiftUI` · `end-to-end iOS app` · `Core Data save flow` · `@FetchRequest refresh` · `ObservableObject chain` · `app wiring`

---

## Build & Run

### Run in Simulator

```
Product > Run   (⌘R)
```

> Sensors (accelerometer, gyroscope, pedometer) return no data in Simulator. Use a physical device to test those features. GPS can be simulated via `Debug > Simulate Location` in Xcode.

### Run on a Device

1. Connect your iPhone via USB.
2. Select your device in the Xcode toolbar (next to the scheme selector).
3. Sign with your Apple ID under **Target > Signing & Capabilities**.
4. Hit `⌘R`.

### Common Build Errors

| Error | Fix |
|---|---|
| `No such module 'CoreMotion'` | Add `CoreMotion.framework` in Target > General > Frameworks |
| `Thread 1: EXC_BAD_ACCESS` in Core Data | Ensure context operations happen on the main thread or use `context.perform {}` |
| Location permission denied silently | Verify `NSLocationWhenInUseUsageDescription` is in `Info.plist` |
| Simulator shows no sensor data | Test on a physical device |
| `FetchRequest` returns empty | Confirm entity name in `.xcdatamodeld` matches exactly (case-sensitive) |

**Keywords:** `Xcode build errors` · `run on device` · `iOS Simulator GPS simulation` · `Core Data EXC_BAD_ACCESS` · `signing Xcode` · `⌘R build and run`

---

## Keyword Reference

A quick-lookup glossary for every keyword used in this guide.

### Architecture & Patterns

| Keyword | What it is | How to use it |
|---|---|---|
| `MVVM architecture iOS` | Model-View-ViewModel — separates UI from business logic | Create a `ViewModel` class per screen; Views only read from it via `@StateObject` or `@ObservedObject` |
| `ObservableObject` | Protocol that lets a class publish changes to SwiftUI views | Conform your ViewModel/Service to `ObservableObject`; mark properties with `@Published` |
| `@Published` | Property wrapper that triggers a SwiftUI refresh when its value changes | Add `@Published var myValue = ""` inside an `ObservableObject` class |
| `service layer Swift` | A class that owns one system API (Location, Motion, etc.) and exposes clean methods | Create `final class FooService: ObservableObject` and inject it into your ViewModel |
| `separation of concerns` | Each type does one job — Views display, ViewModels format, Services call APIs | Don't put `CLLocationManager` directly in a View; wrap it in a `LocationService` |
| `weak self closure` | Prevents retain cycles inside async callbacks | Always capture `[weak self]` in callbacks passed to system APIs like `CMMotionManager` |

### SwiftUI Navigation

| Keyword | What it is | How to use it |
|---|---|---|
| `NavigationStack SwiftUI` | iOS 16+ replacement for `NavigationView` — manages a push/pop stack | Wrap your root view in `NavigationStack { ... }` |
| `NavigationLink` | A button that pushes a destination onto the `NavigationStack` | `NavigationLink(destination: DetailView()) { RowView() }` |
| `programmatic navigation NavigationPath` | Drive navigation from code instead of user taps | `@State var path = NavigationPath()` → `path.append(item)` |
| `TabView iOS` | Bottom tab bar container | `TabView { View1().tabItem { Label(...) } }` |
| `sheet modifier` | Presents a modal sheet | `.sheet(isPresented: $flag) { ModalView() }` |
| `toolbar SwiftUI` | Adds buttons to the navigation bar | `.toolbar { ToolbarItem(placement: .navigationBarTrailing) { Button(...) } }` |
| `modal presentation` | Any overlay that covers content (sheet, fullScreenCover, alert) | Use `.sheet` for partial cover, `.fullScreenCover` for full screen |

### SwiftUI List & Data Display

| Keyword | What it is | How to use it |
|---|---|---|
| `SwiftUI List` | Scrollable, lazily-loaded list of rows | `List { ForEach(items) { item in RowView(item) } }` |
| `searchable modifier` | Adds a search bar to a `List` | `.searchable(text: $query, prompt: "Search...")` |
| `onDelete swipe` | Enables swipe-to-delete on `ForEach` rows inside a `List` | `.onDelete(perform: deleteFunc)` on the `ForEach` |
| `EditButton` | Built-in button that toggles `List` edit mode (enables row deletion/reordering) | Add `EditButton()` inside a `.toolbar { }` block |
| `ForEach CoreData` | Iterates over `FetchedResults` in a `List` | `ForEach(fetchedResults) { item in ... }` |
| `NSSortDescriptor` | Defines sort order for a fetch request | `NSSortDescriptor(keyPath: \Trail.date, ascending: false)` |

### Core Data

| Keyword | What it is | How to use it |
|---|---|---|
| `NSPersistentContainer` | Manages the full Core Data stack (store + context) | Create once as a singleton in `PersistenceController`; access via `.shared` |
| `NSManagedObjectContext` | The "scratch pad" where objects live before being saved to disk | Inject via `.environment(\.managedObjectContext, ...)` and read with `@Environment` |
| `@FetchRequest SwiftUI` | Property wrapper that auto-fetches Core Data objects and updates the UI | `@FetchRequest(sortDescriptors: [...]) private var items: FetchedResults<Item>` |
| `NSManagedObject subclass` | A Swift class representing one Core Data entity | Generated by Xcode from your `.xcdatamodeld`; use it like a regular Swift class |
| `save context` | Commits in-memory changes to the SQLite store | `try context.save()` — always wrap in `do/catch` |
| `rollback` | Discards unsaved in-memory changes | `context.rollback()` — call in the `catch` block after a failed save |
| `in-memory store preview` | A Core Data store that lives in RAM and disappears on exit | Set `url = URL(fileURLWithPath: "/dev/null")` — use for SwiftUI previews and unit tests |
| `automaticallyMergesChangesFromParent` | Keeps `viewContext` in sync when background contexts save | Set `container.viewContext.automaticallyMergesChangesFromParent = true` in your stack init |
| `Core Data relationships` | Links between entities (one-to-many, many-to-many) | Define in `.xcdatamodeld` editor; access in Swift via the generated property (e.g., `trail.points`) |
| `Core Data environment` | Passing the context down the view tree via SwiftUI environment | `.environment(\.managedObjectContext, container.viewContext)` on the root view |

### CoreLocation & GPS

| Keyword | What it is | How to use it |
|---|---|---|
| `CLLocationManager delegate` | Receives location events via callbacks | Set `locationManager.delegate = self` and implement `didUpdateLocations` |
| `requestWhenInUseAuthorization` | Asks the user for location access while the app is open | Call once before starting location updates; requires `NSLocationWhenInUseUsageDescription` in `Info.plist` |
| `horizontalAccuracy filter` | `CLLocation.horizontalAccuracy` measures GPS precision in meters | Skip locations where `horizontalAccuracy >= 20` to filter out noisy readings |
| `distanceFilter` | Minimum meters the device must move before a new location event fires | `locationManager.distanceFilter = 5` — reduces battery use and data volume |
| `kCLLocationAccuracyBest` | Highest GPS accuracy mode (most battery) | `locationManager.desiredAccuracy = kCLLocationAccuracyBest` |
| `MKCoordinateRegion` | A map region defined by a center coordinate and a lat/lon span | `MKCoordinateRegion(center: coord, span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01))` |
| `MapKit SwiftUI` | Apple's map framework with a SwiftUI `Map` view | `import MapKit` then `Map(coordinateRegion: $region, showsUserLocation: true)` |
| `polyline map` | A line drawn on a map connecting a series of coordinates | Use `MKPolyline` + `MKPolylineRenderer` (UIKit) or `MapPolyline` (SwiftUI iOS 17+) |

### CoreMotion & Sensors

| Keyword | What it is | How to use it |
|---|---|---|
| `CMMotionManager` | Provides raw accelerometer and gyroscope data | Create one instance per app; call `startAccelerometerUpdates(to:withHandler:)` |
| `CMPedometer` | High-level step counter and pace tracker | `pedometer.startUpdates(from: Date()) { data, error in ... }` |
| `accelerometer SwiftUI` | Reading X/Y/Z acceleration forces (in g) | Access via `CMAccelerometerData.acceleration.x/.y/.z` |
| `gyroscope iOS` | Reading rotation rate around X/Y/Z axes (radians/sec) | Access via `CMGyroData.rotationRate.x/.y/.z` |
| `10 Hz sensor update` | Sensor polling rate of 10 times per second | `motionManager.accelerometerUpdateInterval = 0.1` (0.1 s = 10 Hz) |

### Permissions & Setup

| Keyword | What it is | How to use it |
|---|---|---|
| `NSLocationWhenInUseUsageDescription` | `Info.plist` key that provides the location permission prompt text | Add the key with a user-facing string explaining why you need location |
| `NSMotionUsageDescription` | `Info.plist` key for the motion/sensor permission prompt | Add with a string explaining why you need motion data |
| `Info.plist permissions` | Privacy usage description strings required before accessing protected APIs | Missing keys = silent permission denial; always add before calling the API |
| `Xcode frameworks` | System libraries linked to your target | Add under **Target > General > Frameworks, Libraries, and Embedded Content** |

---

## License

MIT. Build something, ship it.
