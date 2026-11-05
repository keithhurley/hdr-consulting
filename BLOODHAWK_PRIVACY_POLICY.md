# BloodHawk™ Privacy Policy

**Application:** BloodHawk™ (Android Mobile Application)  
**Developer & Publisher:** Hurley Data and Research Consulting LLC  
**Postal Address:** 679 Hickory Circle, Wahoo, NE 68066, United States of America  
**Contact Email:** HDRConsulting.support@gmail.com  
**Effective Date:** November 1, 2026  

---

## 1. Overview & Core Privacy Commitment

BloodHawk™ is an on-device field tool designed to assist hunters in tracking blood trails and recovering game. 

- **Your Data Stays on Your Device:** BloodHawk processes camera imagery in real time and stores your saved recovery trails, custom blood profiles, and user preferences locally on your handset.
- **No Developer Servers or Tracking:** Hurley Data and Research Consulting LLC does not operate remote user servers in conjunction with and use with BloodHawk, does not maintain user accounts, and does not collect or receive your camera frames, photos, GPS trails, or telemetry.
- **No Sale or Advertising:** Hurley Data and Research Consulting LLC does not sell, rent, or share personal information with third parties for commercial, marketing, or advertising purposes.

---

## 2. On-Device Processing & Data Storage

- **Live Camera Scanning (Temporary RAM Only):** When you use optical blood tracking, frames from your device’s camera are analyzed in temporary device memory (RAM) in real time to highlight blood droplets. Live camera frames are discarded from memory after rendering and are never recorded to disk or transmitted over the network.
- **Calibration & Hit Diagnostic Photos:** When you freeze a camera frame to calibrate a blood profile or take a photo in Hit Site Diagnostic, the image is held only in temporary memory (RAM) while that screen is open. BloodHawk does not save calibration or diagnostic photos to storage—only your numeric color bounds, profile labels, notes, and timestamps are saved in local app storage.
- **Foreground GPS & Trail Plotting:** BloodHawk accesses your device’s GPS location only while the application is actively open in the foreground. Location updates stop immediately when you pause tracking, switch away from the app, or lock your device screen. Saved waypoints, custom field markers, and path coordinates remain in private local storage on your device.
- **User-Initiated Exports & Sharing:** If you choose to export or share a recovery trail or custom profile (`.bhawk`, `.gpx`, `.geojson`, `.kml`, `.json`), BloodHawk generates an unencrypted file and passes it to the Android share sheet or file picker. Trail exports can contain location coordinates, timestamps, names, and notes, depending on the selected format. Any subsequent storage, transmission, or cloud sync is governed by the destination app or service you choose (such as Google Drive, email, or a messaging app).
- **Data Security & Device Sandboxing:** On-device data is protected by standard Android operating-system application sandboxing. BloodHawk does not independently apply proprietary encryption to internal database files or exported files. You are responsible for maintaining your device's passcode, lock screen, and physical security.
- **Local Data Retention & Deletion:** Saved recovery sessions, custom profiles, and cached map tiles remain in the app's private storage until you delete them inside the app or uninstall BloodHawk. Uninstalling the app removes all data stored within the app’s private sandbox; previously exported files or shared copies must be deleted separately where they were saved.

---

## 3. Network Connections & Third-Party Services

Blood tracking and saved-trail features work offline. Map imagery requires previously downloaded or cached tiles, or an internet connection. Network connections occur only in the following specific circumstances:

