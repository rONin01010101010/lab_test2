# iOS Application Development Guide
## Navigation · Sensors · GPS · List/Table View · Core Data

A comprehensive, step-by-step guide to building a full-featured iOS application using Swift and UIKit/SwiftUI. This guide walks you through each concept independently so you can understand and integrate them into a cohesive app.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Setup](#project-setup)
3. [File Structure](#file-structure)
4. [Part 1 — Navigation](#part-1--navigation)
5. [Part 2 — Sensors](#part-2--sensors)
6. [Part 3 — GPS & Location](#part-3--gps--location)
7. [Part 4 — List / Table View](#part-4--list--table-view)
8. [Part 5 — Core Data](#part-5--core-data)
9. [Putting It All Together](#putting-it-all-together)
10. [Build & Run](#build--run)

---

## Prerequisites

| Requirement | Version |
|---|---|
| macOS | 13.0 Ventura or later |
| Xcode | 15.0 or later |
| Swift | 5.9 or later |
| iOS Deployment Target | iOS 16.0 or later |
| Apple Developer Account | Free (for Simulator); Paid (for device) |

Install Xcode from the Mac App Store. Once installed, open it and accept the license agreement. Install additional components if prompted.

---

## Project Setup

### Step 1 — Create the Xcode Project

1. Open **Xcode**.
2. Click **Create New Project** (or `File > New > Project`).
3. Select **iOS** tab → choose **App** → click **Next**.
4. Fill in the project details:
   - **Product Name:** `TrailTracker` (or any name you choose)
   - **Team:** Your Apple ID / Team
   - **Organization Identifier:** `com.yourname` (reverse domain style)
   - **Interface:** `SwiftUI` *(this guide uses SwiftUI)*
   - **Language:** `Swift`
   - **Storage:** `Core Data` ← check this box
   - **Include Tests:** optional
5. Click **Next**, choose a save location, click **Create**.

> **Note:** Checking "Core Data" at creation generates a pre-configured `Persistence.swift` file and updates `AppDelegate`/`@main` entry point automatically. You can always add Core Data manually later.

### Step 2 — Configure Info.plist Permissions

Open `Info.plist` (or the target's **Info** tab in Xcode 14+). Add these keys:

```xml
<!-- Location permissions -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>This app uses your location to track trails and record GPS paths.</string>

<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>This app uses your location in the background to continue tracking your route.</string>

<!-- Motion/sensor permissions -->
<key>NSMotionUsageDescription</key>
<string>This app uses motion sensors to count steps and detect activity.</string>
```

### Step 3 — Add Required Frameworks

In Xcode, select your **Target > General > Frameworks, Libraries, and Embedded Content**. Confirm the following are present (they are included automatically in modern Xcode):

- `CoreData.framework`
- `CoreLocation.framework`
- `CoreMotion.framework`
- `MapKit.framework`

---

## File Structure

Below is the recommended file structure for this project. Create groups in Xcode that mirror this layout (right-click in Project Navigator → New Group).

```
TrailTracker/
├── TrailTrackerApp.swift          # @main entry point, Core Data stack injection
├── ContentView.swift              # Root view — hosts NavigationStack
│
├── Navigation/
│   └── AppTabView.swift           # Tab bar controller (root navigation)
│
├── Models/
│   ├── TrailTracker.xcdatamodeld  # Core Data model (auto-generated)
│   ├── Trail+CoreDataClass.swift  # NSManagedObject subclass
│   ├── Trail+CoreDataProperties.swift
│   └── TrailPoint+CoreDataClass.swift
│
├── ViewModels/
│   ├── TrailListViewModel.swift   # Drives the list view
│   ├── TrailDetailViewModel.swift # Drives the detail view
│   └── SensorViewModel.swift      # Drives the sensors view
│
├── Views/
│   ├── List/
│   │   ├── TrailListView.swift    # List/Table view of saved trails
│   │   └── TrailRowView.swift     # Single row subview
│   ├── Detail/
│   │   ├── TrailDetailView.swift  # Detail screen for one trail
│   │   └── TrailMapView.swift     # MapKit subview
│   ├── Sensors/
│   │   └── SensorsView.swift      # Accelerometer, pedometer display
│   ├── GPS/
│   │   └── LiveTrackingView.swift # Real-time GPS tracking + map
│   └── Shared/
│       └── StatBadgeView.swift    # Reusable stat badge component
│
├── Services/
│   ├── LocationService.swift      # CLLocationManager wrapper
│   ├── MotionService.swift        # CMMotionManager wrapper
│   └── PersistenceController.swift# Core Data stack
│
└── Extensions/
    ├── Color+Extensions.swift
    └── Date+Extensions.swift
```

---

## Part 1 — Navigation

iOS navigation is managed through `NavigationStack` (SwiftUI) or `UINavigationController` (UIKit). This guide uses SwiftUI's `NavigationStack` combined with a `TabView` for a two-level navigation hierarchy.

### 1.1 — Tab Bar Root Navigation

The app uses a `TabView` as the root so users can switch between major sections.

**File:** `Navigation/AppTabView.swift`

```swift
import SwiftUI

struct AppTabView: View {
    var body: some View {
        TabView {
            // Tab 1 — Trail List
            NavigationStack {
                TrailListView()
                    .navigationTitle("My Trails")
            }
            .tabItem {
                Label("Trails", systemImage: "list.bullet")
            }

            // Tab 2 — Live GPS Tracking
            NavigationStack {
                LiveTrackingView()
                    .navigationTitle("Track")
            }
            .tabItem {
                Label("Track", systemImage: "location.fill")
            }

            // Tab 3 — Sensors
            NavigationStack {
                SensorsView()
                    .navigationTitle("Sensors")
            }
            .tabItem {
                Label("Sensors", systemImage: "waveform.path")
            }
        }
    }
}
```

### 1.2 — Entry Point

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

NavigationStack uses `NavigationLink` to push views onto the stack.

**File:** `Views/List/TrailListView.swift` (partial — full version in Part 4)

```swift
NavigationLink(destination: TrailDetailView(trail: trail)) {
    TrailRowView(trail: trail)
}
```

### 1.4 — Programmatic Navigation

For programmatic navigation (e.g., after saving data), use a `@State` path variable:

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

### 1.5 — Modal Sheets and Alerts

```swift
struct TrailListView: View {
    @State private var showAddSheet = false
    @State private var showDeleteAlert = false

    var body: some View {
        List { ... }
        .toolbar {
            ToolbarItem(placement: .navigationBarTrailing) {
                Button(action: { showAddSheet = true }) {
                    Image(systemName: "plus")
                }
            }
        }
        .sheet(isPresented: $showAddSheet) {
            AddTrailView()
        }
        .alert("Delete Trail?", isPresented: $showDeleteAlert) {
            Button("Delete", role: .destructive) { /* delete action */ }
            Button("Cancel", role: .cancel) {}
        }
    }
}
```

---

## Part 2 — Sensors

iOS provides access to device sensors through the `CoreMotion` framework. The key class is `CMMotionManager` for accelerometer, gyroscope, and magnetometer data, and `CMPedometer` for step counting.

### 2.1 — MotionService

Encapsulate all sensor access in a dedicated service class.

**File:** `Services/MotionService.swift`

```swift
import CoreMotion
import Combine

final class MotionService: ObservableObject {
    // MARK: - Published State
    @Published var accelerometerData: CMAccelerometerData?
    @Published var gyroscopeData: CMGyroData?
    @Published var stepCount: Int = 0
    @Published var currentPace: Double = 0.0   // seconds per meter
    @Published var isAvailable: Bool = false

    // MARK: - Private
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
            guard let data = data, error == nil else { return }
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
            guard let data = data, error == nil else { return }
            self?.gyroscopeData = data
        }
    }

    func stopGyroscope() {
        motionManager.stopGyroUpdates()
    }

    // MARK: - Pedometer

    func startPedometer() {
        guard CMPedometer.isStepCountingAvailable() else { return }
        let start = Date()
        pedometer.startUpdates(from: start) { [weak self] data, error in
            guard let data = data, error == nil else { return }
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

    // MARK: - Convenience

    func startAll() {
        startAccelerometer()
        startGyroscope()
        startPedometer()
    }

    func stopAll() {
        stopAccelerometer()
        stopGyroscope()
        stopPedometer()
    }
}
```

### 2.2 — SensorsView

**File:** `Views/Sensors/SensorsView.swift`

```swift
import SwiftUI
import CoreMotion

struct SensorsView: View {
    @StateObject private var motionService = MotionService()

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                // Availability banner
                if !motionService.isAvailable {
                    Label("Motion sensors unavailable on this device/simulator",
                          systemImage: "exclamationmark.triangle")
                        .foregroundStyle(.orange)
                        .padding()
                }

                // Accelerometer Card
                SensorCard(title: "Accelerometer", systemImage: "gyroscope") {
                    if let accel = motionService.accelerometerData {
                        SensorRow(axis: "X", value: accel.acceleration.x)
                        SensorRow(axis: "Y", value: accel.acceleration.y)
                        SensorRow(axis: "Z", value: accel.acceleration.z)
                    } else {
                        Text("No data").foregroundStyle(.secondary)
                    }
                }

                // Gyroscope Card
                SensorCard(title: "Gyroscope", systemImage: "arrow.3.trianglepath") {
                    if let gyro = motionService.gyroscopeData {
                        SensorRow(axis: "X", value: gyro.rotationRate.x)
                        SensorRow(axis: "Y", value: gyro.rotationRate.y)
                        SensorRow(axis: "Z", value: gyro.rotationRate.z)
                    } else {
                        Text("No data").foregroundStyle(.secondary)
                    }
                }

                // Pedometer Card
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

// MARK: - Subviews

struct SensorCard<Content: View>: View {
    let title: String
    let systemImage: String
    @ViewBuilder let content: () -> Content

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            Label(title, systemImage: systemImage)
                .font(.headline)
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

### 2.3 — SensorViewModel

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

---

## Part 3 — GPS & Location

GPS access is provided by `CoreLocation`. The `CLLocationManager` class requests permissions and delivers location updates.

### 3.1 — LocationService

**File:** `Services/LocationService.swift`

```swift
import CoreLocation
import Combine
import MapKit

final class LocationService: NSObject, ObservableObject, CLLocationManagerDelegate {
    // MARK: - Published State
    @Published var authorizationStatus: CLAuthorizationStatus = .notDetermined
    @Published var currentLocation: CLLocation?
    @Published var currentRegion: MKCoordinateRegion = MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 37.3318, longitude: -122.0312),
        span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
    )
    @Published var recordedPath: [CLLocationCoordinate2D] = []
    @Published var isTracking: Bool = false
    @Published var totalDistance: Double = 0.0  // meters

    // MARK: - Private
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

    // MARK: - Tracking Control

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

        // Filter out inaccurate readings
        guard location.horizontalAccuracy < 20 else { return }

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

    func locationManager(_ manager: CLLocationManager,
                         didFailWithError error: Error) {
        print("Location error: \(error.localizedDescription)")
    }

    // MARK: - Computed

    var totalDistanceKm: Double { totalDistance / 1000.0 }
}
```

### 3.2 — LiveTrackingView

**File:** `Views/GPS/LiveTrackingView.swift`

```swift
import SwiftUI
import MapKit

struct LiveTrackingView: View {
    @StateObject private var locationService = LocationService()

    var body: some View {
        ZStack(alignment: .bottom) {
            // Map
            Map(coordinateRegion: $locationService.currentRegion,
                showsUserLocation: true,
                annotationItems: [],
                annotationContent: { _ in MapPin(coordinate: .init()) }) {
                // Polyline overlay for recorded path
                // Note: Full polyline overlay requires MapKit UIViewRepresentable for complex paths
            }
            .ignoresSafeArea(edges: .top)

            // Stats Panel
            VStack(spacing: 0) {
                HStack(spacing: 30) {
                    StatBadgeView(
                        title: "Distance",
                        value: String(format: "%.2f km",
                                      locationService.totalDistanceKm)
                    )
                    StatBadgeView(
                        title: "Points",
                        value: "\(locationService.recordedPath.count)"
                    )
                    StatBadgeView(
                        title: "Status",
                        value: locationService.isTracking ? "Active" : "Idle"
                    )
                }
                .padding()

                // Track / Stop Button
                Button(action: toggleTracking) {
                    Label(
                        locationService.isTracking ? "Stop Tracking" : "Start Tracking",
                        systemImage: locationService.isTracking
                            ? "stop.circle.fill"
                            : "play.circle.fill"
                    )
                    .frame(maxWidth: .infinity)
                    .padding()
                    .background(locationService.isTracking ? Color.red : Color.green)
                    .foregroundStyle(.white)
                    .clipShape(RoundedRectangle(cornerRadius: 14))
                }
                .padding(.horizontal)
                .padding(.bottom)
            }
            .background(.ultraThinMaterial)
        }
        .onAppear {
            locationService.requestPermission()
        }
        // Show alert if denied
        .alert("Location Access Required",
               isPresented: .constant(
                   locationService.authorizationStatus == .denied
               )) {
            Button("Open Settings") {
                if let url = URL(string: UIApplication.openSettingsURLString) {
                    UIApplication.shared.open(url)
                }
            }
            Button("Cancel", role: .cancel) {}
        } message: {
            Text("Please enable location access in Settings to use GPS tracking.")
        }
    }

    private func toggleTracking() {
        if locationService.isTracking {
            locationService.stopTracking()
        } else {
            locationService.startTracking()
        }
    }
}
```

### 3.3 — MapKit Polyline (UIViewRepresentable)

For drawing the GPS path on the map, use a `UIViewRepresentable` wrapper:

**File:** `Views/GPS/TrailMapView.swift`

```swift
import SwiftUI
import MapKit

struct TrailMapView: UIViewRepresentable {
    let coordinates: [CLLocationCoordinate2D]
    let region: MKCoordinateRegion

    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        mapView.delegate = context.coordinator
        mapView.showsUserLocation = true
        return mapView
    }

    func updateUIView(_ mapView: MKMapView, context: Context) {
        mapView.setRegion(region, animated: true)

        // Remove old overlays, redraw path
        mapView.removeOverlays(mapView.overlays)
        if coordinates.count > 1 {
            let polyline = MKPolyline(
                coordinates: coordinates,
                count: coordinates.count
            )
            mapView.addOverlay(polyline)
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator()
    }

    // MARK: - Coordinator

    class Coordinator: NSObject, MKMapViewDelegate {
        func mapView(_ mapView: MKMapView,
                     rendererFor overlay: MKOverlay) -> MKOverlayRenderer {
            if let polyline = overlay as? MKPolyline {
                let renderer = MKPolylineRenderer(polyline: polyline)
                renderer.strokeColor = UIColor.systemBlue
                renderer.lineWidth = 4
                return renderer
            }
            return MKOverlayRenderer(overlay: overlay)
        }
    }
}
```

---

## Part 4 — List / Table View

SwiftUI's `List` is the modern equivalent of `UITableView`. It supports sections, swipe actions, reordering, and dynamic data from Core Data via `@FetchRequest`.

### 4.1 — Core Data Fetch with @FetchRequest

The `@FetchRequest` property wrapper connects a SwiftUI view directly to a Core Data entity.

**File:** `Views/List/TrailListView.swift`

```swift
import SwiftUI
import CoreData

struct TrailListView: View {
    @Environment(\.managedObjectContext) private var viewContext

    // Fetch trails sorted by date, newest first
    @FetchRequest(
        sortDescriptors: [NSSortDescriptor(keyPath: \Trail.date, ascending: false)],
        animation: .default
    )
    private var trails: FetchedResults<Trail>

    @State private var showAddSheet = false
    @State private var searchText = ""

    // Filtered trails based on search
    private var filteredTrails: [Trail] {
        if searchText.isEmpty { return Array(trails) }
        return trails.filter {
            ($0.name ?? "").localizedCaseInsensitiveContains(searchText)
        }
    }

    var body: some View {
        List {
            // Summary section
            Section {
                HStack {
                    Label("\(trails.count) trails recorded",
                          systemImage: "map.fill")
                    Spacer()
                    Text(totalDistanceText)
                        .foregroundStyle(.secondary)
                }
            }

            // Trail rows grouped by month
            ForEach(groupedTrails.keys.sorted().reversed(), id: \.self) { month in
                Section(header: Text(month)) {
                    ForEach(groupedTrails[month] ?? []) { trail in
                        NavigationLink(destination: TrailDetailView(trail: trail)) {
                            TrailRowView(trail: trail)
                        }
                    }
                    .onDelete { indexSet in
                        deleteTrails(from: groupedTrails[month] ?? [], at: indexSet)
                    }
                }
            }
        }
        .searchable(text: $searchText, prompt: "Search trails")
        .toolbar {
            ToolbarItem(placement: .navigationBarTrailing) {
                EditButton()
            }
            ToolbarItem(placement: .navigationBarTrailing) {
                Button { showAddSheet = true } label: {
                    Image(systemName: "plus")
                }
            }
        }
        .sheet(isPresented: $showAddSheet) {
            AddTrailView()
        }
        .overlay {
            if trails.isEmpty {
                ContentUnavailableView(
                    "No Trails Yet",
                    systemImage: "map",
                    description: Text("Tap + to record your first trail.")
                )
            }
        }
    }

    // MARK: - Grouping

    private var groupedTrails: [String: [Trail]] {
        let formatter = DateFormatter()
        formatter.dateFormat = "MMMM yyyy"
        return Dictionary(grouping: filteredTrails) { trail in
            formatter.string(from: trail.date ?? Date())
        }
    }

    private var totalDistanceText: String {
        let total = trails.reduce(0.0) { $0 + $1.distanceMeters }
        return String(format: "%.1f km", total / 1000)
    }

    // MARK: - Delete

    private func deleteTrails(from items: [Trail], at offsets: IndexSet) {
        withAnimation {
            offsets.map { items[$0] }.forEach(viewContext.delete)
            try? viewContext.save()
        }
    }
}
```

### 4.2 — TrailRowView

**File:** `Views/List/TrailRowView.swift`

```swift
import SwiftUI

struct TrailRowView: View {
    let trail: Trail

    var body: some View {
        HStack(spacing: 14) {
            // Color indicator / icon
            ZStack {
                Circle()
                    .fill(Color.blue.opacity(0.15))
                    .frame(width: 44, height: 44)
                Image(systemName: "figure.hiking")
                    .foregroundStyle(.blue)
            }

            VStack(alignment: .leading, spacing: 4) {
                Text(trail.name ?? "Unnamed Trail")
                    .font(.headline)
                    .lineLimit(1)

                HStack(spacing: 8) {
                    Label(
                        String(format: "%.2f km", trail.distanceMeters / 1000),
                        systemImage: "ruler"
                    )
                    .font(.caption)
                    .foregroundStyle(.secondary)

                    Label(
                        "\(trail.stepCount) steps",
                        systemImage: "figure.walk"
                    )
                    .font(.caption)
                    .foregroundStyle(.secondary)
                }
            }

            Spacer()

            // Date
            VStack(alignment: .trailing) {
                Text(trail.date ?? Date(), style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }
        }
        .padding(.vertical, 4)
    }
}
```

### 4.3 — TrailDetailView

**File:** `Views/Detail/TrailDetailView.swift`

```swift
import SwiftUI
import CoreData

struct TrailDetailView: View {
    @ObservedObject var trail: Trail
    @Environment(\.managedObjectContext) private var viewContext
    @State private var isEditing = false
    @State private var editedName: String = ""

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 20) {
                // Map
                if let points = trail.trailPoints?.allObjects as? [TrailPoint],
                   !points.isEmpty {
                    let coords = points
                        .sorted { ($0.timestamp ?? Date()) < ($1.timestamp ?? Date()) }
                        .map { CLLocationCoordinate2D(latitude: $0.latitude,
                                                      longitude: $0.longitude) }
                    TrailMapView(
                        coordinates: coords,
                        region: regionFor(coords)
                    )
                    .frame(height: 250)
                    .clipShape(RoundedRectangle(cornerRadius: 16))
                }

                // Stats Grid
                LazyVGrid(columns: [.init(), .init()], spacing: 16) {
                    StatCard(title: "Distance",
                             value: String(format: "%.2f km",
                                           trail.distanceMeters / 1000),
                             icon: "ruler.fill")
                    StatCard(title: "Steps",
                             value: "\(trail.stepCount)",
                             icon: "figure.walk")
                    StatCard(title: "Duration",
                             value: formattedDuration,
                             icon: "clock.fill")
                    StatCard(title: "Points",
                             value: "\(trail.trailPoints?.count ?? 0)",
                             icon: "mappin.and.ellipse")
                }
                .padding(.horizontal)

                // Date
                Label(
                    (trail.date ?? Date()).formatted(date: .long, time: .shortened),
                    systemImage: "calendar"
                )
                .foregroundStyle(.secondary)
                .padding(.horizontal)
            }
            .padding(.vertical)
        }
        .navigationTitle(trail.name ?? "Trail")
        .navigationBarTitleDisplayMode(.large)
        .toolbar {
            ToolbarItem(placement: .navigationBarTrailing) {
                Button("Edit") { isEditing = true }
            }
        }
        .sheet(isPresented: $isEditing) {
            EditTrailView(trail: trail)
        }
    }

    private var formattedDuration: String {
        let seconds = trail.durationSeconds
        let h = Int(seconds) / 3600
        let m = (Int(seconds) % 3600) / 60
        return h > 0 ? "\(h)h \(m)m" : "\(m)m"
    }

    private func regionFor(_ coords: [CLLocationCoordinate2D]) -> MKCoordinateRegion {
        guard !coords.isEmpty else {
            return MKCoordinateRegion(
                center: .init(latitude: 37.33, longitude: -122.03),
                span: .init(latitudeDelta: 0.01, longitudeDelta: 0.01)
            )
        }
        let lats = coords.map(\.latitude)
        let lons = coords.map(\.longitude)
        let center = CLLocationCoordinate2D(
            latitude: (lats.min()! + lats.max()!) / 2,
            longitude: (lons.min()! + lons.max()!) / 2
        )
        let span = MKCoordinateSpan(
            latitudeDelta: (lats.max()! - lats.min()!) * 1.5 + 0.002,
            longitudeDelta: (lons.max()! - lons.min()!) * 1.5 + 0.002
        )
        return MKCoordinateRegion(center: center, span: span)
    }
}

struct StatCard: View {
    let title: String
    let value: String
    let icon: String

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(.blue)
            Text(value).font(.title3).bold()
            Text(title).font(.caption).foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(Color(.secondarySystemBackground))
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}
```

### 4.4 — StatBadgeView (Shared)

**File:** `Views/Shared/StatBadgeView.swift`

```swift
import SwiftUI

struct StatBadgeView: View {
    let title: String
    let value: String

    var body: some View {
        VStack(spacing: 4) {
            Text(value)
                .font(.headline)
                .monospacedDigit()
            Text(title)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .frame(minWidth: 70)
    }
}
```

---

## Part 5 — Core Data

Core Data is an object graph persistence framework. It maps Swift objects to a SQLite store (by default) and provides powerful querying, change tracking, and undo/redo.

### 5.1 — Core Data Stack (PersistenceController)

**File:** `Services/PersistenceController.swift`

```swift
import CoreData

struct PersistenceController {
    static let shared = PersistenceController()

    // In-memory store for SwiftUI previews
    static let preview: PersistenceController = {
        let controller = PersistenceController(inMemory: true)
        let context = controller.container.viewContext
        // Seed preview data
        for i in 1...5 {
            let trail = Trail(context: context)
            trail.id = UUID()
            trail.name = "Trail \(i)"
            trail.date = Calendar.current.date(byAdding: .day, value: -i, to: Date())
            trail.distanceMeters = Double.random(in: 500...8000)
            trail.stepCount = Int32.random(in: 600...10000)
            trail.durationSeconds = Double.random(in: 600...7200)
        }
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

        container.loadPersistentStores { description, error in
            if let error = error {
                // In production, handle this gracefully
                fatalError("Core Data failed to load: \(error.localizedDescription)")
            }
        }

        // Automatically merge changes from background contexts
        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
    }

    // MARK: - Save Helper

    func save() {
        let context = container.viewContext
        guard context.hasChanges else { return }
        do {
            try context.save()
        } catch {
            print("Core Data save error: \(error)")
        }
    }

    // MARK: - Background Context

    func performBackgroundTask(_ block: @escaping (NSManagedObjectContext) -> Void) {
        container.performBackgroundTask(block)
    }
}
```

### 5.2 — Core Data Model

Open `TrailTracker.xcdatamodeld` in Xcode and create the following entities:

#### Entity: `Trail`

| Attribute | Type | Notes |
|---|---|---|
| `id` | UUID | Default: new UUID |
| `name` | String | Required |
| `date` | Date | Required |
| `distanceMeters` | Double | Default: 0.0 |
| `stepCount` | Integer 32 | Default: 0 |
| `durationSeconds` | Double | Default: 0.0 |
| `notes` | String | Optional |

**Relationship:** `trailPoints` → to-many → `TrailPoint` (inverse: `trail`)

#### Entity: `TrailPoint`

| Attribute | Type | Notes |
|---|---|---|
| `latitude` | Double | |
| `longitude` | Double | |
| `altitude` | Double | |
| `timestamp` | Date | |
| `horizontalAccuracy` | Double | |

**Relationship:** `trail` → to-one → `Trail` (inverse: `trailPoints`)

In Xcode's Data Model Inspector:
- Set **Class > Codegen** to `Category/Extension` for both entities
- Set **Delete Rule** on `trailPoints` to `Cascade` (deleting a Trail deletes its points)

### 5.3 — NSManagedObject Subclasses

**File:** `Models/Trail+CoreDataClass.swift`

```swift
import Foundation
import CoreData

@objc(Trail)
public class Trail: NSManagedObject {}
```

**File:** `Models/Trail+CoreDataProperties.swift`

```swift
import Foundation
import CoreData

extension Trail {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<Trail> {
        return NSFetchRequest<Trail>(entityName: "Trail")
    }

    @NSManaged public var id: UUID?
    @NSManaged public var name: String?
    @NSManaged public var date: Date?
    @NSManaged public var distanceMeters: Double
    @NSManaged public var stepCount: Int32
    @NSManaged public var durationSeconds: Double
    @NSManaged public var notes: String?
    @NSManaged public var trailPoints: NSSet?
}

// MARK: - TrailPoint relationship helpers
extension Trail {
    @objc(addTrailPointsObject:)
    @NSManaged public func addToTrailPoints(_ value: TrailPoint)

    @objc(removeTrailPointsObject:)
    @NSManaged public func removeFromTrailPoints(_ value: TrailPoint)

    @objc(addTrailPoints:)
    @NSManaged public func addToTrailPoints(_ values: NSSet)

    @objc(removeTrailPoints:)
    @NSManaged public func removeFromTrailPoints(_ values: NSSet)
}

extension Trail: Identifiable {}
```

**File:** `Models/TrailPoint+CoreDataClass.swift`

```swift
import Foundation
import CoreData

@objc(TrailPoint)
public class TrailPoint: NSManagedObject {}
```

> Create `TrailPoint+CoreDataProperties.swift` similarly with the attributes listed in 5.2.

### 5.4 — Saving a Trail from GPS Data

Create a save function that takes `LocationService` output and persists it. This is typically called from the tracking view when the user taps "Stop".

**File:** `ViewModels/TrailDetailViewModel.swift`

```swift
import Foundation
import CoreData
import CoreLocation

final class TrailDetailViewModel: ObservableObject {
    private let context: NSManagedObjectContext

    init(context: NSManagedObjectContext) {
        self.context = context
    }

    func saveTrail(
        name: String,
        date: Date,
        distanceMeters: Double,
        stepCount: Int32,
        durationSeconds: Double,
        coordinates: [CLLocationCoordinate2D]
    ) {
        context.perform {
            let trail = Trail(context: self.context)
            trail.id = UUID()
            trail.name = name
            trail.date = date
            trail.distanceMeters = distanceMeters
            trail.stepCount = stepCount
            trail.durationSeconds = durationSeconds

            // Save each GPS point
            for (index, coord) in coordinates.enumerated() {
                let point = TrailPoint(context: self.context)
                point.latitude = coord.latitude
                point.longitude = coord.longitude
                point.timestamp = Date(timeIntervalSinceNow: Double(index))
                trail.addToTrailPoints(point)
            }

            do {
                try self.context.save()
            } catch {
                print("Failed to save trail: \(error)")
            }
        }
    }

    func deleteTrail(_ trail: Trail) {
        context.delete(trail)
        try? context.save()
    }
}
```

### 5.5 — AddTrailView (Save from Sheet)

**File:** `Views/List/AddTrailView.swift` *(create this file)*

```swift
import SwiftUI

struct AddTrailView: View {
    @Environment(\.managedObjectContext) private var viewContext
    @Environment(\.dismiss) private var dismiss

    @State private var name: String = ""
    @State private var notes: String = ""
    @State private var isSaving = false

    var body: some View {
        NavigationStack {
            Form {
                Section("Trail Info") {
                    TextField("Trail name", text: $name)
                    TextField("Notes (optional)", text: $notes, axis: .vertical)
                        .lineLimit(3...6)
                }
            }
            .navigationTitle("New Trail")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("Save") { save() }
                        .disabled(name.trimmingCharacters(in: .whitespaces).isEmpty)
                }
            }
        }
    }

    private func save() {
        let trail = Trail(context: viewContext)
        trail.id = UUID()
        trail.name = name.trimmingCharacters(in: .whitespaces)
        trail.date = Date()
        trail.notes = notes.isEmpty ? nil : notes
        trail.distanceMeters = 0
        trail.stepCount = 0
        trail.durationSeconds = 0

        try? viewContext.save()
        dismiss()
    }
}
```

### 5.6 — Advanced Core Data: Background Saves

For saving large batches (e.g., hundreds of GPS points) without blocking the UI:

```swift
func saveLargeTrailInBackground(coordinates: [CLLocationCoordinate2D]) {
    PersistenceController.shared.performBackgroundTask { bgContext in
        let trail = Trail(context: bgContext)
        trail.id = UUID()
        trail.name = "Background Trail"
        trail.date = Date()

        for coord in coordinates {
            let point = TrailPoint(context: bgContext)
            point.latitude = coord.latitude
            point.longitude = coord.longitude
            point.timestamp = Date()
            trail.addToTrailPoints(point)
        }

        try? bgContext.save()
        // viewContext is automatically updated via automaticallyMergesChangesFromParent
    }
}
```

### 5.7 — Predicates and Filtering

```swift
// Fetch trails from the last 30 days
let calendar = Calendar.current
let thirtyDaysAgo = calendar.date(byAdding: .day, value: -30, to: Date())!

let request = Trail.fetchRequest()
request.predicate = NSPredicate(
    format: "date >= %@ AND distanceMeters > %f",
    thirtyDaysAgo as CVarArg,
    1000.0
)
request.sortDescriptors = [NSSortDescriptor(keyPath: \Trail.date, ascending: false)]

// In SwiftUI use it dynamically:
@FetchRequest(
    sortDescriptors: [NSSortDescriptor(keyPath: \Trail.date, ascending: false)],
    predicate: NSPredicate(format: "distanceMeters > 1000")
)
private var longTrails: FetchedResults<Trail>
```

---

## Putting It All Together

The sections above are independent modules. Here is how they connect in the final app flow:

```
App Launch
    └── AppTabView (TabView)
            ├── Tab 1: TrailListView
            │       └── NavigationLink → TrailDetailView
            │               └── TrailMapView (MapKit)
            ├── Tab 2: LiveTrackingView
            │       ├── LocationService (GPS)
            │       ├── MotionService (Pedometer)
            │       └── "Save" → TrailDetailViewModel.saveTrail()
            │               └── Core Data save → Trail + TrailPoints
            └── Tab 3: SensorsView
                    └── MotionService (Accelerometer, Gyroscope)
```

### Integration: Save a tracked trail to Core Data

When the user finishes a tracked trail, wire the GPS and sensor data to Core Data:

```swift
// In LiveTrackingView — "Save Trail" button action
func saveCurrent() {
    let viewModel = TrailDetailViewModel(context: viewContext)
    viewModel.saveTrail(
        name: "Trail \(Date().formatted(date: .abbreviated, time: .omitted))",
        date: Date(),
        distanceMeters: locationService.totalDistance,
        stepCount: Int32(motionService.stepCount),
        durationSeconds: trackingDuration,
        coordinates: locationService.recordedPath
    )
}
```

---

## Build & Run

### Step 1 — Simulator

1. In Xcode, select a simulator (e.g., **iPhone 16**) from the device picker.
2. Press `Cmd + R` to build and run.
3. To test GPS in the Simulator:
   - With the app running, go to **Simulator menu > Features > Location**
   - Choose a preset like **City Bicycle Ride** or **Freeway Drive**
   - Or set a custom location via **Custom Location...**
4. Sensors (accelerometer, gyroscope) are **not available** in the Simulator. You will see the "unavailable" banner in `SensorsView`. Use a real device to test sensor data.

### Step 2 — Physical Device

1. Connect your iPhone via USB.
2. In Xcode Target settings, ensure your **Team** is set under **Signing & Capabilities**.
3. Select your device from the picker.
4. Press `Cmd + R`.
5. On first run, go to **Settings > General > VPN & Device Management** and trust your developer certificate.

### Step 3 — Common Build Errors

| Error | Fix |
|---|---|
| `Couldn't load NSManagedObject subclass` | Ensure Codegen is set to `Category/Extension` and the entity class name matches |
| `Thread 1: EXC_BAD_ACCESS` in Core Data | Check that you're not accessing `NSManagedObject` properties from a background thread without a context |
| Location not updating | Check `Info.plist` has `NSLocationWhenInUseUsageDescription`. Re-run after adding. |
| `CMMotionManager` returns nil data | Normal in Simulator. Test on device. |
| `No such module 'CoreMotion'` | Add `CoreMotion.framework` in Target > Frameworks |

### Step 4 — Running Tests

```bash
# Run unit tests
Cmd + U

# Or from terminal
xcodebuild test \
  -scheme TrailTracker \
  -destination 'platform=iOS Simulator,name=iPhone 16'
```

### Step 5 — Checking Core Data Store

To inspect the SQLite store during development:

```swift
// Add to PersistenceController init, after loadPersistentStores
print(container.persistentStoreCoordinator
    .persistentStores.first?.url ?? "unknown")
```

Copy the printed path, then open it in **DB Browser for SQLite** or use the Xcode `CoreData` debugger argument:

Add to **Scheme > Run > Arguments Passed On Launch**:
```
-com.apple.CoreData.SQLDebug 1
-com.apple.CoreData.Logging.stderr 1
```

---

*This guide was written for Xcode 15+ and iOS 16+ using Swift 5.9 and SwiftUI. All APIs demonstrated are part of Apple's standard frameworks and require no third-party dependencies.*
