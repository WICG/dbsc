<span style="color:red">!!! DRAFT - Please note this page is still in draft as we work on platform specific details
</span>

### Local Key Helper on Android
On android platform a Local Key Helper is a protocol interface (`ILocalKeyHelper.java`), and its implementation (`LocalKeyHelperImpl.java`) is yet to be determined.

Local KeyHelpers on the Android platform are implemented using [content providers](https://developer.android.com/guide/topics/providers/content-providers). These content providers allow browser applications, such as Chrome, to interact with them by querying and accessing specific APIs defined by the Local KeyHelpers.

#### Content Provider Authority
Each Local KeyHelper defines a content provider with a authority such as `com.contoso.localkeyhelper`. Different APIs are implemented as separate paths on this content provider.

#### Browser Interaction with Local KeyHelpers
To interact with Local KeyHelpers, browser applications must define a `<provider>` inside the `<queries>` tag in their manifest file with the same authority `com.contoso.localkeyhelper`. This allows the browser to query the Local KeyHelpers.

##### Example Manifest Entry
```xml
<queries>
    <provider android:authorities="com.contoso.localkeyhelper" />
</queries>
```   
To get the installed applications that provide a content provider with authority "com.contoso.localkeyhelper" and select a specific package "com.contoso.localkeyhelper.entra" to query the content provider, you can use the following Java code:

```java
PackageManager packageManager = getPackageManager();
List<PackageInfo> installedPackages = packageManager.getInstalledPackages(PackageManager.GET_PROVIDERS);

for (PackageInfo packageInfo : installedPackages) {
    ProviderInfo[] providers = packageInfo.providers;
    if (providers != null) {
        for (ProviderInfo provider : providers) {
            if ("com.contoso.localkeyhelper".equals(provider.authority)) {
                // select specific package to query
            }
        }
    }
}
```


#### Implementing the ContentProvider in Local KeyHelper
Each Local KeyHelper must implement a ContentProvider to handle the interactions. Below is an example of how to implement a ContentProvider in a Local KeyHelper.

##### Example ContentProvider Implementation

```java
public class LocalKeyHelperProvider extends ContentProvider {
    public static final String AUTHORITY = "com.contoso.localkeyhelper";
    private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        uriMatcher.addURI(AUTHORITY, "keyhelper1/api1", 1);
        // Add more paths as needed
    }

    @Override
    public boolean onCreate() {
        return true;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
        switch (uriMatcher.match(uri)) {
            // call the Implementation specific api based on the path
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }
}
```

#### Browser Interaction with the ContentProvider
Browser applications can interact with the Local KeyHelper's ContentProvider by querying it. Below is an example of how a browser application can query the ContentProvider.

##### Example Code to Query the ContentProvider
    
```java
Uri uri = Uri.parse("content://com.contoso.localkeyhelper/keyhelper1/api1");
Cursor cursor = getContentResolver().query(uri, null, null, null, null);
if (cursor != null) {
    try {
        if (cursor.moveToFirst()) {
            // Read the data from the cursor
        }
    } finally {
        cursor.close();
    }
}
```