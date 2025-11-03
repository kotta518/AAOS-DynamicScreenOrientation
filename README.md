
# Software Architecture: Dynamic Automotive Display Rotation

This document outlines the architecture for achieving seamless dynamic display rotation in Android Automotive OS (AAOS) for both system and third-party (3P) applications. It leverages platform services and Runtime Resource Overlays (RROs) to ensure a high-performance, consistent user experience.

-----

## 1\. System Overview

This system modifies core AAOS services to manage dynamic screen orientation changes (Portrait $\rightleftharpoons$ Landscape) centrally, **eliminating application restarts** and ensuring a **consistent, flicker-free User Experience (UX)** across the infotainment system. This approach is superior to relying on individual application-level code.

-----

## 2\. System Context

The solution operates at the **AAOS Framework Layer**, centralizing resource orchestration and control away from individual applications.

| Component | Role | Data Flow |
| :--- | :--- | :--- |
| **Vehicle HAL (VHAL)** | Hardware Source: Reports physical display rotation status (e.g., $0^{\circ}$ to $90^{\circ}$). | **Input** to CarService. |
| **Car Service** | **Data Broker:** Monitors the VHAL property and provides the standardized rotation signal to the framework. | Passes rotation signal to WMS. |
| **WindowManagerService (WMS)** | **Central Controller:** Intercepts VHAL events, triggers RRO switch, and broadcasts the **Configuration Change** event. | Orchestrates system-wide resource update. |
| **Runtime Resource Overlays (RROs)** | **Dynamic Resource Layer:** Provides orientation-specific resource values (dimens, styles, layouts) to target apps. | Overlays base app resources instantly. |
| **Applications (Compose / Views)** | **Consumer:** Receives Configuration Change; triggers UI redraw via **Recomposition** or View update. | Updates UI using resources supplied by the active RRO. |

-----

## 3\. Key Components

  * **WMS Patch:** The critical platform modification. Implements VHAL listening and calls the `IOverlayManager` (a privileged system API) directly within its internal rotation handler.

  * **RRO Packages:** Two separate overlay APKs (`LandscapeRRO.apk` and `PortraitRRO.apk`) containing only the resources required to modify system and 3P apps to fit the respective orientation.

  * **Application Activity:** Must include the `android:configChanges="orientation|screenLayout|screenSize"` flag in its manifest to allow the system to handle the rotation without triggering `onDestroy()` and `onCreate()`.

-----

## 4\. Architectural Decisions

<img width="601" height="714" alt="Picture1" src="https://github.com/user-attachments/assets/5b59f402-486d-456d-93ee-b2baf217d3be" />


| Decision | Justification |
| :--- | :--- |
| **WMS Patching (No Custom Service)** | **Performance Optimization:** Eliminates IPC and the overhead of a dedicated system service, achieving maximum speed and efficiency in a core system loop. |
| **Runtime Resource Overlays (RROs)** | **System Consistency & 3P Compatibility:** Guarantees all apps adopt the OEM-defined style and dimensions upon rotation. This simplifies 3P app development dramatically. |
| **Focus on Jetpack Compose / View Model** | **Seamless UX:** Both Compose (via Recomposition) and View-based apps (via simplified `onConfigurationChanged`) benefit from the instantaneous resource update provided by the RRO. |

-----

## 5\. Quality Attributes

  * **Availability:** High. Activity restart is avoided, minimizing application downtime during rotation.

  * **Scalability:** High. Adding new applications requires no platform changes; they automatically inherit the dynamic behavior.

  * **UX/Performance:** Near-instantaneous transition achieved by the RRO's fast resource swapping and Compose's efficient rendering pipeline.

-----

## 6\. Diagrams and Pseudocode

### 6.1. System Architecture Flow Description

-----

  * **Flow:** **VHAL** $\rightarrow$ **CarService** $\rightarrow$ **WMS (Patched)** $\rightarrow$ **IOverlayManager (RRO Switch)** $\rightarrow$ **WMS (Broadcast Config Change)** $\rightarrow$ **Application UI Update**.

  * **Key Action:** The WMS simultaneously enables one RRO (e.g., Landscape) and disables the other (e.g., Portrait), *then* broadcasts the system configuration change.

### 6.2. Application Lifecycle Diagram Description

-----
<img width="587" height="840" alt="rotation" src="https://github.com/user-attachments/assets/27834f57-2b7c-441c-8532-a27128d78963" />

  * **Comparison:** Contrasts the default lifecycle (`onDestroy` $\rightarrow$ `onCreate`) with the RRO approach: **Configuration Change** $\rightarrow$ **WMS RRO Switch** $\rightarrow$ `onConfigurationChanged()` $\rightarrow$ **Compose/View Redraw**.

  * **Result:** Highlights that the Activity remains alive and receives updated resources seamlessly via the RRO layer.

### 6.3. Pseudocode: WMS RRO Logic

This demonstrates the core logic injected into the **WindowManagerService** to control the resource switch.

```java
// WMS Internal Logic (Pseudocode within the rotation handler)

private void applyRROandRotate(int newRotation) {
    IOverlayManager om = getOverlayManager();

    // 1. Switch RRO based on new physical orientation
    if (isLandscape(newRotation)) {
        // Activate Landscape RRO; Deactivate Portrait RRO
        om.setEnabled(LANDSCAPE_RRO_PACKAGE, true, UserHandle.SYSTEM);
        om.setEnabled(PORTRAIT_RRO_PACKAGE, false, UserHandle.SYSTEM);
    } else {
        // Activate Portrait RRO; Deactivate Landscape RRO
        om.setEnabled(PORTRAIT_RRO_PACKAGE, true, UserHandle.SYSTEM);
        om.setEnabled(LANDSCAPE_RRO_PACKAGE, false, UserHandle.SYSTEM);
    }

    // 2. Broadcast the configuration change. This forces all apps 
    // to redraw using the now-active RRO resources.
    sendConfigurationChangeToApps(); 
}
```
## 7\. ❓ Assumtions/queries


 * **VHAL Integration Success (Assumption):** We assume the physical rotation event is reliably detected and communicated from the Vehicle ECU to the Vehicle HAL (VHAL), and subsequently exposed to the Android framework via the Car Service.

* **Rotation Lock Strategy (Query):** We need a policy decision on how rotation is blocked while driving. Should this be an absolute hardware lock enforced by the ECU, or a software lock enforced by the Car Service checking the drive state? (This impacts system robustness and safety.)

* **Legacy/Non-Compliant App Handling (Query):** What is the specific strategy and required level of testing for legacy applications that might not handle configuration changes gracefully or may not adhere to modern resource standards?

* **AOSP Patching Privilege (Assumption):** We assume the development team has the necessary access and permission to modify and compile the core WindowManagerService (WMS).

* **3rd Party App Compliance (Assumption):** We assume that all 3rd party applications running on the system will include the critical android:configChanges="orientation|screenLayout|screenSize" flag in their manifest to prevent activity restarts.

* **Fixed Orientation Apps are Unaffected (Assumption):** We confirm that applications with a hard-coded fixed orientation (set via manifest) will not be impacted by the dynamic RRO switch, as Android's framework prioritizes the manifest declaration. This ensures backward compatibility for rigid apps.
