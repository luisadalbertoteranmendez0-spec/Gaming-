# IA Gaming: archivos para GitHub

En tu repositorio toca **Add file → Create new file**. En el nombre escribe la ruta exacta (con las barras `/`, GitHub crea las carpetas solo), pega el contenido y toca **Commit changes**. Repite con los 7 archivos.


## Archivo 1: `settings.gradle`

```groovy
pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement { repositories { google(); mavenCentral() } }
rootProject.name = "IAGaming"
include ':app'
```


## Archivo 2: `build.gradle`

```groovy
plugins {
    id 'com.android.application' version '8.5.2' apply false
    id 'org.jetbrains.kotlin.android' version '1.9.24' apply false
}
```


## Archivo 3: `app/build.gradle`

```groovy
plugins { id 'com.android.application'; id 'org.jetbrains.kotlin.android' }
android {
    namespace 'com.iagaming'
    compileSdk 34
    defaultConfig { applicationId 'com.iagaming'; minSdk 26; targetSdk 34; versionCode 1; versionName '1.0' }
    compileOptions { sourceCompatibility JavaVersion.VERSION_17; targetCompatibility JavaVersion.VERSION_17 }
    kotlinOptions { jvmTarget = '17' }
}
```


## Archivo 4: `app/src/main/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE"/>
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
    <uses-permission android:name="android.permission.QUERY_ALL_PACKAGES"/>
    <application android:label="IA Gaming" android:theme="@android:style/Theme.Material.NoActionBar">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <service android:name=".OverlayService" android:exported="false" android:foregroundServiceType="specialUse">
            <property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE" android:value="overlay"/>
        </service>
    </application>
</manifest>
```


## Archivo 5: `app/src/main/java/com/iagaming/MainActivity.kt`

```kotlin
package com.iagaming

import android.Manifest
import android.app.Activity
import android.content.Context
import android.content.Intent
import android.graphics.Color
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.provider.Settings
import android.widget.*

object Games {
    fun load(c: Context): MutableList<Pair<String, String>> =
        c.getSharedPreferences("iag", 0).getString("g", "")!!.lines()
            .filter { it.contains("|") }
            .map { it.substringBefore("|") to it.substringAfter("|") }.toMutableList()

    fun save(c: Context, l: List<Pair<String, String>>) {
        c.getSharedPreferences("iag", 0).edit()
            .putString("g", l.joinToString("\n") { "${it.first}|${it.second}" }).apply()
    }
}

class MainActivity : Activity() {
    private lateinit var list: LinearLayout
    private fun tv(t: String, s: Float = 16f) = TextView(this).apply { text = t; textSize = s; setTextColor(Color.parseColor("#E9ECF8")) }
    private fun btn(t: String, f: () -> Unit) = Button(this).apply { text = t; setOnClickListener { f() } }
    private fun input(h: String) = EditText(this).apply { hint = h; setHintTextColor(Color.GRAY); setTextColor(Color.WHITE); isSingleLine = true }

    override fun onCreate(b: Bundle?) {
        super.onCreate(b)
        val col = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; setPadding(48, 96, 48, 48) }
        val root = ScrollView(this).apply { setBackgroundColor(Color.parseColor("#101322")); addView(col) }
        col.addView(tv("IA Gaming", 28f))
        col.addView(tv("Activa las pestañas flotantes y abre tu juego: el botón IA se queda encima de cualquier app."))
        col.addView(btn("Activar pestañas flotantes") { start() })
        col.addView(btn("Detener") { stopService(Intent(this, OverlayService::class.java)) })
        col.addView(tv("Mis juegos", 20f))
        list = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL }
        col.addView(list)
        val n = input("Nombre del juego")
        val p = input("Paquete (ej. com.dts.freefireth)")
        col.addView(n); col.addView(p)
        col.addView(btn("Guardar juego") {
            val name = n.text.toString().trim(); val pkg = p.text.toString().trim()
            if (name.isNotEmpty() && pkg.isNotEmpty()) {
                val l = Games.load(this); l.add(name to pkg); Games.save(this, l)
                n.text.clear(); p.text.clear(); render()
            }
        })
        col.addView(btn("Usar Free Fire") { n.setText("Free Fire"); p.setText("com.dts.freefireth") })
        setContentView(root); render()
    }

    private fun render() {
        list.removeAllViews()
        val l = Games.load(this)
        l.forEachIndexed { i, g -> list.addView(btn("Quitar ${g.first}") { l.removeAt(i); Games.save(this, l); render() }) }
    }

    private fun start() {
        if (!Settings.canDrawOverlays(this)) {
            Toast.makeText(this, "Permite 'Mostrar sobre otras apps' y vuelve a tocar Activar", Toast.LENGTH_LONG).show()
            startActivity(Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, Uri.parse("package:$packageName")))
            return
        }
        if (Build.VERSION.SDK_INT >= 33) requestPermissions(arrayOf(Manifest.permission.POST_NOTIFICATIONS), 1)
        startForegroundService(Intent(this, OverlayService::class.java))
    }
}
```


## Archivo 6: `app/src/main/java/com/iagaming/OverlayService.kt`

```kotlin
package com.iagaming