### 3.1 Cartographic Map Tile Providers
When you view an online map layer or download an offline map pack, your handset connects directly via HTTPS to public map tile services:
- **U.S. Geological Survey (USGS) National Map (`basemap.nationalmap.gov`):** Serves public domain aerial/satellite orthoimagery (`USGSImageryOnly`) and topographic contour (`USGSTopo`) tiles for map viewing and offline pack downloads. (Privacy Policy: https://www.doi.gov/privacy)
- **OpenStreetMap (`tile.openstreetmap.org`):** Serves community map tiles during interactive viewing. Viewed tiles are cached locally on your device to prevent repeat network requests. (Privacy Policy: https://osmfoundation.org/wiki/Privacy_Policy)

**Information Transmitted:** Map tile requests include the requested tile coordinates (`zoom/x/y`), your device’s IP address, timestamp, and standard HTTPS request headers identifying BloodHawk. Map tile coordinates indicate the geographic area currently displayed on your screen. Map providers do not receive your blood drop pins, trail names, notes, or custom markers.

**Disabling Map Connections:** You can prevent all map tile network requests by selecting the **Offline Coordinate Grid (`GRID`)** map layer (which makes zero network requests) or by disabling mobile data and Wi-Fi on your device. Pre-downloading **USGS Offline Map Packs** also provides local tile storage for your hunting area.

**Third-Party Provider Disclaimer:** Hurley Data and Research Consulting LLC does not control and is not responsible for the privacy practices, data retention, or security of third-party map providers (USGS or OpenStreetMap Foundation).

### 3.2 App Distribution, Diagnostics & Updates
When distributed through the Google Play Store, app installation, updates, and platform diagnostics are managed natively by Google LLC under the [Google Privacy Policy](https://policies.google.com/privacy). Google may provide aggregated, de-identified crash logs and performance metrics through the Google Play Console to help us identify and fix software bugs. HDRC does not receive identifying user telemetry from these reports.

---

## 4. Hardware Permissions & Sensor Use

BloodHawk requests only the device permissions strictly necessary for its field tools:

| Permission / Sensor | Purpose in BloodHawk | Network Transmission |
|---|---|---|
| **Camera** (`CAMERA`) | Optical detection of blood droplets on the ground and LED torch control. | **None.** Frames are processed in temporary RAM and discarded immediately. |
| **Location** (`ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`) | Foreground plotting of your position, blood drop pins, and compass vectors. | **None to Developer.** Saved trails remain local. Online map tiles request geographic grid squares from map servers as described in Section 3. |
| **Internet** (`INTERNET`) | Fetching public map tiles when online map layers are enabled. | Transmits standard HTTPS request data to map servers (IP address and requested tile coordinates). |
| **Vibration** (`VIBRATE`) | Haptic pulse alerts when blood droplets are detected. | **None.** Local hardware actuator only. |
| **Compass / Motion Sensors** (Magnetometer & Rotation Vector) | Rendering 360° compass heading on the tracking HUD and map. | Processed locally; saved headings may be included in trail exports. |

---

## 5. Contact & Privacy Inquiries

### 5.1 Inquiries & Correspondence
Hurley Data and Research Consulting LLC accepts support inquiries, general questions, and formal privacy requests by email at HDRConsulting.support@gmail.com or by postal mail:

> **Hurley Data and Research Consulting LLC**  
> Attn: Privacy & Legal Compliance  
> 679 Hickory Circle  
> Wahoo, NE 68066  
> United States of America  
> Email: HDRConsulting.support@gmail.com  

If you contact us, we receive your email address, name, return mailing address, and the contents of your message. We use this information solely to respond to your inquiry and retain correspondence only as long as necessary to address your request and satisfy applicable legal or recordkeeping obligations.

### 5.2 U.S. and International Inquiries
Whether you reside in the United States, the European Economic Area, the United Kingdom, Switzerland, or elsewhere:
- Data generated within BloodHawk remains in your direct custody and control on your handset.
- You can view, export, or delete your local recovery trails, custom profiles, and map caches directly inside the app at any time.
- If you have questions regarding this policy or any correspondence records held by Hurley Data and Research Consulting LLC, you may contact us by email at HDRConsulting.support@gmail.com or by mail at the address above. We respond to privacy requests within the time required by applicable law.

---

## 6. Children's Privacy & Policy Updates

- **Intended Audience:** BloodHawk is intended for adults (18 years of age and older) engaged in lawful hunting and wildlife recovery. The app is not directed to children, and Hurley Data and Research Consulting LLC does not knowingly collect personal information from children.
- **Policy Changes:** We review and update this Privacy Policy as application features or distribution channels evolve. Any updates will be published with a revised effective date. Your continued use of the App following the posting of an updated Privacy Policy constitutes your acceptance of the updated terms.
