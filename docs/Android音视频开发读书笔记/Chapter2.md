### 第2章  常见的系统播放器MediaPlayer

#### 2.2.2 创建过程

android_media_MediaPlayer.cpp函数**android_media_MediaPlayer_native_init**

```cpp
// This function gets some field IDs, which in turn causes class initialization.
// It is called from a static block in MediaPlayer, which won't run until the
// first time an instance of this class is used.
static void
android_media_MediaPlayer_native_init(JNIEnv *env)
//一个万能指针表，通过操作符（->）访问JNI中的函数
{
    jclass clazz; //类的句柄

    clazz = env->FindClass("android/media/MediaPlayer");
    //通过Native层调用JAVA层，获取MediaPlayer对象
    if (clazz == NULL) {
        return;
    }

    fields.context = env->GetFieldID(clazz, "mNativeContext", "J");
    //获取成员变量mNativeContext, 它在MediaPlayer.java中是一个long型整
    //数，实际是一个内存地址
    if (fields.context == NULL) {
        return;
    }

    fields.post_event = env->GetStaticMethodID(clazz, "postEventFromNative",
                                               "(Ljava/lang/Object;IIILjava/lang/Object;)V");
    if (fields.post_event == NULL) {
        return;
    }

    fields.surface_texture = env->GetFieldID(clazz, "mNativeSurfaceTexture", "J");
    if (fields.surface_texture == NULL) {
        return;
    }

    env->DeleteLocalRef(clazz);

    clazz = env->FindClass("android/net/ProxyInfo");
    if (clazz == NULL) {
        return;
    }

    fields.proxyConfigGetHost =
        env->GetMethodID(clazz, "getHost", "()Ljava/lang/String;");

    fields.proxyConfigGetPort =
        env->GetMethodID(clazz, "getPort", "()I");

    fields.proxyConfigGetExclusionList =
        env->GetMethodID(clazz, "getExclusionListAsString", "()Ljava/lang/String;");

    env->DeleteLocalRef(clazz);

    gBufferingParamsFields.init(env);

    // Modular DRM
    FIND_CLASS(clazz, "android/media/MediaDrm$MediaDrmStateException");
    if (clazz) {
        GET_METHOD_ID(gStateExceptionFields.init, clazz, "<init>", "(ILjava/lang/String;)V");
        gStateExceptionFields.classId = static_cast<jclass>(env->NewGlobalRef(clazz));

        env->DeleteLocalRef(clazz);
    } else {
        ALOGE("JNI android_media_MediaPlayer_native_init couldn't "
              "get clazz android/media/MediaDrm$MediaDrmStateException");
    }

    gPlaybackParamsFields.init(env);
    gSyncParamsFields.init(env);
    gVolumeShaperFields.init(env);
}
```


```java
    /*
     * Called from native code when an interesting event happens.  This method
     * just uses the EventHandler system to post the event back to the main app thread.
     * We use a weak reference to the original MediaPlayer object so that the native
     * code is safe from the object disappearing from underneath it.  (This is
     * the cookie passed to native_setup().)
     */
    private static void postEventFromNative(Object mediaplayer_ref,
    				int what, int arg1, int arg2, Object obj)
    {
        //得到弱引用对象
        final MediaPlayer mp = (MediaPlayer)((WeakReference)mediaplayer_ref).get();
        if (mp == null) {
            return;
        }

        switch (what) {
        case MEDIA_INFO:
            if (arg1 == MEDIA_INFO_STARTED_AS_NEXT) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        // this acquires the wakelock if needed, and sets the client side state
                        mp.start();
                    }
                }).start();
                Thread.yield();
            }
            break;

        case MEDIA_DRM_INFO:
            // We need to derive mDrmInfo before prepare() returns so processing it here
            // before the notification is sent to EventHandler below. EventHandler runs in the
            // notification looper so its handleMessage might process the event after prepare()
            // has returned.
            Log.v(TAG, "postEventFromNative MEDIA_DRM_INFO");
            if (obj instanceof Parcel) {
                Parcel parcel = (Parcel)obj;
                DrmInfo drmInfo = new DrmInfo(parcel);
                synchronized (mp.mDrmLock) {
                    mp.mDrmInfo = drmInfo;
                }
            } else {
                Log.w(TAG, "MEDIA_DRM_INFO msg.obj of unexpected type " + obj);
            }
            break;

        case MEDIA_PREPARED:
            // By this time, we've learned about DrmInfo's presence or absence. This is meant
            // mainly for prepareAsync() use case. For prepare(), this still can run to a race
            // condition b/c MediaPlayerNative releases the prepare() lock before calling notify
            // so we also set mDrmInfoResolved in prepare().
            synchronized (mp.mDrmLock) {
                mp.mDrmInfoResolved = true;
            }
            break;

        }
		//Handler不为空，发送一条message
        if (mp.mEventHandler != null) {
            Message m = mp.mEventHandler.obtainMessage(what, arg1, arg2, obj);
            mp.mEventHandler.sendMessage(m);
        }
    }
```

JAVA层MediaPlayer的构造函数中，**native_setup**方法在android_media_MediaPlayer.cpp中对应实现：

```cpp
static void
android_media_MediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)
{
    ALOGV("native_setup");
    sp<MediaPlayer> mp = new MediaPlayer();
    if (mp == NULL) {
        jniThrowException(env, "java/lang/RuntimeException", "Out of memory");
        return;
    }

    // create new listener and give it to MediaPlayer
    //给MediaPlayer创建一个listener以便Java层设置的
    //setPrepareListener、setOnCompleteListener能产生回调
    sp<JNIMediaPlayerListener> listener = new JNIMediaPlayerListener(env, thiz, weak_this);
    mp->setListener(listener);

    // Stow our new C++ MediaPlayer in an opaque field in the Java object.
    //对于Java层来说，C++中的MediaPlayer是不透明的，无需关心实现逻辑，各司其职即可
    setMediaPlayer(env, thiz, mp);
}
```

-----------------------------------------------------------------------------------------------------------------------------------------------------------

#### 2.2.3 setDataSource过程