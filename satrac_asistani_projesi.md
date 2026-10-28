# Satranç Asistanı / Ekran Overlay Projesi Dosyaları

Bu dosya, herhangi bir harici oyuna doğrudan kodsal bağ kurmadan, tamamen bağımsız (ekranı dışarıdan okuyarak ve overlay çizerek) çalışan bir Android uygulamasının ana kaynak kodlarını ve mimari yapısını içerir.

---

## 1. AndroidManifest.xml
Gerekli ekran izleme, overlay çizme izinleri ve servis tanımları:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.chesshelper">

    <!-- Diğer uygulamaların üzerinde yeşil kutu çizmek için overlay izni -->
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
    <!-- Arka planda kesintisiz çalışabilmek için ön plan servis izni -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <!-- Ekran görüntüsünü anlık yakalamak için gerekli servis tipi izni -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="Satranç Asistanı"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar">
        
        <activity android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Arka plan servisimizi ekran yansıtma tipiyle kaydediyoruz -->
        <service
            android:name=".ChessHelperService"
            android:enabled="true"
            android:exported="false"
            android:foregroundServiceType="mediaProjection" />
    </application>

</manifest>
```

---

## 2. MainActivity.kt
Uygulama açıldığında kullanıcıdan ekran kayıt ve overlay izinlerini talep eden katman:

```kotlin
package com.example.chesshelper

import android.app.Activity
import android.content.Context
import android.content.Intent
import android.media.projection.MediaProjectionManager
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.provider.Settings
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    private val REQUEST_OVERLAY_PERMISSION = 101
    private val REQUEST_MEDIA_PROJECTION = 102
    private lateinit var mediaProjectionManager: MediaProjectionManager

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        mediaProjectionManager = getSystemService(Context.MEDIA_PROJECTION_SERVICE) as MediaProjectionManager

        checkPermissionsAndStart()
    }

    private fun checkPermissionsAndStart() {
        if (!Settings.canDrawOverlays(this)) {
            val intent = Intent(
                Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                Uri.parse("package:$packageName")
            )
            startActivityForResult(intent, REQUEST_OVERLAY_PERMISSION)
        } else {
            startScreenCaptureRequest()
        }
    }

    private fun startScreenCaptureRequest() {
        val captureIntent = mediaProjectionManager.createScreenCaptureIntent()
        startActivityForResult(captureIntent, REQUEST_MEDIA_PROJECTION)
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode: Int, resultCode: Int, data)

        if (requestCode == REQUEST_OVERLAY_PERMISSION) {
            if (Settings.canDrawOverlays(this)) {
                startScreenCaptureRequest()
            } else {
                Toast.makeText(this, "Overlay izni verilmedi, uygulama çalışamaz.", Toast.LENGTH_SHORT).show()
                finish()
            }
        }

        if (requestCode == REQUEST_MEDIA_PROJECTION) {
            if (resultCode == Activity.RESULT_OK && data != null) {
                val serviceIntent = Intent(this, ChessHelperService::class.java).apply {
                    putExtra("RESULT_CODE", resultCode)
                    putExtra("DATA_INTENT", data)
                }
                
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    startForegroundService(serviceIntent)
                } else {
                    startService(serviceIntent)
                }
                finish()
            } else {
                Toast.makeText(this, "Ekran yakalama izni reddedildi.", Toast.LENGTH_SHORT).show()
                finish()
            }
        }
    }
}
```

---

## 3. ChessHelperService.kt
Görüntü piksellerini izleyen ve WindowManager ile bağımsız yeşil kutu (Overlay) oluşturan ana motor:

```kotlin
package com.example.chesshelper

import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import android.content.Intent
import android.graphics.Color
import android.graphics.PixelFormat
import android.hardware.display.DisplayManager
import android.hardware.display.VirtualDisplay
import android.media.ImageReader
import android.media.projection.MediaProjection
import android.media.projection.MediaProjectionManager
import android.os.Build
import android.os.IBinder
import android.view.Gravity
import android.view.View
import android.view.WindowManager
import androidx.core.app.NotificationCompat