import android.app.*
import android.content.Intent
import android.content.pm.ServiceInfo
import android.graphics.Color
import android.graphics.PixelFormat
import android.graphics.drawable.GradientDrawable
import android.os.*
import android.provider.Settings
import android.view.*
import android.widget.*
import java.text.SimpleDateFormat
import java.util.*
import kotlin.math.abs

class OverlayService : Service() {
    private lateinit var wm: WindowManager
    private var handle: View? = null
    private var panel: View? = null
    private var info: TextView? = null
    private var tipI = 0
    private val h = Handler(Looper.getMainLooper())
    private val tips = listOf(
        "Juega con el celular fresco: si se calienta, quítale la funda.",
        "Calienta 2 minutos antes de una partida seria.",
        "Usa audífonos: escuchar pasos te da ventaja.",
        "Descansa 5 minutos cada hora para mantener buenos reflejos.",
        "Cierra apps en segundo plano para liberar RAM."
    )
    private val ticker = object : Runnable {
        override fun run() {
            val bm = getSystemService(BATTERY_SERVICE) as BatteryManager
            val bat = bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
            info?.text = SimpleDateFormat("HH:mm:ss", Locale.getDefault()).format(Date()) + "   Batería $bat%"
            h.postDelayed(this, 1000)
        }
    }

    override fun onBind(i: Intent?): IBinder? = null

    override fun onStartCommand(i: Intent?, f: Int, id: Int): Int {
        val nm = getSystemService(NotificationManager::class.java)
        nm.createNotificationChannel(NotificationChannel("iag", "IA Gaming", NotificationManager.IMPORTANCE_LOW))
        val n = Notification.Builder(this, "iag").setContentTitle("IA Gaming activo")
            .setSmallIcon(android.R.drawable.ic_media_play).build()
        if (Build.VERSION.SDK_INT >= 34) startForeground(1, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE)
        else startForeground(1, n)
        if (handle == null) { wm = getSystemService(WINDOW_SERVICE) as WindowManager; makeHandle() }
        return START_STICKY
    }

    private fun dp(v: Int) = (v * resources.displayMetrics.density).toInt()
    private fun lp(x: Int, y: Int, w: Int, hh: Int) =
        WindowManager.LayoutParams(w, hh, WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE, PixelFormat.TRANSLUCENT
        ).apply { gravity = Gravity.TOP or Gravity.START; this.x = x; this.y = y }

    private fun tv(t: String, s: Float = 15f) = TextView(this).apply { text = t; textSize = s; setTextColor(Color.parseColor("#E9ECF8")); setPadding(0, dp(6), 0, dp(6)) }
    private fun btn(t: String, f: () -> Unit) = Button(this).apply { text = t; isAllCaps = false; setOnClickListener { f() } }
    private fun title(t: String) = tv(t, 13f).apply { setTextColor(Color.parseColor("#5CE1E6")); setPadding(0, dp(14), 0, dp(2)) }

