<span style="color:red">!!! DRAFT - Please note this page is still in draft as we work on platform specific details
</span>

### Local Key Helper on Android
On android platform a Local Key Helper is a protocol interface (`ILocalKeyHelper.java`), and its implementation (`LocalKeyHelperImpl.java`) is yet to be determined.

Local KeyHelpers on the Android platform are implemented using [content providers](https://developer.android.com/guide/topics/providers/content-providers). These content providers allow browser applications, such as Chrome, to interact with them by querying and accessing specific APIs defined by the Local KeyHelpers.

#### Registering LocalkeyHelpers with Browser Interaction
To register the localkeyhelpers with a browser (e.g. Chrome), the browser application exports a content provider with the authority `$browserPackageName.localkeyhelpersregistration` in it's manifest file. 

```xml
<provider
    android:authorities="com.custombrowser.localkeyhelpersregistration"
    android:name="LocalKeyHelpersRegistrationProvider"
    android:exported="true" >
</provider>
```

##### Implementing localkeyhelpersregistration provider in Browser
Browser application implements `localkeyhelpersregistration` content provider and provides path `/register` for the local key helpers to register themselves.
When the LocalKeyHelper queries the browser's content provider, it register itself with the browser.
Example implementation.
``` java
public class LocalKeyHelperConsumerProvider extends ContentProvider {
    public static final String AUTHORITY = "com.custombrowser.localkeyhelpersregistration";
    private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        uriMatcher.addURI("AUTHORITY", "/register", 1);
    }

    @Override
    public boolean onCreate() {
        return true;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
        switch (uriMatcher.match(uri)) {
            case 1 :
                return new MatrixCursor(new String[0]) {
                    @Override
                    public Bundle getExtras() {
                        Bundle bundle = new Bundle();
                        bundle.putBoolean("registered", true);
                        return bundle;
                    }
                };
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }
    ...

}
```
##### Calling Browser Content Provider to register a local key helper. 
LocalKeyhelpers make the browser content provider visible to them by adding queries tag with browser content provider authority to their manifest file.
```xml
<queries>
    <provider android:authorities="com.custombrowser.localkeyhelpersregistration" />
</queries>
```

LocalKeyHelper call the browser's content provider to register itself with the browser.
```java
Uri uri = Uri.parse("content://com.custombrowser.localkeyhelpersregistration/register");
Cursor cursor = getContentResolver().query(uri, null, null, null, null);
if (cursor != null) {
    try {
        val resultBundle = cursor.extras
        if (resultBundle != null) {
            val registered = resultBundle.getBoolean("registered")
            Log.i("TAG", "Registered: $registered")
        } else {
            Log.i("TAG", "Failed to register.")
        }
    } finally {
        cursor.close();
    }
}
``` 

#### Calling Local KeyHelpers from Browser
Once a localkeyhelper is registered with the  Browser application, the browser can query the Local KeyHelper's content providers without defining the query tag.
To implement the localkeyhelper protocol interface LocalKeyHelper defines a content provider with the authority `$packagename.**localkeyhelper**`. Different APIs are implemented as separate paths on this content provider.

##### Sample Manifest Entry for Local KeyHelper Content Provider
```xml
<provider
    android:authorities="com.contoso.localkeyhelper"
    android:name="ContosoLocalKeyHelperProvider" />
```   

##### Implementing the ContentProvider in Local KeyHelper
Each Local KeyHelper must implement a ContentProvider to handle the interactions. Below is an example of how to implement a ContentProvider in a Local KeyHelper.

```java
public class LocalKeyHelperProvider extends ContentProvider {
    public static final String AUTHORITY = "com.contoso.localkeyhelper";
    private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        uriMatcher.addURI(AUTHORITY, "sample_api_path", 1);
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