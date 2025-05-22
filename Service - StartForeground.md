# Android Service 前台服務 StartForeground
官方網址: https://developer.android.com/about/versions/14/changes/fgs-types-required

一般的Service容易被系統殺掉，利用前台服務可以預防系統被殺掉。  
使用前台服務必須指定適當的前景服務類型，如以下

* camera
* connectedDevice
* dataSync
* health
* location
* mediaPlayback
* mediaProjection
* microphone
* phoneCall
* remoteMessaging
* shortService
* specialUse
* systemExempted

Android 14 新增 `health`, `remoteMessaging`, `shortService`, `specialUse` 和 `systemExempted` 類型。  

## 用`location`搭配前景(台)服務作範例  
* 先增新一個class名字叫做`MainService`
    
若是有用`location`需在`AndroidManifest.xml`補上以下權限
```
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
```

需要前景(台)服務，需在`AndroidManifest.xml`中，`<application`上方補上以下權限，以及在`<application`中的補上`<service`。
```
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />

    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rule"
        ...

        <service
            android:name=".MainService"
            android:foregroundServiceType="location"
            android:exported="false"/>
```

* `MainService`的前景(台)服務範例
* 和一般服務不同的是需要`Notification`和`public int onStartCommand(Intent intent, int flags, int serviceId)`
```

public class MainService extends Service{
    public static final String TAG = "MainService";
    static final String NOTIFICATION_CHANNEL_ID = "example_channel";
    static final String NOTIFICATION_CHANNEL_NAME = "Example"
    @VisibleForTesting static final int NOTIFICATION_ID_PROGRESS = 1;
    @VisibleForTesting NotificationManager notificationManager;
    @VisibleForTesting ForegroundManager foregroundManager;
    private int mMainService;

    public MainService(){
    }

    @Override
    public void onCreate(){
        super.onCreate();
        Log.v(TAG, "onCreate");

        if(notificationManager == null){
            notificationManager = getSystemService(NotificationManager.class);
        }

        if(foregroundManager == null){
            foregroundManager = createForegroundManager(this);
        }

        setUpNotificationChannel();
        startForegroundService();
    }

    @Override
    public void onDestroy(){
        super.onDestroy();
        Log.v(TAG, "onDestroy");
        Log.d(TAG, "Shutting down. Last serviceId was " + mMainService);
        LocationHelper.getInstance(this).stopLocationUpdates();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int serviceId){
        Log.v(TAG, "onStartCommand: " + "serviceId " + serviceId);

        locationManager = (LocationManager) context.getSystemService(Context.LOCATION_SERVICE);
        boolean providerEnabled = locationManager.isProviderEnabled(LocationManager.GPS_PROVIDER);
        if(providerEnabled){
            locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1000, 0, locationListener);
        }
        
        startWorking();

        mMainService = serviceId;

        return START_STICKY;
    }

        private void startForegroundService(){
        NotificationCompat.Builder builder = new NotificationCompat.Builder(this, NOTIFICATION_CHANNEL_ID)
                .setContentTitle(NOTIFICATION_TITLE)
                .setContentText(NOTIFICATION_TEXT)
                .setSmallIcon(R.drawable.example_24)
                .setPriority(Notification.PRIORITY_MAX);

        Notification notification = builder.build();
        foregroundManager.startForeground(NOTIFICATION_ID_PROGRESS, notification);
    }

    @Override
    public IBinder onBind(Intent intent){
        return null;
    }

    private static ForegroundManager createForegroundManager(final Service service){
        return new ForegroundManager(){
            @Override
            public void startForeground(int id, Notification notification){
                service.startForeground(id, notification);
            }

            @Override
            public void stopForeground(boolean removeNotification){
                service.stopForeground(removeNotification);
            }
        };
    }

    interface ForegroundManager{
        void startForeground(int id, Notification notification);
        void stopForeground(boolean removeNotification);
    }

    private void startWorking(){

        //Write your work.

    }

    private final LocationListener locationListener = new LocationListener() {
        @Override
        public void onLocationChanged(@NonNull Location location) {
            Log.v(TAG, "onLocationChanged");
            Log.d(TAG, "Latitude: " + location.getLatitude() + " ,Longitude: " + location.getLongitude());
        }

        @Override
        public void onStatusChanged(String provider, int status, Bundle extras) {}

        @Override
        public void onProviderEnabled(@NonNull String provider) {}

        @Override
        public void onProviderDisabled(@NonNull String provider) {}
    };
```

* 從`MainActivity.java`啟動Service
```
public class MainActivity extends AppCompatActivity {
    public static final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        EdgeToEdge.enable(this);
        setContentView(R.layout.activity_main);
        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main), (v, insets) -> {
            Insets systemBars = insets.getInsets(WindowInsetsCompat.Type.systemBars());
            v.setPadding(systemBars.left, systemBars.top, systemBars.right, systemBars.bottom);
            return insets;
        });

       startService(new Intent(MainActivity.this, MainService.class)); // 補上啟動MainService.class程式碼
    }

    @Override
    protected void onDestroy(){
        super.onDestroy();
        Log.v(TAG, "onDestroy()");
    }

    @Override
    protected void onResume(){
        super.onResume();
        Log.v(TAG, "onResume");
    }
```
