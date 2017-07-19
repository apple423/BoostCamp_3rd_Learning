# 안드로이드 3주차 커리큘럼

안드로이드의 서비스와 브로드캐스트리서버 등을 살펴보고 위치와 구글맵을 활용하는 방법을 살펴본다. 그리고 개발시 자주 사용하는 주요 외부 라이브러리를 살펴본다.

## 1. 서비스 살펴보기
 - ## 서비스의 이해  
 서비스는 백그라운드에서 오래 실행되는 작업을 수행할 수 있는 애플리케이션 구성 요소이며 사용자 인터페이스를 제공하지 않습니다. 또 다른 애플리케이션 구성 요소가 서비스를 시작할 수 있으며, 이는 사용자가 또 다른 애플리케이션으로 전환하더라도 백그라운드에서 계속해서 실행됩니다.  

 - ## 서비스 생명주기 메소드
    ```java
    public class ExampleService extends Service {
    int mStartMode;       // indicates how to behave if the service is killed
    IBinder mBinder;      // interface for clients that bind
    boolean mAllowRebind; // indicates whether onRebind should be used

    @Override
    public void onCreate() {
        // The service is being created
    }
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // The service is starting, due to a call to startService()
        return mStartMode;
    }
    @Override
    public IBinder onBind(Intent intent) {
        // A client is binding to the service with bindService()
        return mBinder;
    }
    @Override
    public boolean onUnbind(Intent intent) {
        // All clients have unbound with unbindService()
        return mAllowRebind;
    }
    @Override
    public void onRebind(Intent intent) {
        // A client is binding to the service with bindService(),
        // after onUnbind() has already been called
    }
    @Override
    public void onDestroy() {
        // The service is no longer used and is being destroyed
    }
    }
  
    ```
    - ### onStartCommand() :
    시스템이 이 메서드를 호출하는 것은 또 다른 구성 요소(예: 액티비티)가 서비스를 시작하도록 요청하는 경우입니다.  
    이때 startService()를 호출하는 방법을 씁니다.  
  
    - ### onBind() :
    시스템이 이 메서드를 호출하는 것은 또 다른 구성 요소가 해당 서비스에 바인드되고자 하는 경우 (예를 들어 RPC를 수행하기 위해)입니다.  
    이때 bindService()를 호출하는 방법을 씁니다.  
    - ### onCreate() :  
    시스템이 이 메서드를 호출하는 것은 서비스가 처음 생성되어 일회성 설정 절차를 수행하는 경우입니다(onStartCommand() 또는 onBind()를 호출하기 전에)  
    - ### onDestroy() :
    시스템이 이 메서드를 호출하는 것은 해당 서비스를 더 이상 사용하지 않고 소멸시키는 경우입니다.  
  
- ## 서비스 실행 및 중지
   - ### 서비스의 실행 :
   액티비티나 다른 구성 요소에서 서비스를 시작하려면 Intent(시작할 서비스를 지정)를 **startService()**에 전달하면 됩니다.
  
    ```java
   
    Intent intent = new Intent(this, HelloService.class);
    startService(intent);
  
    ```
   - ### 바인드서비스의 실행 :  
  바인드된 서비스는 애플리케이션 구성 요소가 자신에게 바인드될 수 있도록 허용하는 서비스이며,
  이때 bindService()를 호출하여 오래 지속되는 연결을 생성합니다.  
  바인드된 서비스를 생성하려면 가장 먼저 해야 할 일은 클라이언트가 서비스와 통신할 수 있는 방법을 나타내는 인터페이스를 정의하는 것입니다. 
  서비스와 클라이언트 사이에서 쓰이는 이 인터페이스는 반드시 IBinder의 구현이어야 하며 이를 서비스가 onBind() 콜백 메서드에서 반환해야 합니다.  
  ### (ex) LocalService  
  ```java
  
    public class LocalService extends Service {
    // Binder given to clients
    private final IBinder mBinder = new LocalBinder();
    // Random number generator
    private final Random mGenerator = new Random();

    /**
     * Class used for the client Binder.  Because we know this service always
     * runs in the same process as its clients, we don't need to deal with IPC.
     */
    public class LocalBinder extends Binder {
        LocalService getService() {
            // Return this instance of LocalService so clients can call public methods
            return LocalService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    /** method for clients */
    public int getRandomNumber() {
      return mGenerator.nextInt(100);
    }
   }
  
   ```  

   ### (ex) BindingActivity
   ```java
   public class BindingActivity extends Activity {
    LocalService mService;
    boolean mBound = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
    }

    @Override
    protected void onStart() {
        super.onStart();
        // Bind to LocalService
        Intent intent = new Intent(this, LocalService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        // Unbind from the service
        if (mBound) {
            unbindService(mConnection);
            mBound = false;
        }
    }

    /** Called when a button is clicked (the button in the layout file attaches to
      * this method with the android:onClick attribute) */
    public void onButtonClick(View v) {
        if (mBound) {
            // Call a method from the LocalService.
            // However, if this call were something that might hang, then this request should
            // occur in a separate thread to avoid slowing down the activity performance.
            int num = mService.getRandomNumber();
            Toast.makeText(this, "number: " + num, Toast.LENGTH_SHORT).show();
        }
    }

    /** Defines callbacks for service binding, passed to bindService() */
    private ServiceConnection mConnection = new ServiceConnection() {

        @Override
        public void onServiceConnected(ComponentName className,
                IBinder service) {
            // We've bound to LocalService, cast the IBinder and get LocalService instance
            LocalBinder binder = (LocalBinder) service;
            mService = binder.getService();
            mBound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName arg0) {
            mBound = false;
        }
    };
    }
   
   ```
   - ### 서비스의 중지 :
   시작된 서비스는 자신만의 수명 주기를 직접 관리해야 합니다.  
   다시 말해, 시스템이 서비스를 중단하거나 소멸시키지 않는다는 뜻입니다.  
   다만 시스템 메모리를 회복해야 하고 서비스가 onStartCommand() 반환 후에도 계속 실행되는 경우는 예외입니다.  
   따라서, 서비스는 __stopSelf()__를 호출하여 스스로 중지시켜야 하고, 아니면 다른 구성 요소가 __stopService()__를 호출하여 이를 중지시킬 수 있습니다.
   
   - ### 바인드서비스의 중지 :
   클라이언트가 서비스와의 상호작용을 완료하면 이는 **unbindService()**를 호출하여 바인딩을 해제합니다.   
   
  
## 2. 브로드캐스트리시버 살펴보기

- 브로드캐스트리시버의 이해
- 브로드캐스트리시버 등록 및 제거
- 브로드캐스트리시버를 활용한 SMS 수신

## 3. 위치 및 구글 맵 살펴보기

- 마커 등록 및 삭제
- 커스텀 마커 등록
- 현재 위치 파악
- 근접 경보 설정
- 사용자 이동 위치 실시간 표시

## 4.외부 주요 라이브러리

- 피카소(Picasso)와 그라이드(Glide)
- 토스티(Toasty)
- 서클이미지뷰(CircleImageView)
- 리트로핏2(Retrofit2)
- 지슨(Gson)
- 버터나이프(Butter Knife)
- 켄번스뷰(KenBurnsView)
- 대거(Dagger)
- 코틀린(Kotlin)
