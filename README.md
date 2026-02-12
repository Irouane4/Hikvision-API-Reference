# Hikvision API Reference — ISAPI Endpoints & SDK Functions

This document lists all ISAPI endpoints and SDK functions used in the Controle d'accès application.

---

## ISAPI Endpoints (HTTP REST)

ISAPI endpoints use standard HTTP methods (GET, POST, PUT) with JSON payloads. Authentication is handled via Digest auth (username/password).

### 1. Search for Enrolled Users

**Endpoint:** `POST /ISAPI/AccessControl/UserInfo/Search?format=json`

**Purpose:** Retrieve all enrolled users from the device with pagination.

**Request:**
```json
{
  "UserInfoSearchCond": {
    "searchID": "a1b2c3d4",
    "searchResultPosition": 0,
    "maxResults": 50
  }
}
```

**Response:**
```json
{
  "UserInfoSearch": {
    "totalMatches": 150,
    "UserInfo": [
      {
        "employeeNo": "208",
        "name": "test1",
        "userType": "normal",
        "Valid": {
          "enable": true,
          "beginTime": "2026-01-01T00:00:00",
          "endTime": "2036-01-01T00:00:00"
        }
      }
    ]
  }
}
```


---

### 2. Block/Allow User Access

**Endpoint:** `PUT /ISAPI/AccessControl/UserInfo/Modify?format=json`

**Purpose:** Enable or disable a user's access permissions.

#### Allow Access
```json
{
  "UserInfo": {
    "employeeNo": "208",
    "userType": "normal",
    "Valid": {
      "enable": true,
      "beginTime": "2026-02-12T10:06:16",
      "endTime": "2036-02-12T10:06:16",
      "timeType": "local"
    },
    "RightPlan": [
      { "doorNo": 1, "planTemplateNo": "1" }
    ]
  }
}
```

#### Block Access
```json
{
  "UserInfo": {
    "employeeNo": "208",
    "userType": "blackList"
  }
}
```

---

### 3. Get Punch History (Event Search)

**Endpoint:** `POST /ISAPI/AccessControl/AcsEvent?format=json`

**Purpose:** Retrieve historical access events (punches) from the device.

**Request:**
```json
{
  "AcsEventCond": {
    "searchID": "abc12345",
    "searchResultPosition": 0,
    "maxResults": 150,
    "major": 5,
    "minor": 75
  }
}
```

**Parameters:**
- `major: 5` — Access control events
- `minor: 75` — All event types (can filter to specific events like fingerprint, face, card)
- `maxResults` — Max 150 per request (device limit)

**Response:**
```json
{
  "AcsEvent": {
    "totalMatches": 500,
    "InfoList": [
      {
        "employeeNoString": "208",
        "name": "test1",
        "time": "2026-02-12T09:30:45",
        "cardReaderNo": 1,
        "verifyMode": "fingerPrint",
        "currentVerifyMode": "fingerPrint"
      }
    ]
  }
}
```


---

### 4. Get Face Data

**Endpoint:** `POST /ISAPI/Intelligent/FDLib/FDSearch?format=json`

**Purpose:** Retrieve face image data for a user.

**Request:**
```json
{
  "FDSearchDescription": {
    "searchID": "xyz789",
    "FDID": "208"
  }
}
```

**Response:** Face image in base64 format within the JSON.


---

### 5. Get Fingerprint Data

**Endpoint:** `POST /ISAPI/AccessControl/FingerPrintUpload?format=json`

**Purpose:** Retrieve fingerprint template data for a user.

**Request:**
```json
{
  "FingerPrintCfg": {
    "employeeNo": "208"
  }
}
```

**Response:** List of fingerprint templates (binary data encoded).


---

### 6. Create User with Biometrics

**Endpoint:** `PUT /ISAPI/AccessControl/UserInfo/Record?format=json`

**Purpose:** Create a new user on the device with optional face/fingerprint data.

**Request:**
```json
{
  "UserInfo": {
    "employeeNo": "999",
    "name": "New User",
    "userType": "normal",
    "Valid": {
      "enable": true,
      "beginTime": "2026-02-12T00:00:00",
      "endTime": "2036-02-12T00:00:00"
    },
    "RightPlan": [
      { "doorNo": 1, "planTemplateNo": "1" }
    ]
  }
}
```



---

### 7. Delete User from Device

**Endpoint:** `PUT /ISAPI/AccessControl/UserInfo/Delete?format=json`

**Purpose:** Remove a user and all their biometric data (face, fingerprint) from the device entirely.

**Request:**
```json
{
  "UserInfoDelCond": {
    "EmployeeNoList": [
      { "employeeNo": "208" }
    ]
  }
}
```

**Response:** `200 OK` on success.

**Notes:**
- Accepts multiple employees in `EmployeeNoList` for batch deletion
- Removes user profile + all associated biometrics (face, fingerprints)
- Action is irreversible — user must be re-enrolled on the device


---

### 8. SDK-Tunneled ISAPI (Attempted for Log Clearing)

**Endpoint:** Various (tested via `NET_DVR_STDXMLConfig`)

These were attempted but **not supported** by the device:
- `PUT /ISAPI/AccessControl/AcsEvent/Delete`
- `DELETE /ISAPI/AccessControl/AcsEvent`
- `PUT /ISAPI/AccessControl/AcsEvent/StorageClear`