    // Pestaña pegada al borde izquierdo: se arrastra en vertical y al tocarla abre el panel.
    private fun makeHandle() {
        val v = View(this).apply {
            background = GradientDrawable().apply { cornerRadii = floatArrayOf(0f, 0f, 24f, 24f, 24f, 24f, 0f, 0f); setColor(Color.parseColor("#FF5A36")) }
        }
        val p = lp(0, dp(260), dp(14), dp(90))
        wm.addView(v, p); handle = v
        var dy = 0f; var sy = 0; var moved = false
        v.setOnTouchListener { _, e ->
            when (e.action) {
                MotionEvent.ACTION_DOWN -> { dy = e.rawY; sy = p.y; moved = false }
                MotionEvent.ACTION_MOVE -> {
                    if (abs(e.rawY - dy) > 10) moved = true
                    if (moved) { p.y = sy + (e.rawY - dy).toInt(); wm.updateViewLayout(v, p) }
                }
                MotionEvent.ACTION_UP -> if (!moved) togglePanel()
            }
            true
        }
    }

    private fun tile(t: String, action: String) = btn(t) {
        try { startActivity(Intent(action).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)) }
        catch (e: Exception) { Toast.makeText(this, "Tu celular no tiene esa opción", Toast.LENGTH_SHORT).show() }
    }

    private fun togglePanel() {
        if (panel != null) { h.removeCallbacks(ticker); wm.removeView(panel); panel = null; info = null; return }
        val col = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; setPadding(dp(14), dp(40), dp(14), dp(24)) }
        val scroll = ScrollView(this).apply {
            addView(col)
            background = GradientDrawable().apply { cornerRadii = floatArrayOf(0f, 0f, 40f, 40f, 40f, 40f, 0f, 0f); setColor(Color.parseColor("#F2101322")); setStroke(dp(1), Color.parseColor("#2B3260")) }
        }
        val top = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL; gravity = Gravity.CENTER_VERTICAL }
        top.addView(tv("IA Gaming", 20f).apply { layoutParams = LinearLayout.LayoutParams(0, -2, 1f) })
        top.addView(btn("✕") { togglePanel() })
        col.addView(top)
        info = tv("").also { col.addView(it) }

        col.addView(title("Atajos"))
        val row1 = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL }
        val row2 = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL }
        fun half(b: Button) = b.apply { layoutParams = LinearLayout.LayoutParams(0, -2, 1f) }
        row1.addView(half(tile("No molestar", Settings.ACTION_ZEN_MODE_PRIORITY_SETTINGS)))
        row1.addView(half(tile("Brillo", Settings.ACTION_DISPLAY_SETTINGS)))
        row2.addView(half(tile("Ahorro", Settings.ACTION_BATTERY_SAVER_SETTINGS)))
        row2.addView(half(tile("Sonido", Settings.ACTION_SOUND_SETTINGS)))
        col.addView(row1); col.addView(row2)

        col.addView(title("Mis juegos"))
        val gs = Games.load(this)
        if (gs.isEmpty()) col.addView(tv("Añade juegos desde la app."))
        gs.forEach { g -> col.addView(btn("Jugar ${g.first}") { launch(g.second) }) }

        col.addView(title("Coach"))
        val tip = tv(tips[tipI % tips.size]).also { col.addView(it) }
        col.addView(btn("Otro consejo") { tipI++; tip.text = tips[tipI % tips.size] })

        wm.addView(scroll, lp(0, 0, dp(290), WindowManager.LayoutParams.MATCH_PARENT)); panel = scroll
        ticker.run()
    }

    private fun launch(pkg: String) {
        val i = packageManager.getLaunchIntentForPackage(pkg)
        if (i == null) Toast.makeText(this, "No encuentro ese juego instalado", Toast.LENGTH_SHORT).show()
        else startActivity(i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK))
    }

    override fun onDestroy() {
        h.removeCallbacks(ticker)
        panel?.let { wm.removeView(it) }; handle?.let { wm.removeView(it) }
        super.onDestroy()
    }
}
```


## Archivo 7: `.github/workflows/build.yml`

```yaml
name: Build APK
on: [workflow_dispatch, push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: android-actions/setup-android@v3
      - uses: gradle/actions/setup-gradle@v4
        with: { gradle-version: 8.7 }
      - run: gradle assembleDebug
      - uses: actions/upload-artifact@v4
        with: { name: IAGaming-APK, path: app/build/outputs/apk/debug/app-debug.apk }
```
