# services.backgroud_music
Using services to play music in background in all activities!
How to play the same BACKGROUND MUSIC across all activities:
1)	Create MusicService.class for our SERVICE:


public class MusicService extends Service implements MediaPlayer.OnErrorListener {

    private final IBinder mBinder = new ServiceBinder();
    MediaPlayer mPlayer;
    private int length = 0;

    public MusicService() {
    }

    public class ServiceBinder extends Binder {
        MusicService getService() {
            return MusicService.this;
        }
    }

    @Override
    public IBinder onBind(Intent arg0) {
        return mBinder;
    }

    @Override
    public void onCreate() {
        super.onCreate();

        mPlayer = MediaPlayer.create(this, R.raw.slow_shock);
        mPlayer.setOnErrorListener(this);

        if (mPlayer != null) {
            mPlayer.setLooping(true);
            mPlayer.setVolume(50, 50);
        }


        mPlayer.setOnErrorListener(new MediaPlayer.OnErrorListener() {

            public boolean onError(MediaPlayer mp, int what, int
                    extra) {

                onError(mPlayer, what, extra);
                return true;
            }
        });
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (mPlayer != null) {
            mPlayer.start();
        }
        return START_NOT_STICKY;
    }

    public void pauseMusic() {
        if (mPlayer != null) {
            if (mPlayer.isPlaying()) {
                mPlayer.pause();
                length = mPlayer.getCurrentPosition();
            }
        }
    }

    public void resumeMusic() {
        if (mPlayer != null) {
            if (!mPlayer.isPlaying()) {
                mPlayer.seekTo(length);
                mPlayer.start();
            }
        }
    }

    public void startMusic() {
        mPlayer = MediaPlayer.create(this, R.raw.slow_shock);
        mPlayer.setOnErrorListener(this);

        if (mPlayer != null) {
            mPlayer.setLooping(true);
            mPlayer.setVolume(50, 50);
            mPlayer.start();
        }

    }

    public void stopMusic() {
        if (mPlayer != null) {
            mPlayer.stop();
            mPlayer.release();
            mPlayer = null;
        }
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        if (mPlayer != null) {
            try {
                mPlayer.stop();
                mPlayer.release();
            } finally {
                mPlayer = null;
            }
        }
    }

    public boolean onError(MediaPlayer mp, int what, int extra) {

        Toast.makeText(this, "Music player failed", Toast.LENGTH_SHORT).show();
        if (mPlayer != null) {
            try {
                mPlayer.stop();
                mPlayer.release();
            } finally {
                mPlayer = null;
            }
        }
        return false;
    }
}



2)	Add services in Android Manifest:
<service
    android:name=".MusicService"
    android:enabled="true" />


3)	Setting up your Activity:

a)	 Outside onCreate: Bind/Unbind music service.

private boolean mIsBound = false;
private MusicService mServ;
private ServiceConnection Scon =new ServiceConnection(){

    public void onServiceConnected(ComponentName name, IBinder
            binder) {
        mServ = ((MusicService.ServiceBinder)binder).getService();
    }

    public void onServiceDisconnected(ComponentName name) {
        mServ = null;
    }
};

void doBindService(){
    bindService(new Intent(this,MusicService.class),
            Scon,Context.BIND_AUTO_CREATE);
    mIsBound = true;
}

void doUnbindService()
{
    if(mIsBound)
    {
        unbindService(Scon);
        mIsBound = false;
    }
}



b)	 Inside onCreate: Bind music service


doBindService();
Intent music = new Intent();
music.setClass(this, MusicService.class);
startService(music);


4)	 Resume music and unbind it when we no longer need it:

a)	onResume: Resume Music Service

@Override
protected void onResume() {
    super.onResume();
    
    if (mServ != null) {
        mServ.resumeMusic();
    }

}


b)	onDestroy: Unbind Music Service

@Override
protected void onDestroy() {
    super.onDestroy();

    doUnbindService();
    Intent music = new Intent();
    music.setClass(this,MusicService.class);
    stopService(music);

}

5)	Now BONUS:

  Because we are playing the music with a SERVICE, it will run until we stop it. 
  To better understand,  I will give you example: when the user press, let’s say „Home Button” to exit the app, the music will continuously play, because the SERVICE is still running.
   To avoid this, we need to have in consideration 3 aspects:
-	When user press „Home Button”
-	„Recent Apps Button” or
-	„Power Button”



6)	Create HomeWatcher.class:


public class HomeWatcher {

    //static final String TAG = "hg";
    private Context mContext;
    private IntentFilter mFilter;
    private OnHomePressedListener mListener;
    private InnerRecevier mRecevier;

    public HomeWatcher(Context context) {
        mContext = context;
        mFilter = new IntentFilter(Intent.ACTION_CLOSE_SYSTEM_DIALOGS);
    }

    public void setOnHomePressedListener(OnHomePressedListener listener) {
        mListener = listener;
        mRecevier = new InnerRecevier();
    }

    public void startWatch() {
        if (mRecevier != null) {
            mContext.registerReceiver(mRecevier, mFilter);
        }
    }

    public void stopWatch() {
        if (mRecevier != null) {
            mContext.unregisterReceiver(mRecevier);
        }
    }

    class InnerRecevier extends BroadcastReceiver {
        final String SYSTEM_DIALOG_REASON_KEY = "reason";
        final String SYSTEM_DIALOG_REASON_GLOBAL_ACTIONS = "globalactions";
        final String SYSTEM_DIALOG_REASON_RECENT_APPS = "recentapps";
        final String SYSTEM_DIALOG_REASON_HOME_KEY = "homekey";

        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            if (action != null && action.equals(Intent.ACTION_CLOSE_SYSTEM_DIALOGS)) {
                String reason = intent.getStringExtra(SYSTEM_DIALOG_REASON_KEY);
                if (reason != null) {
                    //Log.e(TAG, "action:" + action + ",reason:" + reason);
                    if (mListener != null) {
                        if (reason.equals(SYSTEM_DIALOG_REASON_HOME_KEY)) {
                            mListener.onHomePressed();
                        } else if (reason.equals(SYSTEM_DIALOG_REASON_RECENT_APPS)) {
                            mListener.onHomeLongPressed();
                        }
                    }
                }
            }
        }
    }

    public interface OnHomePressedListener {
        void onHomePressed();

        void onHomeLongPressed();
    }
}




7)	Add HomeWatcher to our Activity:

Global variable:

HomeWatcher mHomeWatcher;

onCreate:

mHomeWatcher = new HomeWatcher(this);
mHomeWatcher.setOnHomePressedListener(new HomeWatcher.OnHomePressedListener() {
    @Override
    public void onHomePressed() {
        if (mServ != null) {
            mServ.pauseMusic();
        }
    }
    @Override
    public void onHomeLongPressed() {
        if (mServ != null) {
            mServ.pauseMusic();
        }
    }
});
mHomeWatcher.startWatch();



8)	Detect Idle Screen (When user press „Power Button” or when the screen expires and goes idle”

onPause:

@Override
protected void onPause() {
    super.onPause();

    PowerManager pm = (PowerManager)
            getSystemService(Context.POWER_SERVICE);
    boolean isScreenOn = false;
    if (pm != null) {
        isScreenOn = pm.isScreenOn();
    }

    if (!isScreenOn) {
        if (mServ != null) {
            mServ.pauseMusic();
        }
    }

}



9)	Setting up other activities:

For music to play in other activities we just need to repeat everything from step 3)
