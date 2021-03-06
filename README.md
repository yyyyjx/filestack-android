# filestack-android

Android UI and upload API's for Filestack. Easily upload files from a user's
local device or cloud services. Provides both a picker activity for easy
integration as well as a client class for manual control and customization.
Supports Facebook, Instagram, Google Drive, Dropbox, Box, GitHub, Gmail, Google
Photos, Microsoft OneDrive, and Amazon Drive.

*The 2.0 version of this library is just being released as of November 2017.*
*Expect that the API and setup process may change somewhat.*
*We're aware of some missing basic features that will be added soon.*

New features:
- Built on new base Java SDK and updated backend API's
- Multipart uploads and Filestack Intelligent Ingestion support
- New UI as well as necessary components to create completely custom UI
- Background upload and notification support

![Demo Screen Recording][screen-recording]

## Including In Your Project

```gradle
compile 'com.filestack:filestack-android:2.0.0-alpha.2'
```

## Usage

*For a working implementation of this project see the `demo` folder.*

### Launch activity
```java
// Create an intent to launch FsActivity
Intent intent = new Intent(this, FsActivity.class);

// Create a config object with your account settings
// Using security (policy and signature) is optional
Config config = new Config("API_KEY", "RETURN_URL", "POLICY", "SIGNATURE");
intent.putExtra(FsConstants.EXTRA_CONFIG, config);

// Setting storage options is also optional
// We'll default to Filestack S3 if unset
StorageOptions storeOpts = new StorageOptions();
intent.putExtra(FsConstants.EXTRA_STORE_OPTS, storeOpts);

// To manually handle uploading, set auto upload to false
// You can upload the user's selections yourself with the Client class
intent.putExtra(FsConstants.EXTRA_AUTO_UPLOAD, false);

// To customize the sources list, pass in a list of constants
// The sources will appear in the order you add them to the list
// Defaults to Camera, Device, Google Drive, Facebook, Instagram, and Dropbox
ArrayList<String> sources = new ArrayList<>();
sources.add(Sources.CAMERA);
sources.add(Sources.DEVICE);
sources.add(Sources.GOOGLE_DRIVE);
sources.add(Sources.GITHUB);
intent.putExtra(FsConstants.EXTRA_SOURCES, sources);

// Start the activity
startActivityForResult(intent, REQUEST_FILESTACK);
```

### Setup for cloud authorization
When the user authorizes their cloud account, they do so inside the default
device browser, not a `WebView`. Therefore you must set your app to respond to a
URL redirect so that the app reopens after the authorization. This requires two
components, an intent filter to respond to the URL, and an entry activity that
opens for the intent. The entry activity is necessary to maintain the UI state
and activity stack. For more information about opening an app by URL, see the
Google documentation on [App Links][app-links].

EntryActivity.java:
```java
public class EntryActivity extends AppCompatActivity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // Check to see if this Activity is the root activity
    if (isTaskRoot()) {
      // This Activity is the only Activity, so
      //  the app wasn't running. So start the app from the
      //  beginning (redirect to MainActivity)
      Intent mainIntent = getIntent(); // Copy the Intent used to launch me
      // Launch the real root Activity (launch Intent)
      mainIntent.setClass(this, MainActivity.class);
      // I'm done now, so finish()
      startActivity(mainIntent);
      finish();
    } else {
      // App was already running, so just finish, which will drop the user
      //  in to the activity that was at the top of the task stack
      finish();
    }
  }
}
```

Inside AndroidManifest.xml:
```xml
<activity
  android:name=".EntryActivity">

  <intent-filter android:label="@string/app_name">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <!-- Accepts URIs that begin with "example://gizmos” -->
    <data android:scheme="filestackdemo"
      android:host="auth" />
    </intent-filter>

  </activity>
```

## Receiving selected items
`FsActivity` returns immediately once a user selects files. The returned
response will always be an `ArrayList` of `Selection` objects. Receive them in
your calling activity like so:

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
  super.onActivityResult(requestCode, resultCode, data);

  if (requestCode == REQUEST_FILESTACK && resultCode == RESULT_OK) {
    Log.d("result", "returned");
    Serializable extra = data.getSerializableExtra(FsConstants.EXTRA_SELECTION_LIST);
    ArrayList<Selection> selections = (ArrayList<Selection>) extra;
    for (Selection selection : selections) {
      Log.i("filestackSelection", selection.getProvider() + " " + selection.getName());
    }
  }
}
```

## Receiving upload status
Because the actual uploading occurs in a background service, we need to
register a `BroadcastReceiver` to get a status and resultant `FileLink` for
each selection. When the picker returns to `onActivityResult()` you receive an
`ArrayList` of `Selection` objects. When an intent message is received in your
`BroadcastReceiver`, you will receive a status string, a `Selection` (matching  
one in the list), and a `FileLink` (if the upload succeeded). As the upload
progresses, the background service will also put up notifications about its
ongoing status.

UploadStatusReceiver.java:
```java
public class UploadStatusReceiver extends BroadcastReceiver {
  @Override
  public void onReceive(Context context, Intent intent) {
    String status = intent.getStringExtra(FsConstants.EXTRA_STATUS);
    Selection selection = (Selection) intent.getSerializableExtra(FsConstants.EXTRA_SELECTION);
    FileLink fileLink = (FileLink) intent.getSerializableExtra(FsConstants.EXTRA_FILE_LINK);

    Log.i("uploadStatus", selection.getName() + " " + status + " "
    + (fileLink != null ? fileLink.getHandle() : ""));
  }
}
```

Register the receiver in your calling activity's `onCreate()`:
```java
IntentFilter intentFilter = new IntentFilter(FsConstants.BROADCAST_UPLOAD);
UploadStatusReceiver receiver = new UploadStatusReceiver();
LocalBroadcastManager.getInstance(this).registerReceiver(receiver, intentFilter);
```

## Custom UI using client
The [base Java SDK][java-sdk] contains a `Client` class that can be used to
perform both local uploads and cloud integrations. To use it, you would create
a your config as mentioned above, then create your client with it. If you want,
you can use the client and get all the same functionality without any of the
Filestack UI. Reference doc for the `Client` class can be found
[here][java-sdk-ref].

[screen-recording]: /demo/media/recording.gif
[app-links]: https://developer.android.com/training/app-links/index.html
[java-sdk]: https://github.com/filestack/filestack-java
[java-sdk-ref]: https://filestack.github.io/filestack-java/