class ChessHelperService : android.app.Service() {

    private val NOTIFICATION_ID = 888
    private val CHANNEL_ID = "ChessHelperChannel"

    private var windowManager: WindowManager? = null
    private var mediaProjection: MediaProjection? = null
    private var virtualDisplay: VirtualDisplay? = null
    private var imageReader: ImageReader? = null
    
    private var greenBoxOverlay: View? = null
    private var overlayParams: WindowManager.LayoutParams? = null

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onCreate() {
        super.onCreate()
        windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        createNotificationChannel()
        
        val notification: Notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Satranç Asistanı Aktif")
            .setContentText("Ekranınız analiz ediliyor ve hamle yönlendirmesi hazır.")
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .build()
        
        startForeground(NOTIFICATION_ID, notification)
        setupGreenBoxOverlay()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val resultCode = intent?.getIntExtra("RESULT_CODE", android.app.Activity.RESULT_CANCELED) ?: android.app.Activity.RESULT_CANCELED
        val dataIntent = intent?.getParcelableExtra<Intent>("DATA_INTENT")

        if (resultCode == android.app.Activity.RESULT_OK && dataIntent != null) {
            val mpManager = getSystemService(Context.MEDIA_PROJECTION_SERVICE) as MediaProjectionManager
            mediaProjection = mpManager.getMediaProjection(resultCode, dataIntent)
            startScreenCapture()
        }

        return START_NOT_STICKY
    }

    private fun setupGreenBoxOverlay() {
        greenBoxOverlay = View(this).apply {
            setBackgroundColor(Color.argb(0x55, 0x00, 0xFF, 0x00)) 
        }

        overlayParams = WindowManager.LayoutParams(
            150, 
            150, 
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE,
            PixelFormat.TRANSLUCENT
        ).apply {
            gravity = Gravity.TOP or Gravity.START
            x = 0
            y = 0
        }

        windowManager?.addView(greenBoxOverlay, overlayParams)
    }

    private fun startScreenCapture() {
        val metrics = resources.displayMetrics
        val screenWidth = metrics.widthPixels
        val screenHeight = metrics.heightPixels
        val screenDensity = metrics.densityDpi

        imageReader = ImageReader.newInstance(screenWidth, screenHeight, PixelFormat.RGBA_8888, 2)
        
        virtualDisplay = mediaProjection?.createVirtualDisplay(
            "ChessCapture",
            screenWidth, screenHeight, screenDensity,
            DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
            imageReader?.surface, null, null
        )

        imageReader?.setOnImageAvailableListener({ reader ->
            val image = reader.acquireLatestImage()
            if (image != null) {
                // Görüntü işleme ve satranç motoru (Stockfish vb.) bağlantısı buraya eklenmelidir.
                image.close()
            }
        }, null)
    }

    private fun updateBoxPosition(targetX: Int, targetY: Int) {
        overlayParams?.let { params ->
            params.x = targetX
            params.y = targetY
            windowManager?.updateViewLayout(greenBoxOverlay, params)
        }
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val serviceChannel = NotificationChannel(
                CHANNEL_ID,
                "Satranç Asistanı Servis Kanalı",
                NotificationManager.IMPORTANCE_DEFAULT
            )
            val manager = getSystemService(NotificationManager::class.java)
            manager?.createNotificationChannel(serviceChannel)
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        if (greenBoxOverlay != null && windowManager != null) {
            windowManager?.removeView(greenBoxOverlay)
        }
        virtualDisplay?.release()
        mediaProjection?.stop()
    }
}
```

---

## 4. Gönderdiğiniz Görseldeki Pozisyonun FEN Yapısı (Başlangıç Durumu)

Görselinizdeki satranç tahtası başlangıç durumunda olup siyah bakış açısına göre terstir. Satranç motorları ile bağımsız çalışırken kullanabileceğiniz standart FEN kodu:

```text
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR b KQkq - 0 1
```