**Result:** Device returned `statusCode: 4 - Invalid Operation`



---

## SDK Functions (HCNetSDK.dll)

SDK functions are called via P/Invoke from the native `HCNetSDK.dll` library. These use binary TCP protocol on port 8000.

### 1. Initialize SDK

```csharp
[DllImport("HCNetSDK.dll")]
public static extern bool NET_DVR_Init();
```

**Purpose:** Initialize the SDK library (called once at startup).


---

### 2. Cleanup SDK

```csharp
[DllImport("HCNetSDK.dll")]
public static extern bool NET_DVR_Cleanup();
```

**Purpose:** Clean up SDK resources (called on app exit).


---

### 3. Login to Device

```csharp
[DllImport("HCNetSDK.dll")]
public static extern int NET_DVR_Login_V40(
    ref NET_DVR_USER_LOGIN_INFO loginInfo,
    ref NET_DVR_DEVICEINFO_V40 deviceInfo
);
```

**Parameters:**
- `loginInfo` — IP, port, username, password
- `deviceInfo` — Output struct with device model, serial number, etc.

**Returns:** User ID (positive integer) on success, -1 on failure.


---

### 4. Logout

```csharp
[DllImport("HCNetSDK.dll")]
public static extern bool NET_DVR_Logout(int userId);
```


---

### 5. Setup Alarm Channel (Real-time Events)

```csharp
[DllImport("HCNetSDK.dll")]
public static extern int NET_DVR_SetupAlarmChan_V41(
    int userId,
    ref NET_DVR_SETUPALARM_PARAM setupParam
);
```

**Purpose:** Subscribe to real-time events (punches, access denials, etc.) via callback.

**Returns:** Alarm handle (positive integer) on success.


---

### 6. Close Alarm Channel

```csharp
[DllImport("HCNetSDK.dll")]
public static extern bool NET_DVR_CloseAlarmChan_V30(int alarmHandle);
```

---

### 7. Set Alarm Message Callback

```csharp
[DllImport("HCNetSDK.dll")]
public static extern bool NET_DVR_SetDVRMessageCallBack_V50(
    int callbackType,
    MessageCallback callback,
    IntPtr userData
);

public delegate bool MessageCallback(
    int command,
    IntPtr alarmInfo,
    IntPtr alarmBuf,
    int bufLen,
    IntPtr userData
);
```

**Purpose:** Register a C# callback function to receive real-time events.

**Callback Command:** `0x5002` — Access control event (fingerprint, face, card)

---

### 8. Get Last Error

```csharp
[DllImport("HCNetSDK.dll")]
public static extern uint NET_DVR_GetLastError();
```

**Returns:** Error code (e.g., 7 = connection timeout, 1 = incorrect password).


---

### 9. SDK-Tunneled ISAPI Commands

```csharp
[DllImport("HCNetSDK.dll")]
public static extern bool NET_DVR_STDXMLConfig(
    int userId,
    IntPtr inputParam,
    IntPtr outputParam
);
```

**Purpose:** Send ISAPI commands through the SDK's authenticated TCP connection (bypasses HTTP auth).

**Structs:**
```csharp
[StructLayout(LayoutKind.Sequential)]
public struct NET_DVR_XML_CONFIG_INPUT
{
    public uint dwSize;
    public IntPtr lpRequestUrl;     // e.g., "GET /ISAPI/AccessControl/UserInfo/Count"
    public uint dwRequestUrlLen;
    public IntPtr lpInBuffer;       // XML/JSON body
    public uint dwInBufferSize;
    public uint dwRecvTimeOut;
    public byte[] byRes;
}

[StructLayout(LayoutKind.Sequential)]
public struct NET_DVR_XML_CONFIG_OUTPUT
{
    public uint dwSize;
    public IntPtr lpOutBuffer;      // Response XML/JSON
    public uint dwOutBufferSize;
    public uint dwReturnedXMLSize;
    public IntPtr lpStatusBuffer;   // Error messages
    public uint dwStatusSize;
    public byte[] byRes;
}
```

---

## Summary Table

| Feature | Protocol | Endpoint/Function | Request Type |
|---|---|---|---|
| **View enrolled users** | ISAPI | `/AccessControl/UserInfo/Search` | POST |
| **Block/Allow user** | ISAPI | `/AccessControl/UserInfo/Modify` | PUT |
| **Get punch history** | ISAPI | `/AccessControl/AcsEvent` | POST |
| **Export faces** | ISAPI | `/Intelligent/FDLib/FDSearch` | POST |
| **Export fingerprints** | ISAPI | `/AccessControl/FingerPrintUpload` | POST |
| **Create user** | ISAPI | `/AccessControl/UserInfo/Record` | PUT |
| **Delete user** | ISAPI | `/AccessControl/UserInfo/Delete` | PUT |
| **Real-time events** | SDK | `NET_DVR_SetupAlarmChan_V41` | Native |
| **Login** | SDK | `NET_DVR_Login_V40` | Native |
| **Logout** | SDK | `NET_DVR_Logout` | Native |
| **Clear logs** | ❌ Not supported | N/A | N/A |
| **Send fingerprint templates** | ❌ Not supported | N/A | N/A |

---

## Authentication

- **ISAPI:** HTTP Digest authentication (username/password sent with each request)
- **SDK:** Login once with `NET_DVR_Login_V40`, then all future commands use the `userId` session handle
