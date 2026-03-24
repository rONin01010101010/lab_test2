# Labtest2_YourName — iOS Location Tracker

> A two-screen iOS app that requests the device's current GPS coordinates on demand and displays them on a dedicated detail screen. Built with Swift and SwiftUI (or UIKit + Storyboard).

---

## Table of Contents

1. [What We're Building](#what-were-building)
2. [Prerequisites](#prerequisites)
3. [Project Setup](#project-setup)
4. [File Structure](#file-structure)
5. [Key Concepts & Keywords](#key-concepts--keywords)
6. [Step-by-Step Implementation](#step-by-step-implementation)
   - [Info.plist — Location Permission String](#1-infoplist--location-permission-string)
   - [LocationManager — CLLocationManager Wrapper](#2-locationmanager--cllocationmanager-wrapper)
   - [Screen 1 — Home View](#3-screen-1--home-view)
   - [Screen 2 — Coordinates View](#4-screen-2--coordinates-view)
   - [App Entry Point](#5-app-entry-point)
7. [Complete Source Code](#complete-source-code)
8. [Build & Run](#build--run)
9. [Grading Checklist](#grading-checklist)

---

## What We're Building

**Labtest2_YourName** is a minimal two-screen iOS app that:

- Asks the user for location permission on first launch (popup must show **your name and student ID**)
- Has a **"Get Location" button** on Screen 1 that triggers a single GPS fix using `CLLocationManager`
- Navigates to **Screen 2** where the latitude and longitude coordinates are shown
- Displays **your name and student ID** on every screen

No persistent storage is required.

**Keywords:** `CLLocationManager` · `CoreLocation` · `SwiftUI NavigationStack` · `requestWhenInUseAuthorization` · `requestLocation` · `CLLocationCoordinate2D`

---

## Prerequisites

| Requirement | Version |
|---|---|
| macOS | 13.0 Ventura or later |
| Xcode | 15.0 or later |
| Swift | 5.9 or later |
| iOS Deployment Target | iOS 16.0 or later |
| Apple Developer Account | Free account (Simulator only) |

---

## Project Setup

### Step 1 — Create the Xcode Project

1. Open **Xcode → Create New Project**
2. Choose **iOS → App → Next**
3. Fill in details:
   - **Product Name:** `Labtest2_YourName` ← replace `YourName`
   - **Organization Identifier:** `com.yourname`
   - **Interface:** `SwiftUI`
   - **Language:** `Swift`
   - **Storage:** `None` (no Core Data needed)
4. Click **Next**, choose a save location, click **Create**

### Step 2 — Add Location Permission to Info.plist

Open `Info.plist` (or the target's **Info** tab) and add one key:

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>Your Name (StudentID) needs your location to display GPS coordinates.</string>
```

> **This string is what appears in the system permission popup.** It MUST contain your name and student ID to receive full marks.

**Keywords:** `NSLocationWhenInUseUsageDescription` · `Info.plist` · `location permission popup`

### Step 3 — No Extra Frameworks Needed

`CoreLocation` is included automatically in modern Xcode. Simply `import CoreLocation` in your Swift files.

---

## File Structure

```
Labtest2_YourName/
├── Labtest2_YourName.xcodeproj/
│   └── project.pbxproj
├── Labtest2_YourName/
│   ├── Labtest2_YournameApp.swift      ← @main entry point
│   ├── ContentView.swift               ← Screen 1 (Home + Get Location button)
│   ├── CoordinatesView.swift           ← Screen 2 (Show lat/lon)
│   ├── LocationManager.swift           ← CLLocationManager wrapper (ObservableObject)
│   ├── Assets.xcassets/
│   └── Info.plist                      ← NSLocationWhenInUseUsageDescription here
└── README.md
```

---

## Key Concepts & Keywords

| Concept | Keyword / API | Purpose |
|---|---|---|
| Location framework | `import CoreLocation` | Gives access to GPS hardware |
| Permission request | `requestWhenInUseAuthorization()` | Triggers the system popup |
| Single GPS fix | `requestLocation()` | Best choice for on-demand position; fires delegate once then stops |
| Continuous tracking | `startUpdatingLocation()` | Continuous stream — NOT needed here |
| Delegate callback | `locationManager(_:didUpdateLocations:)` | Called with the GPS result |
| Error callback | `locationManager(_:didFailWithError:)` | Called if GPS unavailable |
| Coordinate type | `CLLocationCoordinate2D` | Holds `.latitude` and `.longitude` as `Double` |
| Observable state | `@Published var coordinate` | Publishes new location to SwiftUI views |
| Navigation | `NavigationStack` + `NavigationLink` | Two-screen navigation |
| Reactive binding | `@StateObject` / `@ObservedObject` | Inject `LocationManager` into views |

### Why `requestLocation()` instead of `startUpdatingLocation()`?

> The requirement says **"on request"** — a single fix when the button is pressed. `requestLocation()` is designed exactly for this: it calls the delegate **once** with the best available fix and automatically stops. It is battery-efficient and matches the spec perfectly.

---

## Step-by-Step Implementation

### 1. Info.plist — Location Permission String

In Xcode, select your target → **Info** tab → click **+** → add:

| Key | Value |
|---|---|
| `NSLocationWhenInUseUsageDescription` | `Your Name (StudentID) needs your location to display GPS coordinates.` |

This exact string appears in the iOS permission alert.

---

### 2. LocationManager — CLLocationManager Wrapper

Create `LocationManager.swift`. This class:
- Inherits `NSObject` and conforms to `CLLocationManagerDelegate`
- Is marked `ObservableObject` so SwiftUI views can react to changes
- Exposes `@Published var coordinate: CLLocationCoordinate2D?`
- Calls `requestWhenInUseAuthorization()` on init
- Calls `requestLocation()` when `fetchLocation()` is invoked

**Keywords:** `ObservableObject` · `@Published` · `CLLocationManagerDelegate` · `NSObject` · `CLLocationManager`

```swift
import CoreLocation
import Foundation

class LocationManager: NSObject, ObservableObject, CLLocationManagerDelegate {

    private let manager = CLLocationManager()

    @Published var coordinate: CLLocationCoordinate2D? = nil
    @Published var errorMessage: String? = nil

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()   // triggers the popup
    }

    /// Call this when the button is pressed — fires ONE location update then stops.
    func fetchLocation() {
        errorMessage = nil
        manager.requestLocation()   // best service for on-demand single fix
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager,
                         didUpdateLocations locations: [CLLocation]) {
        coordinate = locations.last?.coordinate
    }

    func locationManager(_ manager: CLLocationManager,
                         didFailWithError error: Error) {
        errorMessage = error.localizedDescription
    }
}
```

---

### 3. Screen 1 — Home View

`ContentView.swift` is the first screen. It:
- Displays your **Name and Student ID** at the top
- Has a **"Get Location" button** that calls `locationManager.fetchLocation()`
- Navigates to `CoordinatesView` passing the received coordinate
- Shows a status message while waiting

**Keywords:** `NavigationStack` · `NavigationLink` · `@StateObject` · `Button` · `VStack`

```swift
import SwiftUI
import CoreLocation

struct ContentView: View {

    @StateObject private var locationManager = LocationManager()
    @State private var navigateToCoords = false

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {

                // Name + Student ID — required on every screen
                Text("Your Name — StudentID")
                    .font(.headline)
                    .foregroundColor(.secondary)

                Spacer()

                Text("Location Tracker")
                    .font(.largeTitle.bold())

                Button("Get Current Location") {
                    locationManager.fetchLocation()
                }
                .buttonStyle(.borderedProminent)
                .controlSize(.large)

                // Navigate automatically when coordinate arrives
                .onChange(of: locationManager.coordinate) { newCoord in
                    if newCoord != nil {
                        navigateToCoords = true
                    }
                }

                if let error = locationManager.errorMessage {
                    Text("Error: \(error)")
                        .foregroundColor(.red)
                        .multilineTextAlignment(.center)
                        .padding(.horizontal)
                }

                Spacer()

                NavigationLink(
                    destination: CoordinatesView(
                        coordinate: locationManager.coordinate
                    ),
                    isActive: $navigateToCoords
                ) {
                    EmptyView()
                }
            }
            .padding()
            .navigationTitle("Home")
        }
    }
}
```

---

### 4. Screen 2 — Coordinates View

`CoordinatesView.swift` receives the coordinate and displays it. It also shows your **Name and Student ID**.

**Keywords:** `NavigationLink destination` · `CLLocationCoordinate2D` · `latitude` · `longitude`

```swift
import SwiftUI
import CoreLocation

struct CoordinatesView: View {

    let coordinate: CLLocationCoordinate2D?

    var body: some View {
        VStack(spacing: 20) {

            // Name + Student ID — required on every screen
            Text("Your Name — StudentID")
                .font(.headline)
                .foregroundColor(.secondary)

            Spacer()

            if let coord = coordinate {
                Group {
                    Label("Latitude", systemImage: "location.north.fill")
                        .font(.title2.bold())
                    Text(String(format: "%.6f°", coord.latitude))
                        .font(.title.monospacedDigit())

                    Label("Longitude", systemImage: "location.fill")
                        .font(.title2.bold())
                        .padding(.top, 8)
                    Text(String(format: "%.6f°", coord.longitude))
                        .font(.title.monospacedDigit())
                }
            } else {
                Text("No coordinates available.")
                    .foregroundColor(.secondary)
            }

            Spacer()
        }
        .padding()
        .navigationTitle("Coordinates")
        .navigationBarTitleDisplayMode(.inline)
    }
}
```

---

### 5. App Entry Point

`Labtest2_YournameApp.swift` — this is generated by Xcode, no changes needed:

```swift
import SwiftUI

@main
struct Labtest2_YournameApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

---

## Complete Source Code

Below is every file needed for a complete, buildable project.

---

### `Info.plist` (key to add)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- ... existing keys ... -->
    <key>NSLocationWhenInUseUsageDescription</key>
    <string>Your Name (StudentID) needs your location to display GPS coordinates.</string>
</dict>
</plist>
```

---

### `LocationManager.swift` (complete)

```swift
import CoreLocation
import Foundation

/// Wraps CLLocationManager for SwiftUI consumption.
/// Uses requestLocation() for a single on-demand GPS fix — battery efficient
/// and satisfies the "on button press" requirement.
class LocationManager: NSObject, ObservableObject, CLLocationManagerDelegate {

    private let manager = CLLocationManager()

    /// Latest GPS coordinate published to any observing SwiftUI view.
    @Published var coordinate: CLLocationCoordinate2D? = nil

    /// Populated if the hardware or permissions prevent a fix.
    @Published var errorMessage: String? = nil

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        // This call triggers the system permission popup on first launch.
        // The popup text comes from NSLocationWhenInUseUsageDescription in Info.plist.
        manager.requestWhenInUseAuthorization()
    }

    /// Trigger a one-shot location fetch. Call this when the user taps the button.
    func fetchLocation() {
        errorMessage = nil
        coordinate = nil
        manager.requestLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager,
                         didUpdateLocations locations: [CLLocation]) {
        // requestLocation() guarantees at least one element; take the latest.
        coordinate = locations.last?.coordinate
    }

    func locationManager(_ manager: CLLocationManager,
                         didFailWithError error: Error) {
        errorMessage = error.localizedDescription
    }
}
```

---

### `ContentView.swift` (complete — Screen 1)

```swift
import SwiftUI
import CoreLocation

struct ContentView: View {

    @StateObject private var locationManager = LocationManager()
    @State private var navigateToCoords = false
    @State private var isFetching = false

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {

                // ── Required on every screen ──────────────────────────────
                Text("Your Name — StudentID")
                    .font(.subheadline)
                    .foregroundColor(.secondary)
                // ─────────────────────────────────────────────────────────

                Spacer()

                Image(systemName: "location.circle.fill")
                    .resizable()
                    .frame(width: 80, height: 80)
                    .foregroundColor(.accentColor)

                Text("Location Tracker")
                    .font(.largeTitle.bold())

                Button {
                    isFetching = true
                    locationManager.fetchLocation()
                } label: {
                    Label(
                        isFetching ? "Fetching…" : "Get Current Location",
                        systemImage: "location.fill"
                    )
                }
                .buttonStyle(.borderedProminent)
                .controlSize(.large)
                .disabled(isFetching)

                if let error = locationManager.errorMessage {
                    Text("⚠ \(error)")
                        .foregroundColor(.red)
                        .font(.footnote)
                        .multilineTextAlignment(.center)
                        .padding(.horizontal)
                }

                Spacer()

                // Hidden navigation trigger
                NavigationLink(
                    destination: CoordinatesView(coordinate: locationManager.coordinate),
                    isActive: $navigateToCoords
                ) { EmptyView() }
            }
            .padding()
            .navigationTitle("Home")
            // Navigate to Screen 2 as soon as a coordinate arrives
            .onChange(of: locationManager.coordinate) { newCoord in
                if newCoord != nil {
                    isFetching = false
                    navigateToCoords = true
                }
            }
            .onChange(of: locationManager.errorMessage) { _ in
                isFetching = false
            }
            // Reset navigation state when user comes back to Screen 1
            .onAppear {
                navigateToCoords = false
            }
        }
    }
}

#Preview {
    ContentView()
}
```

---

### `CoordinatesView.swift` (complete — Screen 2)

```swift
import SwiftUI
import CoreLocation

struct CoordinatesView: View {

    let coordinate: CLLocationCoordinate2D?

    var body: some View {
        VStack(spacing: 20) {

            // ── Required on every screen ──────────────────────────────
            Text("Your Name — StudentID")
                .font(.subheadline)
                .foregroundColor(.secondary)
            // ─────────────────────────────────────────────────────────

            Spacer()

            if let coord = coordinate {
                VStack(spacing: 32) {
                    coordinateCard(
                        title: "Latitude",
                        value: coord.latitude,
                        icon: "location.north.fill",
                        color: .blue
                    )
                    coordinateCard(
                        title: "Longitude",
                        value: coord.longitude,
                        icon: "location.fill",
                        color: .green
                    )
                }
            } else {
                VStack(spacing: 12) {
                    Image(systemName: "location.slash")
                        .font(.system(size: 60))
                        .foregroundColor(.secondary)
                    Text("No coordinates available.")
                        .foregroundColor(.secondary)
                }
            }

            Spacer()
        }
        .padding()
        .navigationTitle("Coordinates")
        .navigationBarTitleDisplayMode(.inline)
    }

    @ViewBuilder
    private func coordinateCard(title: String,
                                value: Double,
                                icon: String,
                                color: Color) -> some View {
        VStack(spacing: 8) {
            Label(title, systemImage: icon)
                .font(.title2.bold())
                .foregroundColor(color)
            Text(String(format: "%.6f°", value))
                .font(.system(.title, design: .monospaced))
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(color.opacity(0.1))
        .cornerRadius(12)
    }
}

#Preview {
    NavigationStack {
        CoordinatesView(coordinate: CLLocationCoordinate2D(latitude: 43.7, longitude: -79.4))
    }
}
```

---

### `Labtest2_YournameApp.swift` (complete — entry point)

```swift
import SwiftUI

@main
struct Labtest2_YournameApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

---

## Build & Run

### Simulator

1. Open `Labtest2_YourName.xcodeproj` in Xcode.
2. Select an **iPhone** simulator from the device picker (e.g. iPhone 15).
3. Press **⌘R** to build and run.
4. To simulate GPS in the simulator:
   - While the app is running, go to **Debug → Simulate Location → Apple** (or any preset).
   - The simulator does not have real GPS; you must use a simulated location.
5. Tap **Get Current Location** → the permission popup appears → tap **Allow**.
6. The app navigates to Screen 2 and shows the simulated coordinates.

### Physical Device

1. Plug in an iPhone.
2. Select it as the run target.
3. Ensure your Apple ID is set under **Signing & Capabilities → Team**.
4. Press **⌘R**. The app will use real GPS hardware.

**Keywords:** `⌘R build` · `Simulate Location` · `Signing & Capabilities` · `Apple ID team`

---

## Grading Checklist

| Requirement | Where implemented | Points |
|---|---|---|
| Project named `Labtest2_YourName` | Xcode project name | — |
| Permission popup shows your name + student ID | `Info.plist` → `NSLocationWhenInUseUsageDescription` | -5 if missing |
| GPS location received from device | `CLLocationManager.requestLocation()` in `LocationManager.swift` | 5 pts |
| Location fetched **on button press** | `Button` in `ContentView` calls `fetchLocation()` — uses `requestLocation()` (one-shot, best service) | 2 pts |
| Second screen shows coordinates | `CoordinatesView.swift` displays lat/lon | 3 pts |
| Name + Student ID on **all** screens | `Text("Your Name — StudentID")` in both views | -5 if missing |
| No persistent storage | No Core Data, no UserDefaults | required |

---

## Quick-Reference Keyword Glossary

| Term | Meaning |
|---|---|
| `CoreLocation` | Apple framework for GPS and location services |
| `CLLocationManager` | The main class that interacts with the location hardware |
| `CLLocationManagerDelegate` | Protocol whose callbacks deliver location results |
| `requestWhenInUseAuthorization()` | Shows the system permission popup; reads from `NSLocationWhenInUseUsageDescription` |
| `requestLocation()` | One-shot GPS request — calls delegate once, then stops. Best for on-demand use. |
| `startUpdatingLocation()` | Continuous stream — not appropriate for this task |
| `didUpdateLocations` | Delegate method called with a `[CLLocation]` array when GPS fix arrives |
| `CLLocationCoordinate2D` | Struct with `.latitude` and `.longitude` (both `Double`, in degrees) |
| `ObservableObject` | SwiftUI protocol — lets a class publish state changes |
| `@Published` | Property wrapper — notifies SwiftUI views when value changes |
| `@StateObject` | Creates and owns an `ObservableObject` instance in a SwiftUI view |
| `NavigationStack` | SwiftUI container for push-navigation between screens |
| `NavigationLink` | Button that pushes a destination view onto the `NavigationStack` |
| `NSLocationWhenInUseUsageDescription` | The `Info.plist` key whose value appears verbatim in the permission popup |
| `kCLLocationAccuracyBest` | Constant requesting maximum GPS accuracy |
