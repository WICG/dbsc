<span style="color:red">!!! DRAFT - Please note this page is still in draft as we work on platform specific details
</span>

### Local key helper on Mac

On macOS, a Local Key Helper is a protocol (`LocalKeyHelper.h`), and its implementation (`LocalKeyHelperImpl.h`/`LocalKeyHelperImpl.m`) is yet to be determined.

Each _Local key helper_ vendor will implement and ship as an XPC service on macOS. It will be signed with app sandbox entitlement for security requirement. Each vendor will also manage their own lifecycle of the XPC service depending on their specific and other functionality provided by the service. Each XPC service needs to be defined and registered with launchd as a launch agent. Vendors can decided whether to start XPC service on machine startup or on demand. 

Here is an example of the Local Key Helper XPC service plist:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.contoso.LocalKeyHelper</string>
    <key>Program</key>
    <string>/pathToXcpService/LocalKepHelper.xpc/Contents/MacOS/LocalKepHelper</string>
    <key>KeepAlive</key>
    <true/>
    <key>MachServices</key>
    <dict>
        <key>com.contoso.LocalKeyHelper</key>
        <true/>
    </dict>
</dict>
</plist>
```

Please refer to Apple documentation for building and configuring XPC services.


When the browser (e.g. Chrome) communicates with this service, it needs to establish a service connection first.
Here is an example of starting a new connection with the Local Key Helper:

```objective-c
// Start a new connection by providing the Local Key Helper defined in the plist
NSXPCConnection* localKeyhelper = [[NSXPCConnection alloc] initWithMachServiceName:@"com.contoso.LocalKeyHelper"];

// Make the connection conform to the defined Local Key Helper protocol
[localKeyhelper setRemoteObjectInterface:[NSXPCInterface interfaceWithProtocol:@protocol(LocalKeyHelper)]];
```

The browser (e.g. Chrome) can then call the methods defined in the LocalKeyHelper protocol, and the implementations will be provided by the XPC service.

**Note:**

- It is important to define the LocalKeyHelper protocol exactly the same way in both the browser and the XPC service provider. This allows for easy extension of the protocol in the future (e.g. using a dictionary as an input parameter instead of a strict object type).
- The input and output parameters of the XPC service should follow the `NSSecureCoding` protocol.
- Un-sandboxed applications can communicate directly to the XPC service.

#### Browser discovery process
To inform the browser about the Local Key Helper to use, the manifest file is created within the browser's root folder during the LocalKeyHelper's installation (e.g. `/Library/Google/Chrome/LocalKeyHelpers`). This folder contains the following 2 types of files:

**1st type: files that define the Local Key Helpers**

i.e.
````
com.provider1.LocalKeyHelper.json (if helper was installed)
com.provider2.LocalKeyHelper.json (if helper was installed)
com.provider3.LocalKeyHelper.json (if helper was installed)
````
For example:
````
com.contoso.LocalKeyHelper.json
````

Within the file:

```json
  {
    "provider": "contoso",
      "xpc": // Local key helper type
      {
        "service": "com.contoso.LocalKeyHelper", // Service name/API activation id
        "version": "0.0.1", // Helper service version
        "Idp_resource":"com.contoso.LocalKeyHelper.idP.json" // File path to IdP URL allow-list
      }   
  }
```

**2nd type: files that defines the allow-list for IdP URLs:**

For example:
````
com.contoso.LocalKeyHelper.idP.json
````

```json
  {
    "urls": ["*.contoso1.com", "login.contoso1.net", "login.contoso2.com"],
  }
```

The MDM service can do post script to update the URLs in this file without touching the Local Key Helpers file. 
