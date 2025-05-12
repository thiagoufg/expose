# Exposé for MacOS

Step 1: Create the script  
1. Open any text editor and paste the code that you can find at the end of this page.  
2. Save it in your `Documents` folder as `expose.swift`
3. (Optional) If you want to test this script, run it with the command bellow (but you don't need to do this right now):  
   ```bash
   cd ~/Documents
   swift expose.swift
   ```

Step 2: Create an Automator Application  
1. Open Automator.  
2. Create a New Document:  
   - Select Application as the type of document.  
3. Add a “Run Shell Script” Action:  
   - In the left-hand library pane, search for “Run Shell Script.”  
   - Drag and drop the “Run Shell Script” action into the workflow pane.  
4. Enter Your Shell Script:  
   ```bash
   #!/bin/sh
   /usr/bin/swift ~/Documents/expose.swift
   ```
5. Save as an app.  
6. Move the app to the Applications folder.  
7. Right-click on it, choose Open, and allow necessary permissions.  

Step 3: Update Security and Privacy Settings  
1. Go to **Settings > Security > Accessibility**:  
   - Enable `expose`.  
2. Go to **Settings > Security > Automation**:  
   - Enable `expose`.  
3. For Logi Options:  
   - Select a button to open the app: `expose.app`.  

Step 4: Export the Application  
1. Go to **File > Export**.  
2. In the Export As field, give your app a name.  
3. In the File Format dropdown menu, select **Application**.  
4. (Optional) Check the Stay Open checkbox if the app should remain running after execution.  

Step 5: Add App to Privacy Categories  
1. Go to **System Preferences > Security & Privacy > Privacy**.  
2. Add your app to relevant categories (e.g., Accessibility, Automation).  

---

AppleScript Code for Window Arrangement:  

```swift
import Cocoa
import ApplicationServices

// Get screen dimensions
let screenWidth = NSScreen.main?.frame.size.width ?? 0
let screenHeight = NSScreen.main?.frame.size.height ?? 0

// Initialize array for windows
var allWindows: [(win: NSWindow, xCoord: CGFloat, widthHeight: NSSize, appName: String)] = []

let alternateCenterWindowSize = true
let isLargerCenterPreferred = true
// Function to get app windows
func getAppWindows(bundleIdentifier: String) -> [AXUIElement] {
    guard let app = NSRunningApplication.runningApplications(withBundleIdentifier: bundleIdentifier).first else {
        return []
    }
    
    let appRef = AXUIElementCreateApplication(app.processIdentifier)
    
    var windowsList: CFTypeRef?
    let result = AXUIElementCopyAttributeValue(appRef, kAXWindowsAttribute as CFString, &windowsList)
    
    guard result == .success,
          let windowsArray = (windowsList as? NSArray) as? [AXUIElement] else {
        return []
    }
    
    return windowsArray
}

// Function to collect visible windows
func collectWindows() -> [(window: AXUIElement, xCoord: CGFloat, widthHeight: NSSize, appName: String)] {
    var collectedWindows: [(window: AXUIElement, xCoord: CGFloat, widthHeight: NSSize, appName: String)] = []
    
    let runningApplications = NSWorkspace.shared.runningApplications
    for app in runningApplications {
        if app.activationPolicy == .regular, let appBundle = app.bundleIdentifier {
            let appName = app.localizedName ?? "Unknown"
            let appWindows = getAppWindows(bundleIdentifier: appBundle)
            
            for window in appWindows {
                var position = CGPoint.zero
                var size = CGSize.zero
                
                // Get position
                var positionValue: AnyObject?
                AXUIElementCopyAttributeValue(window, kAXPositionAttribute as CFString, &positionValue)
                if let positionValue = positionValue {
                    AXValueGetValue(positionValue as! AXValue, .cgPoint, &position)
                }
                
                // Get size
                var sizeValue: AnyObject?
                AXUIElementCopyAttributeValue(window, kAXSizeAttribute as CFString, &sizeValue)
                if let sizeValue = sizeValue {
                    AXValueGetValue(sizeValue as! AXValue, .cgSize, &size)
                }
                
                // Get minimized state
                var minimizedValue: AnyObject?
                AXUIElementCopyAttributeValue(window, kAXMinimizedAttribute as CFString, &minimizedValue)
                let isMinimized = (minimizedValue as? NSNumber)?.boolValue ?? false
                
                // Skip Finder windows at (0,0), Zoom windows, and minimized windows
                if (appName == "Finder" && position.x == 0 && position.y == 0) ||
                   appName == "zoom.us" ||
                   isMinimized {
                    continue
                }
                
                // Prepare the window entry
                let newWindow = (window, position.x, NSSize(width: size.width, height: size.height), appName)
                
                // Insert window based on xCoord
                if let index = collectedWindows.firstIndex(where: { $0.xCoord > position.x }) {
                    collectedWindows.insert(newWindow, at: index)
                } else {
                    collectedWindows.append(newWindow)
                }
            }
        }
    }
    return collectedWindows
}

// Function to arrange windows side by side
func arrangeWindowsSideBySide() {
    // Check accessibility permissions
    let options = [kAXTrustedCheckOptionPrompt.takeUnretainedValue() as String: true]
    let accessibilityEnabled = AXIsProcessTrustedWithOptions(options as CFDictionary)
    
    print("Accessibility enabled:", accessibilityEnabled)
    if !accessibilityEnabled {
        print("Please enable accessibility permissions in System Preferences")
        exit(1)
    }

    let windows = collectWindows()
    print("Collected windows count:", windows.count)
    
    if windows.isEmpty {
        print("No windows found")
        exit(0)
    }
    
    let windowCount = windows.count
    let targetWidth: CGFloat
    // Store initial width of 3rd window if it exists
    let centerWindowInitialWidth = windowCount > 0 ? windows[Int(floor(Double(windowCount)/2))].widthHeight.width : 0
    if (windowCount == 4 || windowCount == 5) && 
       (alternateCenterWindowSize ? abs(centerWindowInitialWidth - screenWidth / CGFloat(windowCount)) < 1.0 : isLargerCenterPreferred) {
        targetWidth = screenWidth / 6
    } else if windowCount == 3 && 
       (alternateCenterWindowSize ? abs(centerWindowInitialWidth - screenWidth / CGFloat(windowCount)) < 1.0 : isLargerCenterPreferred) {
        targetWidth = screenWidth / 5
    } else if windowCount <= 2 {
        targetWidth = screenWidth / 3
    } else {
        targetWidth = screenWidth / CGFloat(windowCount)
    }
    let targetHeight = screenHeight
    
    // Animation parameters
    let n: Int = 10  // Number of animation steps
    var hasAnyWindowMoved = false
    
    for step in 1...n {
        for (index, windowDetails) in windows.enumerated() {
            // Get current position and size
            var currentPosition = CGPoint.zero
            var currentSize = CGSize.zero
            
            var positionValue: AnyObject?
            var sizeValue: AnyObject?
            AXUIElementCopyAttributeValue(windowDetails.window, kAXPositionAttribute as CFString, &positionValue)
            AXUIElementCopyAttributeValue(windowDetails.window, kAXSizeAttribute as CFString, &sizeValue)
            
            if let positionValue = positionValue {
                AXValueGetValue(positionValue as! AXValue, .cgPoint, &currentPosition)
            }
            if let sizeValue = sizeValue {
                AXValueGetValue(sizeValue as! AXValue, .cgSize, &currentSize)
            }
            
            // Calculate target position and size based on window count and index
            let finalX: CGFloat
            let finalWidth: CGFloat
            
            if  windowCount == 5 && (alternateCenterWindowSize ? centerWindowInitialWidth == (screenWidth / CGFloat(windowCount)) : isLargerCenterPreferred) {
                if index < 2 {
                    finalX = targetWidth * CGFloat(index)
                    finalWidth = targetWidth
                } else if index == 2 {
                    finalX = targetWidth * CGFloat(index)
                    finalWidth = targetWidth * 2
                } else {
                    finalX = targetWidth * CGFloat(index + 1)
                    finalWidth = targetWidth
                }
            } else if  windowCount == 4 && (alternateCenterWindowSize ? centerWindowInitialWidth == (screenWidth / CGFloat(windowCount)) : isLargerCenterPreferred) {
                if index == 0 {
                    finalX = targetWidth * 0
                    finalWidth = targetWidth
                } else if index == 1 {
                    finalX = targetWidth * 1
                    finalWidth = targetWidth * 2
                } else if index == 2 {
                    finalX = targetWidth * CGFloat(index + 1)
                    finalWidth = targetWidth * 2
                } else {
                    finalX = targetWidth * CGFloat(index + 2)
                    finalWidth = targetWidth
                }
            } else if  windowCount == 3 && (alternateCenterWindowSize ? abs(centerWindowInitialWidth - (screenWidth / CGFloat(windowCount))) < 1.0 : isLargerCenterPreferred) {
                if index == 0 {
                    finalX = targetWidth * 0
                    finalWidth = targetWidth
                } else if index == 1 {
                    finalX = targetWidth * CGFloat(index)
                    finalWidth = targetWidth * 3
                } else {
                    finalX = targetWidth * 4
                    finalWidth = targetWidth
                }
            } else {
                // Original behavior for other window counts
                if windowCount == 1 {
                    finalX = screenWidth / 3
                } else if windowCount == 2 {
                    if index == 0 {
                        finalX = (screenWidth / 2) - targetWidth
                    } else {
                        finalX = screenWidth / 2
                    }
                } else {
                    finalX = targetWidth * CGFloat(index)
                }
                finalWidth = targetWidth
            }
            
            let finalY = screenHeight - targetHeight
            
            // Check if this window is actually moving
            if abs(currentPosition.x - finalX) > 1 ||
               abs(currentPosition.y - finalY) > 1 ||
               abs(currentSize.width - finalWidth) > 1 ||
               abs(currentSize.height - targetHeight) > 1 {
                hasAnyWindowMoved = true
            }

            // Interpolate between current and final positions/sizes
            let progress = CGFloat(step) / CGFloat(n)
            let newX = currentPosition.x + (finalX - currentPosition.x) * progress
            let newY = currentPosition.y + (finalY - currentPosition.y) * progress
            let newWidth = currentSize.width + (finalWidth - currentSize.width) * progress
            let newHeight = currentSize.height + (targetHeight - currentSize.height) * progress
            
            // Create position and size values
            let position = CGPoint(x: newX, y: newY)
            let size = CGSize(width: newWidth, height: newHeight)
            
            // Convert position and size to AXValue
            var axPosition = position
            let axPositionRef = AXValueCreate(.cgPoint, &axPosition)!
            
            var axSize = size
            let axSizeRef = AXValueCreate(.cgSize, &axSize)!
            
            // Set new position and size
            AXUIElementSetAttributeValue(windowDetails.window, kAXPositionAttribute as CFString, axPositionRef)
            AXUIElementSetAttributeValue(windowDetails.window, kAXSizeAttribute as CFString, axSizeRef)
        }
    }
    
    if !hasAnyWindowMoved {
        let alert = NSAlert()
        alert.messageText = "hi"
        alert.runModal()
    }
}

// Replace the CSV output with window arrangement
arrangeWindowsSideBySide()

```
