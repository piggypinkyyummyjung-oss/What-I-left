name: Build APK (single file)
on: [push, workflow_dispatch]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.9"
      - name: Write source files
        run: |
          mkdir -p proj && cd proj
          mkdir -p app/src/main/java/com/example/jars app/src/main/res/layout app/src/main/res/drawable app/src/main/res/xml app/src/main/assets
          cat > settings.gradle.kts <<'JARS_EOF_7Q'
          pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories { google(); mavenCentral() }
          }
          rootProject.name = "Jars"
          include(":app")
          JARS_EOF_7Q
          cat > build.gradle.kts <<'JARS_EOF_7Q'
          plugins {
              id("com.android.application") version "8.5.2" apply false
              id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          JARS_EOF_7Q
          cat > gradle.properties <<'JARS_EOF_7Q'
          org.gradle.jvmargs=-Xmx2g
          JARS_EOF_7Q
          cat > app/build.gradle.kts <<'JARS_EOF_7Q'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.example.jars"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.example.jars"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          JARS_EOF_7Q
          cat > app/src/main/AndroidManifest.xml <<'JARS_EOF_7Q'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.INTERNET" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission android:name="android.permission.USE_BIOMETRIC" />
              <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
              <application
                  android:label="กระปุก"
                  android:icon="@android:drawable/sym_def_app_icon"
                  android:allowBackup="true">
                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:windowSoftInputMode="adjustResize">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
                  <activity
                      android:name=".QuickActivity"
                      android:exported="false"
                      android:excludeFromRecents="true"
                      android:taskAffinity=""
                      android:theme="@android:style/Theme.DeviceDefault.Dialog" />
                  <receiver android:name=".JarWidget" android:exported="false">
                      <intent-filter>
                          <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
                      </intent-filter>
                      <meta-data android:name="android.appwidget.provider" android:resource="@xml/widget_info" />
                  </receiver>
                  <receiver android:name=".ReminderReceiver" android:exported="false" />
                  <receiver android:name=".BootReceiver" android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.BOOT_COMPLETED" />
                      </intent-filter>
                  </receiver>
              </application>
          </manifest>
          JARS_EOF_7Q
          cat > app/src/main/java/com/example/jars/MainActivity.kt <<'JARS_EOF_7Q'
          package com.example.jars

          import android.Manifest
          import android.app.Activity
          import android.content.Context
          import android.content.Intent
          import android.net.Uri
          import android.os.Build
          import android.os.Bundle
          import android.webkit.JavascriptInterface
          import android.webkit.ValueCallback
          import android.webkit.WebChromeClient
          import android.webkit.WebView
          import org.json.JSONObject

          class MainActivity : Activity() {
              private lateinit var web: WebView
              private var chooser: ValueCallback<Array<Uri>>? = null
              private var pendingText = ""

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  web = WebView(this)
                  web.settings.javaScriptEnabled = true
                  web.settings.domStorageEnabled = true
                  web.webChromeClient = object : WebChromeClient() {
                      override fun onShowFileChooser(
                          view: WebView?,
                          callback: ValueCallback<Array<Uri>>?,
                          params: FileChooserParams?
                      ): Boolean {
                          chooser?.onReceiveValue(null)
                          chooser = callback
                          val pdf = params?.acceptTypes?.any { it.contains("pdf") } == true
                          val i = Intent(Intent.ACTION_GET_CONTENT)
                          i.addCategory(Intent.CATEGORY_OPENABLE)
                          i.type = if (pdf) "application/pdf" else "*/*"
                          return try {
                              startActivityForResult(Intent.createChooser(i, "เลือกไฟล์"), 1)
                              true
                          } catch (e: Exception) {
                              chooser = null
                              false
                          }
                      }
                  }
                  web.addJavascriptInterface(Bridge(this, web), "Android")
                  web.loadUrl("file:///android_asset/index.html")
                  setContentView(web)
                  if (Build.VERSION.SDK_INT >= 33) {
                      requestPermissions(arrayOf(Manifest.permission.POST_NOTIFICATIONS), 2)
                  }
                  Reminder.schedule(this)
              }

              fun startSave(name: String, mime: String, text: String) {
                  pendingText = text
                  val i = Intent(Intent.ACTION_CREATE_DOCUMENT)
                  i.addCategory(Intent.CATEGORY_OPENABLE)
                  i.type = mime
                  i.putExtra(Intent.EXTRA_TITLE, name)
                  try {
                      startActivityForResult(i, 3)
                  } catch (e: Exception) {
                      web.evaluateJavascript("toast('บันทึกไฟล์ไม่ได้')", null)
                  }
              }

              @Deprecated("Deprecated in Java")
              override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
                  super.onActivityResult(requestCode, resultCode, data)
                  val uri = data?.data
                  if (requestCode == 1) {
                      chooser?.onReceiveValue(if (resultCode == RESULT_OK && uri != null) arrayOf(uri) else null)
                      chooser = null
                  } else if (requestCode == 3 && resultCode == RESULT_OK && uri != null) {
                      val ok = try {
                          contentResolver.openOutputStream(uri)?.use { it.write(pendingText.toByteArray(Charsets.UTF_8)) }
                          true
                      } catch (e: Exception) {
                          false
                      }
                      web.evaluateJavascript(if (ok) "toast('บันทึกไฟล์แล้ว')" else "toast('บันทึกไฟล์ไม่สำเร็จ')", null)
                      pendingText = ""
                  }
              }
          }

          class Bridge(private val c: Context, private val web: WebView) {
              @JavascriptInterface
              fun load(): String = c.getSharedPreferences("jars", 0).getString("data", "") ?: ""

              @JavascriptInterface
              fun save(json: String) {
                  c.getSharedPreferences("jars", 0).edit().putString("data", json).apply()
                  JarWidget.refresh(c)
              }

              @JavascriptInterface
              fun price(sym: String) {
                  Thread {
                      val p = try { Prices.thb(sym) } catch (e: Exception) { null }
                      web.post {
                          val q = JSONObject.quote(sym)
                          if (p != null) web.evaluateJavascript("window.__price($q,$p)", null)
                          else web.evaluateJavascript("window.__perr($q)", null)
                      }
                  }.start()
              }

              @JavascriptInterface
              fun saveFile(name: String, mime: String, text: String) {
                  val a = c as MainActivity
                  a.runOnUiThread { a.startSave(name, mime, text) }
              }

              @JavascriptInterface
              fun setAlerts(json: String) {
                  c.getSharedPreferences("jars", 0).edit().putString("alerts", json).apply()
              }

              @JavascriptInterface
              fun pushNote(title: String, msg: String) {
                  Reminder.show(c, 200, title, msg)
              }

              @JavascriptInterface
              fun setQuick(json: String) {
                  c.getSharedPreferences("jars", 0).edit().putString("quick", json).apply()
              }

              @JavascriptInterface
              fun bio() {
                  if (Build.VERSION.SDK_INT < 28) {
                      web.post { web.evaluateJavascript("window.__bio&&window.__bio(false)", null) }
                      return
                  }
                  val a = c as Activity
                  a.runOnUiThread { Biometric.ask(a, web) }
              }
          }
          JARS_EOF_7Q
          cat > app/src/main/java/com/example/jars/JarWidget.kt <<'JARS_EOF_7Q'
          package com.example.jars

          import android.app.PendingIntent
          import android.appwidget.AppWidgetManager
          import android.appwidget.AppWidgetProvider
          import android.content.ComponentName
          import android.content.Context
          import android.content.Intent
          import android.widget.RemoteViews
          import org.json.JSONObject
          import java.text.NumberFormat
          import java.time.LocalDate
          import java.time.temporal.ChronoUnit

          class JarWidget : AppWidgetProvider() {
              override fun onUpdate(ctx: Context, m: AppWidgetManager, ids: IntArray) {
                  for (id in ids) m.updateAppWidget(id, build(ctx))
              }

              companion object {
                  fun refresh(ctx: Context) {
                      val m = AppWidgetManager.getInstance(ctx)
                      val ids = m.getAppWidgetIds(ComponentName(ctx, JarWidget::class.java))
                      for (id in ids) m.updateAppWidget(id, build(ctx))
                  }

                  private fun build(ctx: Context): RemoteViews {
                      val v = RemoteViews(ctx.packageName, R.layout.widget)
                      val nf = NumberFormat.getIntegerInstance()
                      val sb = StringBuilder()
                      try {
                          val raw = ctx.getSharedPreferences("jars", 0).getString("data", "") ?: ""
                          val jars = JSONObject(raw).getJSONArray("jars")
                          val today = LocalDate.now()
                          val ts = today.toString()
                          for (i in 0 until minOf(jars.length(), 5)) {
                              val j = jars.getJSONObject(i)
                              val type = j.optString("type", "d")
                              val tx = j.getJSONArray("tx")
                              var bal = 0.0
                              var net = 0.0
                              var units = 0.0
                              for (k in 0 until tx.length()) {
                                  val t = tx.getJSONObject(k)
                                  val a = t.getDouble("a")
                                  bal += a
                                  units += t.optDouble("u", 0.0)
                                  val nt = t.optString("n", "")
                                  val sys = t.optBoolean("g", false) || nt == "แบ่งเงินเดือน" || nt == "ส่วนที่เหลือจากเงินเดือน" ||
                                      nt == "ปรับส่วนที่เหลือ" || nt.startsWith("โอนไป")
                                  if (t.getString("d") == ts && !sys) net += a
                              }
                              val price = j.optDouble("price", 0.0)
                              if (type == "k" && price > 0 && units > 0) bal = units * price
                              if (sb.isNotEmpty()) sb.append("\n")
                              sb.append(j.getString("name")).append("  ").append(nf.format(Math.round(bal))).append(" ฿")
                              if (type == "d") {
                                  var end = LocalDate.parse(j.optString("end", ts))
                                  while (end.isBefore(today)) end = end.plusMonths(1)
                                  val days = maxOf(1L, ChronoUnit.DAYS.between(today, end) + 1)
                                  val left = (bal - net) / days + net
                                  sb.append("  ·  วันนี้เหลือ ").append(nf.format(Math.round(left))).append(" · วันละ ").append(nf.format(Math.round(bal / days)))
                              }
                          }
                      } catch (e: Exception) {
                      }
                      v.setTextViewText(R.id.body, if (sb.isEmpty()) "ยังไม่มีกระปุก แตะเพื่อเปิดแอป" else sb.toString())
                      val pi = PendingIntent.getActivity(ctx, 0, Intent(ctx, MainActivity::class.java), PendingIntent.FLAG_IMMUTABLE)
                      v.setOnClickPendingIntent(R.id.root, pi)
                      val qi = Intent(ctx, QuickActivity::class.java)
                      qi.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                      v.setOnClickPendingIntent(R.id.quick, PendingIntent.getActivity(ctx, 2, qi, PendingIntent.FLAG_IMMUTABLE))
                      return v
                  }
              }
          }
          JARS_EOF_7Q
          cat > app/src/main/java/com/example/jars/QuickActivity.kt <<'JARS_EOF_7Q'
          package com.example.jars

          import android.app.Activity
          import android.os.Bundle
          import android.text.InputType
          import android.widget.ArrayAdapter
          import android.widget.Button
          import android.widget.EditText
          import android.widget.LinearLayout
          import android.widget.Spinner
          import android.widget.TextView
          import android.widget.Toast
          import org.json.JSONArray
          import org.json.JSONObject
          import java.text.NumberFormat
          import java.time.LocalDate

          class QuickActivity : Activity() {
              private lateinit var spin: Spinner
              private lateinit var amt: EditText
              private lateinit var total: TextView

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  title = "บันทึกรายการ"
                  val pad = (16 * resources.displayMetrics.density).toInt()
                  val root = LinearLayout(this)
                  root.orientation = LinearLayout.VERTICAL
                  root.setPadding(pad, pad, pad, pad)

                  spin = Spinner(this)
                  spin.adapter = ArrayAdapter(this, android.R.layout.simple_spinner_dropdown_item, categories())
                  root.addView(spin)

                  amt = EditText(this)
                  amt.hint = "จำนวนเงิน (บาท)"
                  amt.inputType = InputType.TYPE_CLASS_NUMBER or InputType.TYPE_NUMBER_FLAG_DECIMAL
                  root.addView(amt)

                  val row = LinearLayout(this)
                  row.orientation = LinearLayout.HORIZONTAL
                  val plus = Button(this)
                  plus.text = "＋ เติมเงิน"
                  plus.setOnClickListener { save(1) }
                  val minus = Button(this)
                  minus.text = "－ หักเงิน"
                  minus.setOnClickListener { save(-1) }
                  row.addView(plus, LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1f))
                  row.addView(minus, LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1f))
                  root.addView(row)

                  total = TextView(this)
                  total.textSize = 18f
                  root.addView(total)
                  refreshTotal()
                  setContentView(root)
              }

              private fun quick(): JSONObject {
                  val raw = getSharedPreferences("jars", 0).getString("quick", "") ?: ""
                  return try { JSONObject(raw) } catch (e: Exception) { JSONObject() }
              }

              private fun categories(): List<String> {
                  val out = ArrayList<String>()
                  out.add("ไม่ระบุหมวด")
                  val a = quick().optJSONArray("cats")
                  if (a != null && a.length() > 0) {
                      for (i in 0 until a.length()) out.add(a.optString(i))
                  } else {
                      out.add("ค่าอาหาร")
                      out.add("ค่าเดินทาง")
                      out.add("ค่าช้อปปิ้ง")
                  }
                  return out
              }

              private fun refreshTotal() {
                  val s = quick().optDouble("spent", 0.0)
                  total.text = "ใช้รวมทั้งหมด (รอบนี้): " + NumberFormat.getIntegerInstance().format(Math.round(s)) + " ฿"
              }

              private fun msg(t: String) {
                  Toast.makeText(this, t, Toast.LENGTH_SHORT).show()
              }

              private fun save(sign: Int) {
                  val t = amt.text.toString().trim().replace(",", "")
                  if (t.isEmpty()) { msg("กรุณากรอกข้อมูลให้ครบถ้วนก่อน"); return }
                  val v = t.toDoubleOrNull()
                  if (v == null || v.isNaN() || v.isInfinite()) { msg("โปรดระบุเป็นตัวเลขเท่านั้น"); return }
                  if (v <= 0) { msg("จำนวนเงินต้องมากกว่า 0"); return }
                  val sp = getSharedPreferences("jars", 0)
                  val raw = sp.getString("data", "") ?: ""
                  try {
                      val o = if (raw.isEmpty()) JSONObject() else JSONObject(raw)
                      val jars = o.optJSONArray("jars")
                      var jar: JSONObject? = null
                      if (jars != null) {
                          for (i in 0 until jars.length()) {
                              val j = jars.getJSONObject(i)
                              if (j.optString("type") == "d") { jar = j; break }
                          }
                      }
                      val jj = jar
                      if (jj == null) { msg("ยังไม่มีกระปุกรายรับ-รายจ่าย กรุณาเปิดแอปก่อน"); return }
                      var tx = jj.optJSONArray("tx")
                      if (tx == null) { tx = JSONArray(); jj.put("tx", tx) }
                      val e = JSONObject()
                      e.put("id", System.currentTimeMillis() + Math.random())
                      e.put("a", sign * v)
                      e.put("d", LocalDate.now().toString())
                      if (sign < 0 && spin.selectedItemPosition > 0) e.put("c", spin.selectedItem.toString())
                      tx.put(e)
                      sp.edit().putString("data", o.toString()).apply()
                      if (sign < 0) {
                          val q = quick()
                          q.put("spent", q.optDouble("spent", 0.0) + v)
                          sp.edit().putString("quick", q.toString()).apply()
                      }
                      JarWidget.refresh(this)
                      amt.setText("")
                      refreshTotal()
                      msg("บันทึกแล้ว")
                  } catch (e: Exception) {
                      msg("บันทึกไม่สำเร็จ กรุณาเปิดแอปแล้วลองใหม่")
                  }
              }
          }
          JARS_EOF_7Q
          cat > app/src/main/java/com/example/jars/Prices.kt <<'JARS_EOF_7Q'
          package com.example.jars

          import org.json.JSONObject
          import java.net.HttpURLConnection
          import java.net.URL
          import java.net.URLEncoder

          object Prices {
              private fun get(url: String): String {
                  val con = URL(url).openConnection() as HttpURLConnection
                  con.setRequestProperty("User-Agent", "Mozilla/5.0")
                  con.connectTimeout = 10000
                  con.readTimeout = 10000
                  return con.inputStream.bufferedReader().use { it.readText() }
              }

              private fun quote(sym: String): Pair<Double, String> {
                  val url = "https://query1.finance.yahoo.com/v8/finance/chart/" +
                      URLEncoder.encode(sym, "UTF-8") + "?interval=1d&range=1d"
                  val meta = JSONObject(get(url)).getJSONObject("chart")
                      .getJSONArray("result").getJSONObject(0).getJSONObject("meta")
                  return Pair(meta.getDouble("regularMarketPrice"), meta.optString("currency", "THB"))
              }

              /** ราคาเป็นบาท: ถ้าหุ้นเป็นสกุลอื่นจะคูณอัตราแลกเปลี่ยนให้ */
              fun thb(sym: String): Double {
                  val (p, cur) = quote(sym)
                  if (cur == "THB") return p
                  return p * quote(cur + "THB=X").first
              }
          }
          JARS_EOF_7Q
          cat > app/src/main/java/com/example/jars/Reminder.kt <<'JARS_EOF_7Q'
          package com.example.jars

          import android.app.AlarmManager
          import android.app.Notification
          import android.app.NotificationChannel
          import android.app.NotificationManager
          import android.app.PendingIntent
          import android.content.BroadcastReceiver
          import android.content.Context
          import android.content.Intent
          import org.json.JSONArray
          import java.time.LocalDate
          import java.util.Calendar

          object Reminder {
              private const val CH = "jars"

              fun show(ctx: Context, id: Int, title: String, msg: String) {
                  val nm = ctx.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
                  nm.createNotificationChannel(NotificationChannel(CH, "การแจ้งเตือนกระปุก", NotificationManager.IMPORTANCE_DEFAULT))
                  val pi = PendingIntent.getActivity(ctx, 0, Intent(ctx, MainActivity::class.java), PendingIntent.FLAG_IMMUTABLE)
                  val n = Notification.Builder(ctx, CH)
                      .setSmallIcon(android.R.drawable.ic_dialog_info)
                      .setContentTitle(title)
                      .setContentText(msg)
                      .setContentIntent(pi)
                      .setAutoCancel(true)
                      .build()
                  try {
                      nm.notify(id, n)
                  } catch (e: SecurityException) {
                  }
              }

              fun schedule(ctx: Context) {
                  val am = ctx.getSystemService(Context.ALARM_SERVICE) as AlarmManager
                  val pi = PendingIntent.getBroadcast(
                      ctx, 1, Intent(ctx, ReminderReceiver::class.java),
                      PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
                  )
                  val cal = Calendar.getInstance()
                  cal.set(Calendar.HOUR_OF_DAY, 9)
                  cal.set(Calendar.MINUTE, 0)
                  cal.set(Calendar.SECOND, 0)
                  if (cal.timeInMillis < System.currentTimeMillis()) cal.add(Calendar.DAY_OF_YEAR, 1)
                  am.setInexactRepeating(AlarmManager.RTC_WAKEUP, cal.timeInMillis, AlarmManager.INTERVAL_DAY, pi)
              }
          }

          class ReminderReceiver : BroadcastReceiver() {
              override fun onReceive(ctx: Context, intent: Intent?) {
                  try {
                      val raw = ctx.getSharedPreferences("jars", 0).getString("alerts", "") ?: ""
                      if (raw.isEmpty()) return
                      val arr = JSONArray(raw)
                      val today = LocalDate.now().toString()
                      var n = 0
                      for (i in 0 until arr.length()) {
                          val o = arr.getJSONObject(i)
                          if (o.optString("d") == today) {
                              Reminder.show(ctx, 100 + n, o.optString("t"), o.optString("m"))
                              n++
                          }
                      }
                  } catch (e: Exception) {
                  }
              }
          }

          class BootReceiver : BroadcastReceiver() {
              override fun onReceive(ctx: Context, intent: Intent?) {
                  Reminder.schedule(ctx)
              }
          }
          JARS_EOF_7Q
          cat > app/src/main/java/com/example/jars/Biometric.kt <<'JARS_EOF_7Q'
          package com.example.jars

          import android.app.Activity
          import android.content.DialogInterface
          import android.hardware.biometrics.BiometricPrompt
          import android.os.CancellationSignal
          import android.webkit.WebView

          object Biometric {
              @android.annotation.TargetApi(28)
              fun ask(a: Activity, web: WebView) {
                  val cb = object : BiometricPrompt.AuthenticationCallback() {
                      override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult?) {
                          web.evaluateJavascript("window.__bio&&window.__bio(true)", null)
                      }

                      override fun onAuthenticationError(errorCode: Int, errString: CharSequence?) {
                          web.evaluateJavascript("window.__bio&&window.__bio(false)", null)
                      }
                  }
                  val prompt = BiometricPrompt.Builder(a)
                      .setTitle("ปลดล็อกแอป")
                      .setNegativeButton("ใช้ PIN", a.mainExecutor, DialogInterface.OnClickListener { _, _ -> })
                      .build()
                  prompt.authenticate(CancellationSignal(), a.mainExecutor, cb)
              }
          }
          JARS_EOF_7Q
          cat > app/src/main/res/layout/widget.xml <<'JARS_EOF_7Q'
          <?xml version="1.0" encoding="utf-8"?>
          <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
              android:id="@+id/root"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              android:orientation="vertical"
              android:padding="12dp"
              android:background="@drawable/bg">
              <LinearLayout
                  android:layout_width="match_parent"
                  android:layout_height="wrap_content"
                  android:orientation="horizontal"
                  android:gravity="center_vertical">
                  <TextView
                      android:layout_width="0dp"
                      android:layout_height="wrap_content"
                      android:layout_weight="1"
                      android:text="กระปุกของฉัน"
                      android:textColor="#5E7378"
                      android:textSize="12sp" />
                  <TextView
                      android:id="@+id/quick"
                      android:layout_width="wrap_content"
                      android:layout_height="wrap_content"
                      android:paddingLeft="12dp"
                      android:paddingRight="12dp"
                      android:paddingTop="4dp"
                      android:paddingBottom="4dp"
                      android:text="＋ บันทึก"
                      android:textColor="#FFFFFF"
                      android:textSize="13sp"
                      android:textStyle="bold"
                      android:background="@drawable/pill" />
              </LinearLayout>
              <TextView
                  android:id="@+id/body"
                  android:layout_width="match_parent"
                  android:layout_height="wrap_content"
                  android:textColor="#14262B"
                  android:textSize="15sp"
                  android:textStyle="bold" />
          </LinearLayout>
          JARS_EOF_7Q
          cat > app/src/main/res/drawable/bg.xml <<'JARS_EOF_7Q'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
              <solid android:color="#EDF2F3" />
              <corners android:radius="18dp" />
          </shape>
          JARS_EOF_7Q
          cat > app/src/main/res/drawable/pill.xml <<'JARS_EOF_7Q'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
              <solid android:color="#F0578A" />
              <corners android:radius="14dp" />
          </shape>
          JARS_EOF_7Q
          cat > app/src/main/res/xml/widget_info.xml <<'JARS_EOF_7Q'
          <?xml version="1.0" encoding="utf-8"?>
          <appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
              android:minWidth="250dp"
              android:minHeight="110dp"
              android:updatePeriodMillis="1800000"
              android:initialLayout="@layout/widget"
              android:resizeMode="horizontal|vertical"
              android:widgetCategory="home_screen" />
          JARS_EOF_7Q
          cat > app/src/main/assets/index.html <<'JARS_EOF_7Q'
          <!DOCTYPE html>
          <html lang="th">
          <head>
          <meta charset="utf-8">
          <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
          <title>กระปุกรายวัน</title>
          <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Noto+Sans+Thai:wght@400;600;800&family=Mali:wght@500;700&display=swap">
          <style>
          :root{--bg:#FFF3F7;--card:#FFFFFF;--ink:#4B3A4A;--mute:#9A8496;--line:#F4DCE6;--ok:#22A67F;--bad:#EC5479;--pv:#7F6BE8;--acc:#FF7FAF;
          box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px)}
          @media (prefers-color-scheme:dark){:root:not([data-theme="light"]){--bg:#1D1520;--card:#2A1F2E;--ink:#FBEFF5;--mute:#BCA6B7;--line:#43324A;--ok:#5FD9B3;--bad:#FF8AA6;--pv:#B0A2FF}}
          :root[data-theme="dark"]{--bg:#1D1520;--card:#2A1F2E;--ink:#FBEFF5;--mute:#BCA6B7;--line:#43324A;--ok:#5FD9B3;--bad:#FF8AA6;--pv:#B0A2FF}
          *{box-sizing:border-box}
          html{scroll-padding-top:env(safe-area-inset-top,0px)}
          body{margin:0;background:var(--bg);background-image:radial-gradient(circle at 8% 0%,rgba(255,150,195,.28),transparent 38%),radial-gradient(circle at 100% 18%,rgba(150,130,245,.2),transparent 34%);background-repeat:no-repeat;color:var(--ink);font-family:"Noto Sans Thai",system-ui,sans-serif;line-height:1.6}
          main{max-width:480px;margin:0 auto;padding:18px 16px 40px}
          h1{font:700 26px "Mali","Noto Sans Thai",system-ui,sans-serif;margin:6px 0 14px}
          .t-d{--acc:#38C79C}.t-p,.t-f{--acc:#8B78EE}.t-o{--acc:#7ACB5B}.t-s{--acc:#FF9168}.t-k{--acc:#4FAEF2}.t-c{--acc:#FF6FA5}.t-i{--acc:#EFB02E}
          .card{background:var(--card);border:1.5px solid var(--line);border-radius:26px;padding:18px;margin-bottom:14px;box-shadow:0 8px 22px rgba(255,127,175,.12)}
          @supports (background:color-mix(in srgb,red,blue)){
          .card{background:linear-gradient(160deg,color-mix(in srgb,var(--acc) 16%,var(--card)),var(--card) 62%);border-color:color-mix(in srgb,var(--acc) 30%,var(--line));box-shadow:0 8px 22px color-mix(in srgb,var(--acc) 16%,transparent)}
          .tag{background:color-mix(in srgb,var(--acc) 22%,var(--card));border-color:transparent}
          .b.m{background:color-mix(in srgb,var(--bad) 13%,var(--card))}
          .b.p{background:color-mix(in srgb,var(--ok) 13%,var(--card))}
          .b.v{background:color-mix(in srgb,var(--pv) 13%,var(--card))}
          .tabs button.on{background:color-mix(in srgb,var(--pv) 14%,var(--card))}
          }
          .top{display:flex;justify-content:space-between;align-items:center;gap:8px}
          .name{font:700 18px "Mali","Noto Sans Thai",system-ui,sans-serif}
          .tag{font-size:12px;color:var(--ink);border:1px solid var(--line);border-radius:999px;padding:1px 10px;white-space:nowrap}
          .big{font:700 36px/1.2 "Mali","Noto Sans Thai",system-ui,sans-serif;margin:4px 0 6px}
          .neg{color:var(--bad)}.pos{color:var(--ok)}
          .sub{display:flex;justify-content:space-between;gap:8px;color:var(--mute);font-size:14px}
          .sub b{color:var(--ink);font-weight:600;text-align:right}
          .bar{height:12px;border-radius:999px;background:var(--line);overflow:hidden;margin:10px 0}
          .bar i{display:block;height:100%;border-radius:999px;background:var(--acc)}
          .q{display:grid;grid-template-columns:1fr auto auto;gap:8px;margin-top:12px}
          .q.one{grid-template-columns:1fr auto}
          input,select{width:100%;padding:11px 14px;border-radius:16px;border:1.5px solid var(--line);background:var(--card);color:var(--ink);font:400 16px inherit;font-family:inherit}
          input:focus,select:focus,button:focus-visible,summary:focus-visible{outline:2px solid var(--pv);outline-offset:2px}
          .b{border:1.5px solid;border-radius:999px;padding:0 16px;font:700 16px inherit;font-family:inherit;cursor:pointer;background:transparent;min-height:44px;transition:transform .1s}
          .b:active{transform:scale(.95)}
          .b.m{color:var(--bad);border-color:var(--bad)}
          .b.p{color:var(--ok);border-color:var(--ok)}
          .b.v{color:var(--pv);border-color:var(--pv)}
          details{margin-top:12px;font-size:14px;color:var(--mute)}
          summary{cursor:pointer;padding:4px 0}
          ul{list-style:none;padding:0;margin:4px 0}
          li{display:flex;gap:8px;align-items:center;padding:6px 0;border-bottom:1.5px dashed var(--line);color:var(--ink)}
          li .n{flex:1;overflow-wrap:anywhere}
          .lnk,li button{border:0;background:transparent;color:var(--mute);cursor:pointer;font:inherit;font-family:inherit;padding:2px 8px}
          .set{display:grid;gap:4px;margin-top:8px}
          .set label{font-size:13px}
          .tabs{display:grid;grid-template-columns:repeat(2,1fr);gap:8px}
          .tabs button{padding:10px 6px;border-radius:18px;border:1.5px solid var(--line);background:var(--card);color:var(--mute);font:600 15px inherit;font-family:inherit;cursor:pointer}
          .tabs button.on{border-color:var(--pv);color:var(--pv)}
          .save{padding:13px;border:0;border-radius:999px;background:linear-gradient(135deg,#FF8FB8,#9B8CF5);color:#fff;font:700 17px inherit;font-family:inherit;cursor:pointer;box-shadow:0 6px 16px rgba(155,140,245,.35)}
          .save:active{transform:scale(.97)}
          .grid{display:grid;gap:10px}
          .row2{display:grid;grid-template-columns:1fr auto;gap:8px;align-items:center}
          .chk{display:flex;gap:8px;align-items:center;font-size:14px;color:var(--mute)}
          .chk input{width:auto}
          .empty{color:var(--mute);margin:8px 0 12px;text-align:center;font-size:16px}
          .chip{display:flex;justify-content:space-between;align-items:center;gap:8px;border:1.5px solid var(--line);border-radius:20px;padding:8px 14px;margin:8px 0;font-weight:600}
          .chip b{font:700 26px "Mali","Noto Sans Thai",system-ui,sans-serif}
          @supports (background:color-mix(in srgb,red,blue)){.chip{background:color-mix(in srgb,var(--acc) 18%,var(--card));border-color:transparent}}
          .imp textarea{width:100%;min-height:90px;border-radius:14px;border:1.5px solid var(--line);background:var(--card);color:var(--ink);padding:10px;font:inherit;margin:6px 0}
          .stats{display:grid;grid-template-columns:1fr 1fr;gap:8px}
          .st{border:1.5px solid var(--line);border-radius:18px;padding:10px 12px}
          .st small{display:block;color:var(--mute);font-size:13px}
          .st b{font:700 20px "Mali","Noto Sans Thai",system-ui,sans-serif;display:block}
          .st i{font-style:normal;color:var(--mute);font-size:12px}
          .mt{display:grid;grid-template-columns:.9fr 1fr 1fr 1fr;gap:6px 8px;font-size:13px;margin:4px 0 8px;text-align:right}
          .mt b{font-size:12px;color:var(--mute);font-weight:600}
          .mt span:nth-child(4n+1),.mt b:first-child{text-align:left}
          .card.compact{padding:6px 16px}
          .cbtn{width:100%;display:flex;align-items:center;gap:10px;background:transparent;border:0;color:var(--ink);font:inherit;cursor:pointer;padding:6px 0;text-align:left;min-height:46px}
          .cbtn .cn{flex:1;min-width:0;overflow-wrap:anywhere;font:700 17px "Mali","Noto Sans Thai",system-ui,sans-serif}
          .cbtn .cv{font-weight:700}
          .cbtn .cc{color:var(--mute);font-size:22px}
          .t-g{--acc:#FFB347}
          .icons{display:grid;grid-template-columns:repeat(auto-fill,minmax(44px,1fr));gap:6px}
          .icons button{font-size:22px;border:1.5px solid var(--line);background:var(--card);border-radius:14px;min-height:44px;cursor:pointer}
          .icons button.on{border-color:var(--pv)}
          #addc>summary{font:700 17px "Mali","Noto Sans Thai",system-ui,sans-serif;color:var(--ink);cursor:pointer;padding:6px 0}
          .err{border-color:var(--bad)!important}
          #toast{position:fixed;left:12px;right:12px;top:calc(env(safe-area-inset-top,0px) + 10px);z-index:50;background:var(--bad);color:#fff;border-radius:16px;padding:12px 14px;font-weight:600;text-align:center;box-shadow:0 8px 22px rgba(0,0,0,.25);display:none;max-width:456px;margin:0 auto}
          #alerts{border:1.5px solid var(--bad);border-radius:18px;padding:8px 14px;margin-bottom:12px;background:var(--card)}
          .alert{padding:3px 0;font-weight:600}
          #cfg textarea{width:100%;min-height:80px;border-radius:14px;border:1.5px solid var(--line);background:var(--card);color:var(--ink);padding:10px;font:inherit}
          #cfg>details>summary{font:700 17px "Mali","Noto Sans Thai",system-ui,sans-serif;cursor:pointer;padding:4px 0}
          #undo{position:fixed;left:12px;right:12px;bottom:calc(env(safe-area-inset-bottom,0px) + 12px);z-index:40;max-width:456px;margin:0 auto;background:var(--ink);color:var(--bg);border-radius:16px;padding:10px 14px;display:none;justify-content:space-between;align-items:center;font-weight:600}
          #undo button{border:0;background:transparent;color:var(--bg);font:inherit;text-decoration:underline;cursor:pointer;min-height:36px}
          #lock{position:fixed;inset:0;z-index:100;background:var(--bg);display:flex;flex-direction:column;align-items:center;justify-content:center;gap:14px;padding:20px}
          #lock .lt{font:700 20px "Mali","Noto Sans Thai",system-ui,sans-serif}
          #lock .ld{font-size:26px;letter-spacing:6px}
          #lock .lm{color:var(--bad);min-height:24px;font-weight:600}
          #lock .lp{display:grid;grid-template-columns:repeat(3,76px);gap:12px}
          #lock .lp button{height:64px;border-radius:50%;border:1.5px solid var(--line);background:var(--card);color:var(--ink);font:600 24px inherit;font-family:inherit;cursor:pointer}
          .imp{border:1.5px dashed var(--pv);border-radius:18px;padding:12px;margin-top:10px;display:grid;gap:8px}
          </style>
          </head>
          <body>
          <main>
            <h1>กระปุกของฉัน 🐷</h1>
            <div id="alerts" style="display:none"></div>
            <div id="jtop"></div>
            <section class="card" id="sal"></section>
            <div id="jars"></div>
            <button class="b v" id="demob" type="button" hidden style="width:100%;margin-bottom:12px">✨ ลองดูข้อมูลตัวอย่าง</button>
            <details class="card" id="addc">
              <summary>➕ เพิ่มกระปุกใหม่</summary>
              <div class="grid" style="margin-top:12px">
                <input id="nn" placeholder="ชื่อกระปุก">
                <input id="na" type="text" inputmode="decimal" placeholder="เงินตั้งต้น (ถ้ามี)">
                <label style="font-size:13px;color:var(--mute)">เลือกไอคอน</label>
                <div id="nic"></div>
                <button class="save" id="addb" type="button">สร้างกระปุก</button>
              </div>
            </details>
            <section class="card" id="sum"></section>
            <section class="card" id="cfg"></section>
          </main>
          <script>
          const KEY="jars-v2",EM={d:"🐷",f:"🏦",k:"📈",c:"🧸",o:"🍀",i:"🛡️",g:"💰"},TY={d:"รายรับ-รายจ่าย",f:"PVD + สหกรณ์",k:"DCA หุ้น",c:"กองทุนลูก",o:"เงินเหลือ",i:"ประกัน",g:"กระปุกทั่วไป"};
          let S={salary:0,auto:false,last:"",jars:[]},NT="d",OPEN=0,LC="";
          let LOADERR=false,LASTSAVED="";
          try{const r=window.Android?Android.load():localStorage.getItem(KEY);if(r){LASTSAVED=r;try{S=Object.assign(S,JSON.parse(r))}catch(e){LOADERR=true;try{localStorage.setItem(KEY+"-bak",r)}catch(_){}}}}catch(e){LOADERR=true}
          (function(){
            const ps=S.jars.filter(j=>j.type==="p"),ss=S.jars.filter(j=>j.type==="s");
            if(!ps.length&&!ss.length)return;
            const base=ps[0]||ss[0],f={id:base.id,type:"f",name:"PVD + สหกรณ์",emp:0,er:0,prate:0,mon:0,srate:0,yrs:10,tx:[]};
            for(const j of ps){f.emp=j.emp||0;f.er=j.er||0;f.prate=j.rate||0;f.yrs=j.yrs||f.yrs;j.tx.forEach(t=>f.tx.push(Object.assign({},t,{k:"p"})))}
            for(const j of ss){f.mon=j.mon||0;f.srate=j.rate||0;j.tx.forEach(t=>f.tx.push(Object.assign({},t,{k:"s"})))}
            const at=S.jars.indexOf(base);
            S.jars=S.jars.filter(j=>j.type!=="p"&&j.type!=="s");S.jars.splice(Math.min(at,S.jars.length),0,f);
          })();
          const $=id=>document.getElementById(id);
          let _st=null;
          const flush=()=>{if(_st){clearTimeout(_st);_st=null}try{const j=JSON.stringify(S);LASTSAVED=j;if(window.Android)Android.save(j);else localStorage.setItem(KEY,j)}catch(e){toast(e&&e.name==="QuotaExceededError"?"พื้นที่จัดเก็บเต็ม กรุณาลบรายการเก่าบางส่วน":"บันทึกข้อมูลไม่สำเร็จ ลองใหม่อีกครั้ง")}};
          const store=()=>{if(_st)clearTimeout(_st);_st=setTimeout(flush,300)};
          addEventListener("pagehide",flush);document.addEventListener("visibilitychange",()=>{if(document.hidden)flush()});
          const pad=n=>String(n).padStart(2,"0");
          const ymd=d=>d.getFullYear()+"-"+pad(d.getMonth()+1)+"-"+pad(d.getDate());
          const fmt=n=>{const r=Math.round(n);return Number.isFinite(r)?(r===0?0:r).toLocaleString("th-TH"):"0"};
          const td=()=>ymd(new Date());
          const eom=()=>cend();
          const dd=(a,b)=>Math.round((new Date(b)-new Date(a))/864e5);
          const short=d=>new Date(d).toLocaleDateString("th-TH",{day:"numeric",month:"short"});
          const el=(t,c,x)=>{const e=document.createElement(t);if(c)e.className=c;if(x!=null)e.textContent=x;return e};
          const btn=(c,x,f)=>{const b=el("button","b "+c,x);b.type="button";b.onclick=f;return b};
          let V=0;
          const AG=new WeakMap(),CM=new WeakMap(),HN={};
          function agg(j){const c=AG.get(j.tx);if(c&&c.n===j.tx.length)return c;let sum=0,u=0,cost=0,p=0,s=0;for(const t of j.tx){sum+=t.a;u+=t.u||0;if(!t.adj)cost+=t.a;if(t.k==="p")p+=t.a;else if(t.k==="s")s+=t.a}const r={n:j.tx.length,sum,u,cost,p,s};AG.set(j.tx,r);return r}
          function cm(j){const ym=curC(),c=CM.get(j.tx);if(c&&c.n===j.tx.length&&c.m===ym+pd())return c.l;const l=j.tx.filter(t=>cyc(t.d)===ym);CM.set(j.tx,{n:j.tx.length,m:ym+pd(),l});return l}
          const units=j=>agg(j).u;
          const bal=j=>(j.type==="k"&&j.price>0&&agg(j).u>0)?agg(j).u*j.price:agg(j).sum;
          const px=n=>Number.isFinite(+n)?Number(n).toLocaleString("th-TH",{maximumFractionDigits:2}):"0";
          const hasA=()=>!!(window.Android&&Android.price);
          const refresh=()=>{if(!hasA())return;[...new Set(S.jars.filter(j=>j.type==="k"&&j.sym&&!j.mp).map(j=>j.sym))].forEach(x=>Android.price(x))};
          window.__price=(x,p)=>{S.jars.forEach(j=>{if(j.type==="k"&&j.sym===x&&!j.mp){j.price=p;j.pt=Date.now();j.perr=false}});store();render()};
          window.__perr=x=>{S.jars.forEach(j=>{if(j.type==="k"&&j.sym===x)j.perr=true});render()};
          const add=(j,a,n,adj,c,k)=>{V++;const t={id:Date.now()+Math.random(),a,d:td()};if(n)t.n=n;if(adj)t.adj=1;if(c)t.c=c;if(k)t.k=k;if(!isSys(t))UNDO={k:"add",j,t,msg:"บันทึกรายการแล้ว",fresh:1};return j.tx.push(t)};
          const bk=(j,k)=>agg(j)[k];
          const ml=m=>new Date(m+"-01").toLocaleDateString("th-TH",{month:"long",year:"numeric"});
          const done=()=>{hideToast();safe(autoFund);V++;store();render();if(UNDO&&UNDO.fresh){UNDO.fresh=0;showUndo()}};
          const sweep=(j,ym)=>{
            const b=bal(j);if(!(b>0))return false;
            let o=S.jars.find(x=>x.type==="o");
            if(!o){o={id:Date.now()+1,type:"o",name:"เงินเหลือจากเดือนก่อน",tx:[]};S.jars.push(o)}
            add(j,-b,"โอนไป "+o.name);
            o.tx.push({id:Date.now()+Math.random(),a:b,d:td(),n:"จาก "+j.name,adj:false,c:"",k:"",m:ym});
            return true;
          };
          const CATS=["ค่าอาหาร","ค่าน้ำ","ค่าช้อปปิ้ง","ค่าเดินทาง","ค่ารายเดือนให้ที่บ้าน","ค่ารายเดือนทั่วไป","ค่าโทรศัพท์","อื่นๆ"];
          const MO=["ค่ารายเดือนให้ที่บ้าน","ค่ารายเดือนทั่วไป","ค่าโทรศัพท์"];
          let CBO=0,MB=0;
          const isSys=t=>t.g||t.n==="สมทบ PVD"||t.n==="ส่งหุ้นสหกรณ์"||t.n==="แบ่งเงินเดือน"||t.n==="ส่วนที่เหลือจากเงินเดือน"||t.n==="ปรับส่วนที่เหลือ"||String(t.n||"").startsWith("โอนไป");
          const autoFund=()=>{
            const t0=td();for(const j of S.jars)if(j.type==="d"&&j.end&&j.end<t0)j.end=eom();
            for(const j of S.jars)if(j.type==="d")postRec(j);
            if(!(S.salary>0))return;
            (S.mon=S.mon||{})[curC()]={inc:S.salary+(S.extra||0),ded:S.deduct||0};
            const ym=curC();
            for(const j of S.jars){
              if(j.type!=="d"||!j.rest)continue;
              const m=monthly(j);
              if(j.funded===undefined){
                const ex=cm(j).filter(t=>t.n==="แบ่งเงินเดือน").reduce((x,t)=>x+t.a,0);
                if(ex>0){j.funded=ym;j.fa=ex}
              }
              if(j.funded!==ym){
                if(j.funded&&j.sweep)sweep(j,j.funded);
                j.end=eom();
                if(m>0)add(j,m,"ส่วนที่เหลือจากเงินเดือน");
                j.funded=ym;j.fa=m;
              }else if(Math.abs(m-(j.fa||0))>0.5){
                add(j,m-(j.fa||0),"ปรับส่วนที่เหลือ");j.fa=m;
              }
            }
          };
          const monthly=j=>j.type==="f"?S.salary*((j.emp||0)+(j.er||0))/100+(j.mon||0):j.rest?Math.max(0,S.salary+(S.extra||0)-(S.deduct||0)-S.jars.filter(x=>!x.rest).reduce((s,x)=>s+empPart(x),0)):j.type==="p"?S.salary*((j.emp||0)+(j.er||0))/100:(j.type==="s"||j.type==="k"||j.type==="c"||j.type==="g")?(j.mon||0):S.salary*(j.pct||0)/100;
          const empPart=j=>j.type==="f"?S.salary*(j.emp||0)/100+(j.mon||0):j.type==="p"?S.salary*(j.emp||0)/100:monthly(j);
          const proj=(b,m,rate,yrs)=>{const r=(rate||0)/1200,n=(yrs||0)*12;return r?b*Math.pow(1+r,n)+m*((Math.pow(1+r,n)-1)/r):b+m*n};

          const NUM=/\d{1,3}(?:,\d{3})+(?:\.\d{1,2})?|\d+\.\d{1,2}/g;
          const thn=t=>String(t).replace(/[\s\u0E31\u0E34-\u0E3A\u0E47-\u0E4E]/g,"").toLowerCase();
          const K=Object.fromEntries(Object.entries({sal:["เงินเดือน","salary"],ext:["ล่วงเวลา","โอที","overtime"],ded:["ฌาปนกิจ","ภาษีเงินได้หัก","ภาษีหัก"],pvd:["กองทุนสำรอง","provident"],coop:["สหกรณ์"],sav:["ออมทรัพย์","ทุนเรือนหุ้น","เงินฝาก","saving"]}).map(([k,a])=>[k,a.map(thn)]));
          const toNums=t=>(t.match(NUM)||[]).map(x=>parseFloat(x.replace(/,/g,""))).filter(x=>x>=100);
          function parsePdf(lines){
            const v={},nl=lines.map(thn);let ded=0,df=false;
            lines.forEach((ln,i)=>{
              const n=toNums(ln),h=ks=>K[ks].some(k=>nl[i].includes(k));
              if(!n.length)return;
              if(h("sal")&&v.sal==null)v.sal=n[0];
              if(h("ext")&&v.ext==null)v.ext=n[0];
              if(h("ded")){ded+=n[n.length-1];df=true}
              if(h("pvd")&&v.emp==null)v.emp=n[n.length-1];
              if(h("coop")&&v.coop==null)v.coop=n[n.length-1];
              if(h("sav")&&v.savb==null)v.savb=Math.max(...n);
              if(n.length===5&&!/[ก-๙A-Za-z]/.test(ln)&&v.er==null){v.er=n[0];v.pvdb=n[1]+n[2]}
            });
            if(df)v.ded=ded;
            return v;
          }
          const candsOf=lines=>lines.flatMap(l=>toNums(l).map(v=>({v,t:l}))).slice(0,80);
          async function pdfLines(L,buf){
            const doc=await L.getDocument({data:buf}).promise,lines=[];
            for(let p=1;p<=doc.numPages;p++){
              const tc=await(await doc.getPage(p)).getTextContent(),rows={};
              for(const it of tc.items){if(!it.str||!it.str.trim())continue;const y=Math.round(it.transform[5]/3);(rows[y]=rows[y]||[]).push({x:it.transform[4],s:it.str})}
              Object.keys(rows).map(Number).sort((a,b)=>b-a).forEach(y=>lines.push(rows[y].sort((a,b)=>a.x-b.x).map(o=>o.s).join(" ").replace(/\s+/g," ").trim()));
            }
            return lines;
          }
          let IMP=null;
          const PJ="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/";
          function loadScript(u){return new Promise((ok,no)=>{const sc=document.createElement("script");sc.src=u;sc.onload=ok;sc.onerror=()=>no(new Error("load"));document.head.append(sc)})}
          async function pdfLib(){
            if(window.pdfjsLib)return window.pdfjsLib;
            let base=PJ;
            if(window.Android){try{await loadScript("pdf.min.js");base=""}catch(e){}}
            if(!window.pdfjsLib)await loadScript(PJ+"pdf.min.js");
            window.pdfjsLib.GlobalWorkerOptions.workerSrc=base+"pdf.worker.min.js";
            return window.pdfjsLib;
          }
          function pickPdf(j){
            const f=document.createElement("input");f.type="file";f.accept="application/pdf,.pdf";
            f.onchange=async()=>{
              const file=f.files&&f.files[0];if(!file)return;
              IMP={jid:j.id,busy:true};OPEN=j.id;render();
              try{
                const L=await pdfLib();
                const lines=await pdfLines(L,new Uint8Array(await file.arrayBuffer()));
                if(!lines.length)throw new Error("ไม่พบข้อความใน PDF (อาจเป็นรูปสแกน)");
                IMP={jid:j.id,file:file.name,v:parsePdf(lines),cands:candsOf(lines)};
              }catch(e){IMP={jid:j.id,err:e&&e.name==="PasswordException"?"PDF มีรหัสผ่าน":String(e&&e.message||e)}}
              render();
            };
            f.click();
          }
          function impPanel(j){
            const p=el("div","imp");
            if(IMP.busy){p.append(el("div",null,"กำลังอ่าน PDF…"));return p}
            const close=()=>{IMP=null;render()};
            const manual=()=>{IMP={jid:j.id,file:"กรอกจากสลิป",v:{},cands:[]};OPEN=j.id;render()};
            if(IMP.err){
              p.append(el("div","neg","อ่านไฟล์ไม่สำเร็จ: "+IMP.err),el("div","sub","ไฟล์ที่ป้องกันสิทธิ์อาจเปิดในแอปไม่ได้ ลองวางข้อความ หรือกรอกจากสลิปแทน"));
              const r=el("div","row2");r.append(btn("v","✍️ กรอกจากสลิป",manual),btn("m","ปิด",close));p.append(r);return p;
            }
            p.append(el("b",null,IMP.file==="กรอกจากสลิป"?"✍️ กรอกจากสลิป (เว้นว่างได้ถ้าไม่เปลี่ยน)":"ตรวจยอดจาก "+IMP.file+" ก่อนบันทึก"));
            const pd=el("details");pd.open=!!IMP.paste;pd.append(el("summary",null,"📋 วางข้อความที่คัดลอกจากสลิป"));
            const ta=el("textarea");ta.placeholder="เปิดสลิป เลือกข้อความทั้งหมด คัดลอก แล้ววางที่นี่";
            pd.append(ta,btn("v","อ่านข้อความ",()=>{const ls=ta.value.split(/\n/).map(x=>x.trim()).filter(Boolean);IMP.v=parsePdf(ls);IMP.cands=candsOf(ls);IMP.paste=true;render()}));
            p.append(pd);
            const F=[["sal","เงินเดือน (บาท)"],["ext","รายได้อื่นประจำ เช่น ค่าล่วงเวลา"],["ded","หักอื่นๆ ต่อเดือน เช่น ภาษี ฌาปนกิจ"],["emp","เราสมทบ PVD ต่อเดือน (บาท)"],["er","นายจ้างสมทบ PVD ต่อเดือน (บาท)"],["coop","ส่งหุ้นสหกรณ์ต่อเดือน (บาท)"],["pvdb","ยอดสะสม PVD รวม (ส่วนเรา + นายจ้าง)"],["savb","ยอดออมทรัพย์/สหกรณ์สะสม"]],inps={};
            for(const [k,l] of F){
              const row=el("div","set");row.append(el("label",null,l));
              const i=el("input");i.type="text";i.inputMode="decimal";i.inputMode="decimal";i.value=IMP.v[k]!=null?Math.round(IMP.v[k]*100)/100:"";inps[k]=i;row.append(i);
              if(IMP.cands.length){
                const sel=el("select"),o0=el("option",null,"เลือกจากตัวเลขที่พบ…");o0.value="";sel.append(o0);
                for(const c of IMP.cands){const o=el("option",null,px(c.v)+" · "+c.t.slice(0,30));o.value=c.v;sel.append(o)}
                sel.onchange=()=>{if(sel.value)i.value=sel.value};row.append(sel);
              }
              p.append(row);
            }
            const g=k=>{const x=parseNum(inps[k].value).v;return x===undefined?null:x};
            const r=el("div","row2");
            r.append(btn("p","บันทึกทั้งหมด",()=>{
              const vs=Object.values(inps),bd=vs.filter(e=>parseNum(e.value).bad);
              if(bd.length){bd.forEach(e=>e.classList.add("err"));toast(NUMMSG);return}
              if(vs.every(e=>!e.value.trim())){toast(FILLMSG);return}
              if(g("sal")!=null)S.salary=g("sal");
              if(g("ext")!=null)S.extra=g("ext");
              if(g("ded")!=null)S.deduct=g("ded");
              const base=S.salary;
              if(g("emp")!=null&&base>0)j.emp=Math.round(g("emp")/base*10000)/100;
              if(g("er")!=null&&base>0)j.er=Math.round(g("er")/base*10000)/100;
              if(g("coop")!=null)j.mon=g("coop");
              if(g("pvdb")!=null)add(j,g("pvdb")-bk(j,"p"),"ยอดจากสลิป",true,"","p");
              if(g("savb")!=null)add(j,g("savb")-bk(j,"s"),"ยอดจากสลิป",true,"","s");
              IMP=null;done()}),btn("m","ยกเลิก",close));
            p.append(r);return p;
          }
          const EXP={};
          let SM=null,SYR=null,SUO=false,SPV="m",SPM="cat",SDO=false;
          const COL=["#FF7FAF","#8B78EE","#38C79C","#4FAEF2","#FF9168","#EFB02E","#7ACB5B","#E86AC1","#8AA0B5"];
          const shiftM=(m,n)=>{const [y,mo]=m.split("-").map(Number);return ymd(new Date(y,mo-1+n,1)).slice(0,7)};
          const pc=(v,t)=>t>0?(v/t*100).toFixed(1)+"%":"–";
          let IX=null,IXV=-1,SC={},SCV=-1;
          function idx(){if(IX&&IXV===V)return IX;IX={};for(const j of S.jars)for(const t of j.tx){if(t.adj||t.o)continue;const m=cyc(t.d);(IX[m]=IX[m]||[]).push([j,t])}IXV=V;return IX}
          const stats=ym=>{if(SCV!==V){SC={};SCV=V}return SC[ym]||(SC[ym]=stats0(ym))};
          function stats0(ym){
            const mon=ym.length===4?Object.entries(S.mon||{}).filter(([k])=>k.startsWith(ym)).reduce((a,[,v])=>({inc:a.inc+(v.inc||0),ded:a.ded+(v.ded||0)}),{inc:0,ded:0}):((S.mon||{})[ym]||{}),ex={},sv={};let add2=0;
            const X=idx(),E=ym.length===4?Object.keys(X).filter(k=>k.startsWith(ym)).flatMap(k=>X[k]):(X[ym]||[]);
            for(const [j,t] of E){
              const tp=j.type;
              {
                if(tp==="d"||tp==="o"){
                  if(isSys(t)||t.m)continue;
                  if(t.a<0){const k=t.c||(tp==="o"?"จากเงินเหลือ":"ไม่ระบุหมวด");ex[k]=(ex[k]||0)-t.a}else add2+=t.a;
                }else if(tp==="i"&&t.a<0){ex["เบี้ยประกัน"]=(ex["เบี้ยประกัน"]||0)-t.a}
                else if(t.a>0&&"fkcipsg".includes(tp)){
                  let a=t.a;
                  if(tp==="f"&&t.n==="สมทบ PVD"&&(j.emp||0)+(j.er||0)>0)a=a*(j.emp||0)/((j.emp||0)+(j.er||0));
                  sv[j.name]=(sv[j.name]||0)+a;
                }
              }
            }
            if(mon.ded>0)ex["หักอื่นๆ (ภาษี ฯลฯ)"]=mon.ded;
            const sum=o=>Object.values(o).reduce((x,y)=>x+y,0);
            const inc=(mon.inc||0)+add2;
            return {inc,reg:mon.inc||0,add2,ex,sv,spend:sum(ex),save:sum(sv),left:inc-sum(ex)-sum(sv)};
          }
          function pie(items,total,center){
            const NS="http://www.w3.org/2000/svg",mk=(t,at)=>{const e=document.createElementNS(NS,t);for(const k in at)e.setAttribute(k,at[k]);return e};
            const svg=mk("svg",{viewBox:"-1.05 -1.05 2.1 2.1",width:"200",height:"200",role:"img","aria-label":"แผนภูมิวงกลม"});
            svg.style.cssText="display:block;margin:8px auto";
            let a0=-Math.PI/2;
            items.forEach((it,i)=>{
              const f=it.v/total;let e;
              if(f>=0.9999)e=mk("circle",{r:1});
              else{const a1=a0+f*2*Math.PI;e=mk("path",{d:`M ${Math.cos(a0)} ${Math.sin(a0)} A 1 1 0 ${f>.5?1:0} 1 ${Math.cos(a1)} ${Math.sin(a1)} L 0 0 Z`});a0=a1}
              e.style.fill=it.col||COL[i%COL.length];e.style.stroke="var(--card)";e.style.strokeWidth="0.025";svg.append(e);
            });
            const h=mk("circle",{r:.56});h.style.fill="var(--card)";svg.append(h);
            const t1=mk("text",{x:0,y:-.05,"text-anchor":"middle","font-size":".13"});t1.textContent=center[0];t1.style.fill="var(--mute)";
            const t2=mk("text",{x:0,y:.17,"text-anchor":"middle","font-size":".2","font-weight":"700"});t2.textContent=center[1];t2.style.fill="var(--ink)";
            svg.append(t1,t2);return svg;
          }
          function sumPanel(){
            const c=$("sum");c.innerHTML="";
            const cur=curC(),curY=cur.slice(0,4);
            c.append(el("b",null,"📊 สรุปรายรับ-รายจ่าย"));
            const bt=el("div","tabs");bt.style.margin="10px 0 4px";
            for(const [k,t] of [["m","📅 รายเดือน"],["y","🗓️ รายปี"]]){
              const b=el("button",SUO&&SPV===k?"on":"",t);b.type="button";
              b.onclick=()=>{if(SUO&&SPV===k)SUO=false;else{SUO=true;SPV=k}render()};bt.append(b);
            }
            c.append(bt);
            if(!SUO)return;
            const isY=SPV==="y",key=isY?(SYR||curY):(SM||cur),st=stats(key);
            const go=n=>()=>{if(isY)SYR=String(+key+n);else SM=shiftM(key,n);render()};
            const nav=el("div");nav.style.cssText="display:grid;grid-template-columns:auto 1fr auto;gap:8px;align-items:center;text-align:center;margin:8px 0";
            const pb=btn("v","‹",go(-1)),nb=btn("v","›",go(1));
            nb.disabled=isY?key>=curY:key>=cur;pb.setAttribute("aria-label","ก่อนหน้า");nb.setAttribute("aria-label","ถัดไป");
            nav.append(pb,el("b",null,isY?"ปี "+(+key+543):ml(key)+(pd()>1?" ("+short(cs(key))+" – "+short(cend(key))+")":"")),nb);c.append(nav);
            if(!(st.inc>0||st.spend>0||st.save>0)){c.append(el("div","empty","ยังไม่มีรายการในช่วงนี้"));return}
            const sg=el("div","stats");
            for(const [l,v,cl] of [["เงินเข้า",st.inc,"pos"],["ค่าใช้จ่าย",st.spend,"neg"],["เงินออม/ลงทุน",st.save,""],["คงเหลือ",st.left,st.left<0?"neg":"pos"]]){
              const b=el("div","st");b.append(el("small",null,l),el("b",cl,fmt(v)+" ฿"),el("i",null,l==="เงินเข้า"?"100%":pc(v,st.inc)+" ของเงินเข้า"));sg.append(b);
            }
            c.append(sg);
            const tb=el("div","tabs");tb.style.marginTop="12px";
            for(const [k,t] of [["cat","ค่าใช้จ่ายตามหมวด"],["all","ภาพรวมเข้า-ออก"]]){const b=el("button",SPM===k?"on":"",t);b.type="button";b.onclick=()=>{SPM=k;render()};tb.append(b)}
            c.append(tb);
            let items,total,center;
            if(SPM==="cat"){items=Object.entries(st.ex).filter(x=>x[1]>0).sort((a,b)=>b[1]-a[1]).map(([n,v])=>({n,v}));total=st.spend;center=["ค่าใช้จ่าย",fmt(st.spend)]}
            else{items=[{n:"ค่าใช้จ่าย",v:st.spend,col:"#FF7FAF"},{n:"เงินออม/ลงทุน",v:st.save,col:"#8B78EE"},{n:"คงเหลือ",v:Math.max(0,st.left),col:"#38C79C"}].filter(x=>x.v>0);total=items.reduce((x,y)=>x+y.v,0);center=["เงินเข้า",fmt(st.inc)]}
            if(total>0){
              c.append(pie(items,total,center));
              const lg=el("ul");
              items.forEach((it,i)=>{const li=el("li"),dot=el("span");dot.style.cssText="width:12px;height:12px;border-radius:50%;flex:none;background:"+(it.col||COL[i%COL.length]);li.append(dot,el("span","n",it.n),el("b",null,fmt(it.v)+" ฿ · "+pc(it.v,total)));lg.append(li)});
              c.append(lg);
            }else c.append(el("div","empty","ยังไม่มีค่าใช้จ่ายในช่วงนี้"));
            if(isY){
              c.append(el("div","sub","แยกรายเดือน (บาท)"));
              const mt=el("div","mt");
              for(const t of ["เดือน","เข้า","ใช้จ่าย","ออม"])mt.append(el("b",null,t));
              for(let m=1;m<=12;m++){
                const s2=stats(key+"-"+pad(m));if(!(s2.inc>0||s2.spend>0||s2.save>0))continue;
                mt.append(el("span",null,new Date(+key,m-1,1).toLocaleDateString("th-TH",{month:"short"})),el("span",null,fmt(s2.inc)),el("span","neg",fmt(s2.spend)),el("span",null,fmt(s2.save)));
              }
              c.append(mt);
            }
            const dt=el("details");dt.open=SDO;dt.addEventListener("toggle",()=>{SDO=dt.open});
            dt.append(el("summary",null,"รายละเอียดเงินเข้า-ออก (% ของเงินเข้า)"));
            const ul=el("ul");
            const head=t=>{const li=el("li");li.append(el("b",null,t));ul.append(li)};
            const row=(l,v)=>{const li=el("li");li.append(el("span","n",l),el("span",null,fmt(v)+" ฿ · "+pc(v,st.inc)));ul.append(li)};
            head("⬇️ เงินเข้า");row("รายได้ประจำ (เงินเดือน + อื่นๆ)",st.reg);if(st.add2>0)row("เติม/รับเพิ่ม",st.add2);
            head("💜 เงินออม/ลงทุน");Object.entries(st.sv).sort((a,b)=>b[1]-a[1]).forEach(([n,v])=>row(n,v));
            head("🛍️ ค่าใช้จ่าย");Object.entries(st.ex).sort((a,b)=>b[1]-a[1]).forEach(([n,v])=>row(n,v));
            dt.append(ul);c.append(dt);
          }
          const NUMMSG="โปรดระบุเป็นตัวเลขเท่านั้น",FILLMSG="กรุณากรอกข้อมูลให้ครบถ้วนก่อน";
          let _tt=null;
          function toast(m){let t=document.getElementById("toast");if(!t){t=document.createElement("div");t.id="toast";t.setAttribute("role","alert");document.body.append(t)}t.textContent=m;t.style.display="block";clearTimeout(_tt);_tt=setTimeout(()=>{t.style.display="none"},3500)}
          function hideToast(){const t=document.getElementById("toast");if(t)t.style.display="none"}
          function report(e){try{console.error(e)}catch(_){}toast("ขออภัย เกิดข้อผิดพลาดบางส่วน ข้อมูลของคุณยังปลอดภัย ลองใหม่อีกครั้ง")}
          function safe(f){try{f()}catch(e){report(e)}}
          function parseNum(raw){const t=String(raw==null?"":raw).trim();if(!t)return{empty:true};const c=t.replace(/,/g,"");if(!/^(\d+\.?\d*|\.\d+)$/.test(c))return{bad:true};const v=parseFloat(c);return Number.isFinite(v)?{v}:{bad:true}}
          function pn(e){const r=parseNum(e.value);e.classList.toggle("err",!!r.bad);e._why=r.bad?"bad":r.empty?"empty":"";if(r.bad)toast(NUMMSG);return r.v===undefined?NaN:r.v}
          function need(e){e.classList.add("err");if(e.focus)e.focus();if(e._why!=="bad")toast(e._why==="empty"?FILLMSG:"จำนวนเงินต้องมากกว่า 0")}
          addEventListener("error",e=>report(e.error||e.message));addEventListener("unhandledrejection",e=>report(e.reason));
          document.addEventListener("input",e=>{if(e.target.classList)e.target.classList.remove("err")});
          function failCard(j,e,jt,root){report(e);const c=el("section","card");c.append(el("div","neg","⚠️ กระปุก “"+(j&&j.name||"")+"” แสดงผลไม่ได้เพราะข้อมูลผิดปกติ"));const b=el("button","lnk","ลบกระปุกนี้");b.type="button";b.onclick=()=>{S.jars=S.jars.filter(x=>x!==j);done()};c.append(b);(j&&j.type==="d"?jt:root).append(c)}
          let EDIT=null,DT="",UNDO=null,RCO=0,CFO=false,IMPD=null,_ut=null,LASTAL="",PREVO=null,LOCKED=false,HIDT=0;
          /* ---- categories / widget bridge ---- */
          const cats=()=>Array.isArray(S.cats)&&S.cats.length?S.cats:CATS;
          let LASTQ="";
          function syncQuick(){
            if(!window.Android||!Android.setQuick)return;
            const dj=S.jars.find(j=>j.type==="d"),sp=dj?-cm(dj).filter(t=>t.a<0&&!isSys(t)).reduce((x,t)=>x+t.a,0):0;
            const q=JSON.stringify({cats:cats(),spent:Math.round(sp*100)/100});
            if(q!==LASTQ){LASTQ=q;try{Android.setQuick(q)}catch(e){}}
          }
          function syncNative(){
            if(!window.Android)return;
            try{const r=Android.load();if(r&&r!==LASTSAVED){S=Object.assign({salary:0,auto:false,last:"",jars:[]},JSON.parse(r));LASTSAVED=r;clean();V++;render()}}catch(e){report(e)}
          }
          function renameCat(a,b){
            S.cats=cats().map(x=>x===a?b:x);
            for(const j of S.jars){for(const t of j.tx)if(t.c===a)t.c=b;if(j.cb&&a in j.cb){j.cb[b]=j.cb[a];delete j.cb[a]}for(const r of j.rec||[])if(r.c===a)r.c=b}
            CFO=true;S.jars.forEach(touch);done();
          }
          function catPanel(g){
            const box=el("div","set");box.append(el("label",null,"หมวดรายจ่าย (ใช้ทั้งในแอปและ widget)"));
            const list=cats();
            list.forEach((n,i)=>{
              const row=el("div","row2"),inp=el("input");inp.value=n;inp.setAttribute("aria-label","ชื่อหมวด");
              inp.onchange=()=>{const v=inp.value.trim();if(!v){inp.value=n;toast(FILLMSG);return}if(v!==n&&list.includes(v)){inp.value=n;toast("มีหมวดนี้แล้ว");return}if(v!==n)renameCat(n,v)};
              const x=el("button","lnk","ลบ");x.type="button";x.onclick=()=>{S.cats=list.filter((_,k)=>k!==i);CFO=true;done()};
              row.append(inp,x);box.append(row);
            });
            const ni=el("input");ni.placeholder="เพิ่มหมวดใหม่ เช่น ค่ากาแฟ";
            box.append(ni,btn("p","เพิ่มหมวด",()=>{const v=ni.value.trim();if(!v){ni.classList.add("err");toast(FILLMSG);return}if(list.includes(v)){toast("มีหมวดนี้แล้ว");return}S.cats=[...list,v];CFO=true;done()}));
            g.append(box);
          }
          /* ---- pay cycle ---- */
          const pd=()=>{const p=Math.round(+S.payday);return p>=1&&p<=31?p:1};
          const dimM=(y,m)=>new Date(y,m,0).getDate();
          const pdt=x=>new Date(+x.slice(0,4),+x.slice(5,7)-1,+x.slice(8,10));
          function cs(l){const [y,m]=l.split("-").map(Number);return ymd(new Date(y,m-1,Math.min(pd(),dimM(y,m))))}
          function cyc(d){const y=+d.slice(0,4),m=+d.slice(5,7);return +d.slice(8,10)>=Math.min(pd(),dimM(y,m))?d.slice(0,7):shiftM(d.slice(0,7),-1)}
          const curC=()=>cyc(td());
          function cend(l){const x=pdt(cs(shiftM(l||curC(),1)));x.setDate(x.getDate()-1);return ymd(x)}
          /* ---- edit / undo ---- */
          function touch(j){AG.delete(j.tx);CM.delete(j.tx);V++}
          function showUndo(){let u=document.getElementById("undo");if(!u){u=document.createElement("div");u.id="undo";document.body.append(u)}u.innerHTML="";u.append(el("span",null,UNDO.msg));const b=el("button",null,"เลิกทำ");b.type="button";b.onclick=undo;u.append(b);u.style.display="flex";clearTimeout(_ut);_ut=setTimeout(hideUndo,6000)}
          function hideUndo(){const u=document.getElementById("undo");if(u)u.style.display="none"}
          function undo(){const u=UNDO;if(!u)return;UNDO=null;
            if(u.k==="add")u.j.tx=u.j.tx.filter(y=>y!==u.t);
            else if(u.k==="del")u.j.tx.splice(Math.min(u.i,u.j.tx.length),0,u.t);
            else if(u.k==="edit"){for(const k of ["a","d","c","n"]){if(u.was[k]===undefined)delete u.t[k];else u.t[k]=u.was[k]}}
            hideUndo();touch(u.j);done()}
          function editForm(j,t){
            const li=el("li");li.style.cssText="display:grid;grid-template-columns:1fr 1fr;gap:6px";
            const a=el("input");a.type="text";a.inputMode="decimal";a.value=Math.abs(t.a);a.setAttribute("aria-label","จำนวนเงิน");
            const d=el("input");d.type="date";d.value=t.d;d.max=td();d.setAttribute("aria-label","วันที่");
            li.append(a,d);
            let c=null;
            if(j.type==="d"&&t.a<0){c=el("select");for(const x of ["",...cats()]){const o=el("option",null,x||"ไม่ระบุหมวด");o.value=x;c.append(o)}c.value=t.c||"";li.append(c)}
            const n=el("input");n.value=t.n||"";n.placeholder="โน้ต";n.setAttribute("aria-label","โน้ต");li.append(n);
            li.append(btn("p","บันทึก",()=>{
              const v=pn(a);if(!(v>0)){need(a);return}
              if(!d.value||d.value>td()){d.classList.add("err");toast(FILLMSG);return}
              const was={a:t.a,d:t.d,c:t.c,n:t.n};
              t.a=(t.a<0?-1:1)*v;t.d=d.value;
              if(c){if(c.value)t.c=c.value;else delete t.c}
              if(n.value.trim())t.n=n.value.trim();else delete t.n;
              UNDO={k:"edit",j,t,was,msg:"แก้ไขรายการแล้ว",fresh:1};EDIT=null;touch(j);done()}),btn("m","ยกเลิก",()=>{EDIT=null;render()}));
            return li;
          }
          /* ---- recurring ---- */
          function postRec(j){const cur=curC();for(const r of j.rec||[]){if(!r.last)r.last=cur;let g=0;while(r.last<cur&&g++<24){r.last=shiftM(r.last,1);const d=cs(r.last),t={id:Date.now()+Math.random(),a:-r.a,d:d>td()?td():d,n:"ประจำ: "+r.n};if(r.c)t.c=r.c;j.tx.push(t)}}}
          /* ---- alerts ---- */
          function alertsList(){
            const L=[],t0=td();
            if(S.salary>0&&!S.auto&&S.last!==curC())L.push("💰 รอบเงินเดือนใหม่เริ่มแล้ว กด “แบ่งตอนนี้” ที่การ์ดเงินเดือน");
            for(const j of S.jars){
              if(j.type==="i"&&j.due&&j.premium>0){const n=dd(t0,j.due);if(n<=7)L.push("🛡️ "+j.name+(n<0?" เลยกำหนดจ่ายเบี้ย "+(-n)+" วัน":n===0?" ครบกำหนดจ่ายเบี้ยวันนี้":" ครบกำหนดจ่ายเบี้ยอีก "+n+" วัน"))}
              if(j.type==="d"&&j.cb)for(const k of Object.keys(j.cb)){const b=j.cb[k];if(!(b>0))continue;const sp=-cm(j).filter(t=>t.a<0&&t.c===k).reduce((x,t)=>x+t.a,0);if(sp>b)L.push("⚠️ "+k+" เกินงบ "+fmt(sp-b)+" ฿");else if(sp>=b*0.9)L.push("⚡ "+k+" ใช้ไป "+Math.round(sp/b*100)+"% ของงบ")}
            }
            return L;
          }
          function alertsPanel(){const c=$("alerts"),L=alertsList();c.innerHTML="";c.style.display=L.length?"":"none";for(const x of L)c.append(el("div","alert",x))}
          function syncAlerts(){
            if(!window.Android)return;
            const out=[],t0=td(),lim=ymd(new Date(Date.now()+90*864e5));
            const put=(d,t,m)=>{if(d>=t0&&d<=lim)out.push({d,t,m})};
            for(const j of S.jars)if(j.type==="i"&&j.due&&j.premium>0)for(const o of [7,3,1,0]){const x=pdt(j.due);x.setDate(x.getDate()-o);put(ymd(x),"🛡️ "+j.name,o?"ครบกำหนดจ่ายเบี้ยอีก "+o+" วัน":"ครบกำหนดจ่ายเบี้ยวันนี้")}
            if(S.salary>0)put(cs(shiftM(curC(),1)),"💰 เงินเดือนออกวันนี้","เปิดแอปเพื่อแบ่งเงินเข้ากระปุก");
            const j2=JSON.stringify(out);if(j2!==LASTAL){LASTAL=j2;try{if(Android.setAlerts)Android.setAlerts(j2)}catch(e){}}
            const over=new Set(alertsList().filter(x=>x.startsWith("⚠️")));
            if(PREVO)for(const x of over)if(!PREVO.has(x)){try{if(Android.pushNote)Android.pushNote("แจ้งเตือนงบ",x.slice(2).trim())}catch(e){}}
            PREVO=over;
          }
          /* ---- PIN lock ---- */
          const h53=(str,seed=0)=>{let h1=0xdeadbeef^seed,h2=0x41c6ce57^seed;for(let i=0,ch;i<str.length;i++){ch=str.charCodeAt(i);h1=Math.imul(h1^ch,2654435761);h2=Math.imul(h2^ch,1597334677)}h1=Math.imul(h1^(h1>>>16),2246822507)^Math.imul(h2^(h2>>>13),3266489909);h2=Math.imul(h2^(h2>>>16),2246822507)^Math.imul(h1^(h1>>>13),3266489909);return String(4294967296*(2097151&h2)+(h1>>>0))};
          const pinH=p=>h53("jars|"+p);
          function lockUI(){
            if(!S.pin||LOCKED)return;LOCKED=true;
            const o=document.createElement("div");o.id="lock";
            const dots=el("div","ld"),msg=el("div","lm");let v="";
            const draw=()=>{dots.textContent="●".repeat(v.length)+"○".repeat(Math.max(0,6-v.length))};draw();
            const unlock=()=>{LOCKED=false;o.remove()};
            const pad=el("div","lp");
            for(const k of ["1","2","3","4","5","6","7","8","9","⌫","0","✔"]){
              const b=el("button",null,k);b.type="button";
              b.onclick=()=>{msg.textContent="";if(k==="⌫")v=v.slice(0,-1);else if(k==="✔"){if(pinH(v)===S.pin)return unlock();msg.textContent="PIN ไม่ถูกต้อง";v=""}else if(v.length<6)v+=k;draw();if(k!=="✔"&&k!=="⌫"&&pinH(v)===S.pin)unlock()};
              pad.append(b);
            }
            o.append(el("div","lt","🔒 ใส่ PIN เพื่อเปิดแอป"),dots,msg,pad);
            if(window.Android&&Android.bio){const f=el("button","lnk","👆 ใช้ลายนิ้วมือ");f.type="button";f.onclick=()=>Android.bio();o.append(f);window.__bio=ok=>{if(ok)unlock()};setTimeout(()=>{try{Android.bio()}catch(e){}},300)}
            const fg=el("button","lnk","ลืม PIN (ล้างข้อมูลทั้งหมด)");fg.type="button";
            fg.onclick=()=>{if(fg.dataset.c){S={salary:0,auto:false,last:"",jars:[]};flush();location.reload()}else{fg.dataset.c="1";fg.textContent="แตะอีกครั้งเพื่อยืนยันล้างข้อมูล"}};
            o.append(fg);document.body.append(o);
          }
          document.addEventListener("visibilitychange",()=>{if(document.hidden)HIDT=Date.now();else{syncNative();if(S.pin&&Date.now()-HIDT>30000)lockUI()}});
          /* ---- backup / settings ---- */
          const stamp=()=>td().replace(/-/g,"");
          function csvOf(){const rows=[["วันที่","กระปุก","ประเภท","หมวด","โน้ต","จำนวนเงิน"]];for(const j of S.jars)for(const t of j.tx)rows.push([t.d,j.name,TY[j.type]||j.type,t.c||"",t.n||"",t.a]);return "\ufeff"+rows.map(r=>r.map(x=>'"'+String(x).replace(/"/g,'""')+'"').join(",")).join("\r\n")}
          function saveFile(name,mime,text){
            if(window.Android&&Android.saveFile){Android.saveFile(name,mime,text);return}
            try{const a=document.createElement("a");a.href=URL.createObjectURL(new Blob([text],{type:mime}));a.download=name;document.body.append(a);a.click();a.remove();toast("ส่งออกแล้ว ถ้าไม่เห็นไฟล์ ให้ใช้ปุ่มคัดลอกข้อความสำรอง")}catch(e){toast("ส่งออกไฟล์ไม่ได้ในหน้านี้ ใช้ปุ่มคัดลอกข้อความสำรองแทน")}
          }
          function stageImport(text){
            try{const o=JSON.parse(text),d=o&&o.data?o.data:o;if(!d||!Array.isArray(d.jars))throw 0;
              IMPD={d,nj:d.jars.length,nt:d.jars.reduce((x,j)=>x+((j&&j.tx)||[]).length,0)};CFO=true;render()}
            catch(e){toast("ไฟล์สำรองไม่ถูกต้อง โปรดเลือกไฟล์ที่ส่งออกจากแอปนี้")}
          }
          function cfgPanel(){
            const c=$("cfg");c.innerHTML="";
            const d=el("details");d.open=CFO;d.addEventListener("toggle",()=>{CFO=d.open});
            d.append(el("summary",null,"⚙️ สำรองข้อมูล ความปลอดภัย และตรวจระบบ"));
            const g=el("div","grid");g.style.marginTop="10px";
            g.append(btn("v","💾 ส่งออกไฟล์สำรอง (JSON)",()=>saveFile("jars-backup-"+stamp()+".json","application/json",JSON.stringify({v:2,data:S}))));
            g.append(btn("v","📊 ส่งออก CSV (เปิดใน Excel)",()=>saveFile("jars-"+stamp()+".csv","text/csv",csvOf())));
            g.append(btn("v","📂 นำเข้าไฟล์สำรอง",()=>{const f=document.createElement("input");f.type="file";f.accept="application/json,.json";f.onchange=()=>{const fl=f.files&&f.files[0];if(!fl)return;const r=new FileReader();r.onload=()=>stageImport(String(r.result));r.onerror=()=>toast("อ่านไฟล์ไม่สำเร็จ");r.readAsText(fl)};f.click()}));
            const ta=el("textarea");ta.placeholder="หรือวางข้อความสำรองที่นี่ แล้วกดนำเข้า";
            g.append(ta,btn("v","นำเข้าจากข้อความ",()=>stageImport(ta.value)),btn("v","📋 คัดลอกข้อความสำรอง",()=>{ta.value=JSON.stringify({v:2,data:S});ta.select();try{document.execCommand("copy");toast("คัดลอกแล้ว")}catch(e){}}));
            if(IMPD){
              const b=el("div","imp");b.append(el("div",null,"พบข้อมูล "+IMPD.nj+" กระปุก "+IMPD.nt+" รายการ ข้อมูลปัจจุบันจะถูกแทนที่ (สำรองไว้ให้ 1 ชุด)"));
              const ok=btn("p","แทนที่ข้อมูลปัจจุบัน",()=>{if(!ok.dataset.c){ok.dataset.c="1";ok.textContent="แตะอีกครั้งเพื่อยืนยัน";return}
                try{localStorage.setItem(KEY+"-bak",JSON.stringify(S))}catch(e){}
                S=Object.assign({salary:0,auto:false,last:"",jars:[],seeded:true},IMPD.d);IMPD=null;clean();done();toast("นำเข้าข้อมูลแล้ว")});
              const r=el("div","row2");r.append(ok,btn("m","ยกเลิก",()=>{IMPD=null;render()}));b.append(r);g.append(b);
            }
            catPanel(g);
            const pr=el("div","set");pr.append(el("label",null,S.pin?"เปลี่ยน PIN (ตัวเลข 4–6 หลัก)":"ตั้ง PIN ล็อกแอป (ตัวเลข 4–6 หลัก)"));
            const pi=el("input");pi.type="password";pi.inputMode="numeric";pi.maxLength=6;pi.placeholder="PIN 4–6 หลัก";pi.autocomplete="off";pr.append(pi);
            const pr2=el("div","row2");
            pr2.append(btn("p","บันทึก PIN",()=>{if(!/^\d{4,6}$/.test(pi.value)){pi.classList.add("err");toast("PIN ต้องเป็นตัวเลข 4–6 หลัก");return}S.pin=pinH(pi.value);done();toast("ตั้ง PIN แล้ว")}),S.pin?btn("m","ปิด PIN",()=>{delete S.pin;done();toast("ปิด PIN แล้ว")}):el("span"));
            pr.append(pr2);g.append(pr);
            const out=el("ul");
            g.append(btn("v","🔧 ตรวจระบบ",async()=>{
              out.innerHTML="";const row=(ok,t)=>{const li=el("li");li.append(el("span","n",(ok?"✅ ":"❌ ")+t));out.append(li)};
              try{localStorage.setItem("__t","1");row(localStorage.getItem("__t")==="1","เก็บข้อมูลในเครื่อง");localStorage.removeItem("__t")}catch(e){row(false,"เก็บข้อมูลในเครื่อง")}
              row(!!window.Android,"เชื่อมต่อแอป Android (ไฟล์ / widget / แจ้งเตือน / ลายนิ้วมือ)");
              try{await pdfLib();row(true,"ตัวอ่าน PDF")}catch(e){row(false,"ตัวอ่าน PDF")}
              if(window.Android&&Android.price){const p=await new Promise(res=>{const o1=window.__price,o2=window.__perr;let f=false;const fin=v=>{if(f)return;f=true;window.__price=o1;window.__perr=o2;res(v)};window.__price=(x,v)=>fin(v);window.__perr=()=>fin(0);Android.price("PTT.BK");setTimeout(()=>fin(-1),8000)});row(p>0,"ดึงราคาหุ้น (PTT.BK)"+(p>0?" = "+px(p):""))}
              row(true,"ข้อมูลทั้งหมด "+S.jars.length+" กระปุก "+S.jars.reduce((x,j)=>x+j.tx.length,0)+" รายการ");
            }),out);
            d.append(g);c.append(d);
          }
          const ICONS=["🐷","💰","🏦","🏠","🚗","✈️","🎓","🧸","🍼","🛒","🍜","☕","🎁","💍","🏖️","📱","💊","🐶","🐱","🎮","👗","💻","🔧","⭐","❤️","🌱"];
          const firstG=v=>{v=v.trim();if(!v)return "";return window.Intl&&Intl.Segmenter?[...new Intl.Segmenter(undefined,{granularity:"grapheme"}).segment(v)][0].segment:Array.from(v)[0]};
          function iconPick(p,cur,cb){
            const g=el("div","icons");
            for(const x of ICONS){const b=el("button",x===cur?"on":"",x);b.type="button";b.setAttribute("aria-label","ไอคอน "+x);b.onclick=()=>cb(x);g.append(b)}
            const i=el("input");i.placeholder="หรือพิมพ์อีโมจิเอง";i.style.marginTop="8px";i.onchange=()=>{const x=firstG(i.value);if(x)cb(x)};
            p.append(g,i);
          }
          function fld(p,label,o,k,kind,opts){
            p.append(el("label",null,label));
            let i;
            if(opts){i=el("select");for(const [v,t] of opts){const op=el("option",null,t);op.value=v;i.append(op)}i.value=o[k]||opts[0][0]}
            else{i=el("input");i.type=kind==="date"?"date":(kind==="t"||kind==="x")?"text":"text";if(kind==="n"){i.inputMode="decimal"}i.value=o[k]||""}
            i.onchange=()=>{if(kind==="n"){const r=parseNum(i.value);if(r.bad){i.classList.add("err");toast(NUMMSG);return}o[k]=r.empty?0:r.v}else o[k]=kind==="t"?i.value.trim().toUpperCase():kind==="x"?(i.value.trim()||o[k]):(kind==="date"||opts)?i.value:0;if(k==="sym")o.price=0;done();if(k==="sym")refresh()};
            p.append(i);
          }
          function run(){
            if(!(S.salary>0))return;
            const ym=curC();
            if(S.last&&S.last!==ym)for(const j of S.jars.slice())if(j.type==="d"&&j.sweep&&!j.rest)sweep(j,S.last);
            for(const j of S.jars){
              if(j.rest){j.end=eom();continue}
              if(j.type==="f"){const pv=S.salary*((j.emp||0)+(j.er||0))/100;if(pv>0)add(j,pv,"สมทบ PVD",false,"","p");if(j.mon>0)add(j,j.mon,"ส่งหุ้นสหกรณ์",false,"","s");continue}
              const a=monthly(j);if(a>0)add(j,a,"แบ่งเงินเดือน");
            }
            S.last=curC();
          }
          function demo(){
            S.salary=30000;S.auto=false;
            const m=(a,n)=>[{id:Math.random(),a,d:td(),n}];
            S.jars=[
              {id:1,type:"d",name:"รายรับ-รายจ่าย (ตัวอย่าง)",end:eom(),rest:true,sweep:true,cb:{"ค่ารายเดือนทั่วไป":3000,"ค่าโทรศัพท์":599,"ค่าอาหาร":6000,"ค่าเดินทาง":2500,"ค่าช้อปปิ้ง":1500},tx:[]},
              {id:2,type:"f",name:"PVD + สหกรณ์ (ตัวอย่าง)",emp:5,er:5,prate:5,mon:3000,srate:5,yrs:10,tx:[{id:Math.random(),a:120000,d:td(),n:"ยอดปัจจุบัน",k:"p"},{id:Math.random(),a:60000,d:td(),n:"ทุนเรือนหุ้น",k:"s"}]},
              {id:3,type:"i",name:"ประกัน (ตัวอย่าง)",pct:7,premium:24000,cyc:"y",due:ymd(new Date(Date.now()+60*864e5)),cover:1000000,tx:m(4000,"เริ่มต้น")},
              {id:8,type:"o",name:"เงินเหลือจากเดือนก่อน (ตัวอย่าง)",tx:[{id:Math.random(),a:850,d:td(),n:"จาก ใช้รายวัน",m:shiftM(curC(),-1)}]},
              {id:5,type:"k",name:"DCA หุ้น (ตัวอย่าง)",mon:5000,rate:7,yrs:10,sym:"PTT.BK",price:34,pt:Date.now(),tx:[{id:Math.random(),a:48000,d:td(),n:"ลงทุนสะสม",u:1500}]},
              {id:7,type:"c",name:"กองทุนลูก (ตัวอย่าง)",mon:3000,goal:500000,yrs:10,rate:3,tx:m(40000,"เริ่มต้น")}
            ];
            const od=ymd(new Date(Date.now()-70*864e5));S.jars.forEach(x=>{if(x.type!=="d"&&x.type!=="o")x.tx.forEach(t=>{t.d=od;t.o=1})});
            const r=S.jars[0];for(const [a,c] of [[120,"ค่าอาหาร"],[230,"ค่าอาหาร"],[45,"ค่าเดินทาง"],[105,"ค่าเดินทาง"],[80,"ค่าน้ำ"],[420,"ค่าช้อปปิ้ง"],[599,"ค่าโทรศัพท์"]])add(r,-a,"",false,c);run();
            done();
          }

          function salary(){
            const c=$("sal");c.innerHTML="";
            const g=el("div","grid");
            const used=S.salary>0?Math.round(S.jars.filter(j=>!j.rest).reduce((s,j)=>s+empPart(j),0)/S.salary*100):0;
            g.append(el("b",null,"💰 เงินเดือนและกฎแบ่งเงิน"));
            fld(g,"เงินเดือนต่อเดือน (บาท)",S,"salary","n");
            fld(g,"เงินเดือนออกวันที่ (1-31) เว้นว่าง = นับตามเดือนปฏิทิน",S,"payday","n");
            const xd=el("details");xd.open=!!(S.extra||S.deduct);xd.append(el("summary",null,"รายได้อื่นและรายการหักประจำ"));
            const xg=el("div","set");fld(xg,"รายได้อื่นประจำต่อเดือน (เช่น ค่าล่วงเวลา)",S,"extra","n");fld(xg,"หักอื่นๆ ต่อเดือน (ภาษี ฌาปนกิจ ฯลฯ)",S,"deduct","n");xd.append(xg);g.append(xd);
            const s=el("div","sub");s.innerHTML="<span>แบ่งเข้ากระปุกแล้ว</span><b class='"+(used>100?"neg":"")+"'>"+used+"% (เหลือ "+(100-used)+"%)</b>";
            const ch=el("label","chk");const cb=el("input");cb.type="checkbox";cb.checked=!!S.auto;cb.onchange=()=>{S.auto=cb.checked;done()};ch.append(cb,"แบ่งให้อัตโนมัติทุกต้นเดือน");
            const row=el("div","row2");row.append(ch,btn("v","แบ่งตอนนี้",()=>{run();done()}));
            g.append(s,row);
            if(S.last)g.append(el("div","sub","แบ่งล่าสุดเดือน "+S.last));
            c.append(g);
          }

          function render(){
            safe(salary);safe(sumPanel);safe(alertsPanel);safe(cfgPanel);safe(syncAlerts);safe(syncQuick);
            const root=$("jars");root.innerHTML="";const jt=$("jtop");jt.innerHTML="";$("demob").hidden=!!(S.salary>0||S.jars.some(j=>j.tx.length));
            if(!S.jars.length){
              root.append(el("p","empty","🐷 ยังไม่มีกระปุก กด ➕ เพิ่มกระปุกใหม่ด้านล่าง"));
            }
            for(const j of S.jars){try{
              const b=bal(j),c=el("section","card t-"+j.type),top=el("div","top");
              if(j.type!=="d"&&!EXP[j.id]){
                const cb=el("button","cbtn");cb.type="button";cb.setAttribute("aria-expanded","false");
                cb.append(el("span","cn",(j.ic||EM[j.type])+" "+j.name),el("span","cv"+(b<0?" neg":""),fmt(b)+" ฿"),el("span","cc","›"));
                cb.onclick=()=>{EXP[j.id]=true;render()};
                c.className="card compact t-"+j.type;c.append(cb);(j.type==="d"?jt:root).append(c);continue;
              }
              top.append(el("span","name",(j.ic||EM[j.type])+" "+j.name),(()=>{const t=el("span","tag",TY[j.type]);if(j.name===TY[j.type])t.style.display="none";return t})());
              if(j.type!=="d"){const cl=el("button","lnk","ย่อ ▲");cl.type="button";cl.setAttribute("aria-expanded","true");cl.onclick=()=>{EXP[j.id]=false;render()};top.append(cl)}
              c.append(top,el("div","big "+(b<0?"neg":""),fmt(b)+" ฿"));
              const line=(l,v)=>{const s=el("div","sub");s.append(el("span",null,l),el("b",null,v));c.append(s)};
              const det=el("details");if(OPEN===j.id)det.open=true;
              const set=el("div","set");
              const inp=el("input");inp.type="text";inp.inputMode="decimal";inp.inputMode="decimal";inp.min="0";inp.setAttribute("aria-label","จำนวนเงิน "+j.name);
              const q=el("div","q");let extra=null;
              const go=s=>()=>{const a=pn(inp);if(!(a>0)){need(inp);return}add(j,s*a);done()};

              if(j.type==="d"){
                const net=cm(j).filter(t=>t.d===td()&&!isSys(t)).reduce((s,t)=>s+t.a,0);
                const spent=-cm(j).filter(t=>t.d===td()&&t.a<0&&!isSys(t)).reduce((s,t)=>s+t.a,0);
                const dl=Math.max(1,dd(td(),j.end||eom())+1),lim=(b-net)/dl,tl=lim+net;
                if(j.rest&&!(S.salary>0))line("⚠️ ยังไม่ได้ใส่เงินเดือน","ใส่ที่การ์ดบนสุดก่อน");
                const chip=el("div","chip");chip.append(el("span",null,"วันนี้ใช้ได้อีก"),el("b",tl<0?"neg":"pos",fmt(tl)+" ฿"));c.append(chip);
                line("ใช้ได้วันละ (คงเหลือ ÷ "+dl+" วัน)",fmt(b/dl)+" ฿");
                line("วันนี้ใช้ไป "+fmt(spent)+" · เหลือ",fmt(tl)+" ฿");
                if(j.rest){const dim=dd(cs(curC()),cend())+1;line("ส่วนที่เหลือจากเงินเดือน",fmt(monthly(j))+" ฿ (เฉลี่ย "+fmt(monthly(j)/dim)+"/วัน)")}
                const bar=el("div","bar"),i=el("i");i.style.width=(lim>0?Math.max(0,Math.min(100,tl/lim*100)):0)+"%";if(tl<0)i.style.background="var(--bad)";bar.append(i);c.append(bar);
                {const cb=j.cb||{},ym2=curC();
                 for(const k of Object.keys(cb).filter(x=>cb[x]>0)){
                   const sp=-cm(j).filter(t=>t.a<0&&t.c===k).reduce((x,t)=>x+t.a,0);
                   const spT=-cm(j).filter(t=>t.a<0&&t.c===k&&t.d===td()).reduce((x,t)=>x+t.a,0);
                   const left=cb[k]-sp;
                   if(MO.includes(k))line(k,left<0?"เกินงบ "+fmt(-left)+" ฿":"เหลือ "+fmt(left)+" ฿ (งบ "+fmt(cb[k])+")");
                   else{const l2=(left+spT)/dl;line(k,left<0?"เกินงบ "+fmt(-left)+" ฿":"วันนี้เหลือ "+fmt(l2-spT)+" · วันละ "+fmt(left/dl)+" ฿")}
                 }}
                inp.placeholder="จำนวนเงิน";const cat=el("select");cat.setAttribute("aria-label","หมวดรายจ่าย");cat.style.marginTop="10px";
                for(const x of ["",...cats()]){const o=el("option",null,x||"หมวดรายจ่าย (ไม่ระบุ)");o.value=x;cat.append(o)}
                cat.value=LC;cat.onchange=()=>{LC=cat.value};
                const dti=el("input");dti.type="date";dti.max=td();dti.value=DT||td();dti.setAttribute("aria-label","วันที่ของรายการ");dti.onchange=()=>{DT=dti.value&&dti.value!==td()?dti.value:""};
                const crow=el("div");crow.style.cssText="display:grid;grid-template-columns:1fr auto;gap:8px;margin-top:10px";cat.style.marginTop="0";crow.append(cat,dti);c.append(crow);
                const goD=sg=>()=>{const a=pn(inp);if(!(a>0)){need(inp);return}if(DT>td()){toast("เลือกวันที่ในอนาคตไม่ได้");return}add(j,sg*a,"",false,sg<0?LC:"");if(DT){j.tx[j.tx.length-1].d=DT;DT=""}done()};
                q.append(inp,btn("m","− ใช้",goD(-1)),btn("p","+ เติม",goD(1)));
                fld(set,"ใช้ให้หมดภายในวันที่",j,"end","date");
                const lb=el("label","chk"),cb=el("input");cb.type="checkbox";cb.checked=!!j.rest;cb.onchange=()=>{OPEN=j.id;j.rest=cb.checked;done()};lb.append(cb,"ใช้ส่วนที่เหลือของเงินเดือนอัตโนมัติ");set.append(lb);
                if(!j.rest)fld(set,"สัดส่วนจากเงินเดือน (%)",j,"pct","n");
                const sl=el("label","chk"),sc=el("input");sc.type="checkbox";sc.checked=!!j.sweep;sc.onchange=()=>{OPEN=j.id;j.sweep=sc.checked;done()};sl.append(sc,"สิ้นเดือนโอนเงินที่เหลือไปกระปุก \"เงินเหลือ\"");set.append(sl);
                const sw=el("button","lnk","โอนเงินที่เหลือตอนนี้");sw.type="button";sw.onclick=()=>{OPEN=j.id;sweep(j,curC());done()};set.append(sw);
                j.cb=j.cb||{};
                const cd=el("details");cd.open=CBO===j.id;cd.append(el("summary",null,"งบแยกตามหมวดต่อเดือน (บาท)"));
                const cg=el("div","set");cg.addEventListener("focusin",()=>{OPEN=j.id;CBO=j.id});
                for(const k of cats())fld(cg,k,j.cb,k,"n");
                const ab=el("button","lnk","✨ แบ่งงบที่เหลือจากค่ารายเดือนให้หมวดอื่นอัตโนมัติ");ab.type="button";
                ab.onclick=()=>{OPEN=j.id;CBO=j.id;const ms=-cm(j).filter(t=>t.a<0&&!isSys(t)).reduce((x,t)=>x+t.a,0);
                  const base=j.rest?monthly(j):Math.max(0,bal(j)+ms),fixed=MO.reduce((x,k)=>x+(j.cb[k]||0),0),rest=Math.max(0,base-fixed);
                  const w={"ค่าอาหาร":45,"ค่าน้ำ":5,"ค่าช้อปปิ้ง":15,"ค่าเดินทาง":20,"อื่นๆ":15};
                  for(const k in w)j.cb[k]=Math.round(rest*w[k]/100);done()};
                cg.append(ab);cd.append(cg);set.append(cd);
                const rd=el("details");rd.open=RCO===j.id;rd.append(el("summary",null,"🔁 รายการประจำ (ตัดทุกรอบเงินเดือน)"));
                const rg=el("div","set");rg.addEventListener("focusin",()=>{OPEN=j.id;RCO=j.id});
                const rl=el("ul");
                for(const r of j.rec||[]){const li=el("li");li.append(el("span","n",r.n+(r.c?" · "+r.c:"")),el("b","neg",fmt(r.a)+" ฿"));const rx=el("button",null,"×");rx.type="button";rx.onclick=()=>{OPEN=j.id;RCO=j.id;j.rec=j.rec.filter(y=>y!==r);done()};li.append(rx);rl.append(li)}
                const rn=el("input");rn.placeholder="ชื่อรายการ เช่น ค่าโทรศัพท์";
                const ra=el("input");ra.type="text";ra.inputMode="decimal";ra.placeholder="จำนวนเงินต่อรอบ (บาท)";
                const rc=el("select");for(const x of ["",...cats()]){const o=el("option",null,x||"หมวด (ไม่ระบุ)");o.value=x;rc.append(o)}
                rg.append(rl,rn,ra,rc,btn("p","เพิ่มรายการประจำ (เริ่มรอบถัดไป)",()=>{
                  const a=pn(ra),n=rn.value.trim();
                  if(!n){rn.classList.add("err");toast(FILLMSG);return}
                  if(!(a>0)){need(ra);return}
                  (j.rec=j.rec||[]).push({id:Date.now(),n,a,c:rc.value,last:curC()});OPEN=j.id;RCO=j.id;done()}));
                rd.append(rg);set.append(rd);
              }
              if(j.type==="i"){
                const dl=j.due?dd(td(),j.due):null,sa=(j.premium||0)/(j.cyc==="y"?12:1);
                line("เบี้ย "+fmt(j.premium||0)+" ฿/"+(j.cyc==="y"?"ปี":"เดือน"),dl==null?"ยังไม่ตั้งวัน":dl<0?"เลยกำหนด "+(-dl)+" วัน":"อีก "+dl+" วัน");
                line("ควรกันไว้เดือนละ",fmt(sa)+" ฿");
                if(j.cover)line("ทุนประกัน",fmt(j.cover)+" ฿");
                inp.placeholder="จำนวนเงิน";q.append(inp,btn("m","− ใช้",go(-1)),btn("p","+ เติม",go(1)));
                c.append(q);
                const pay=btn("v","จ่ายเบี้ยงวดนี้",()=>{add(j,-(j.premium||0),"จ่ายเบี้ย");if(j.due){const d=new Date(j.due);j.cyc==="y"?d.setFullYear(d.getFullYear()+1):d.setMonth(d.getMonth()+1);j.due=ymd(d)}done()});
                pay.style.marginTop="8px";pay.style.width="100%";c.append(pay);
                fld(set,"เบี้ยต่องวด (บาท)",j,"premium","n");fld(set,"รอบจ่าย",j,"cyc","s",[["m","รายเดือน"],["y","รายปี"]]);
                fld(set,"ครบกำหนดจ่ายวันที่",j,"due","date");fld(set,"ทุนประกัน/มูลค่าเงินคืน (กรอกเอง)",j,"cover","n");
                fld(set,"สัดส่วนจากเงินเดือน (%)",j,"pct","n");
              }
              if(j.type==="k"){
                const u=units(j),cost=agg(j).cost,pl=b-cost;
                line("ลงทุนสะสม (ต้นทุน)",fmt(cost)+" ฿");
                const g=el("div","sub");g.append(el("span",null,"กำไร/ขาดทุน"),el("b",pl<0?"neg":"pos",(pl<0?"−":"+")+fmt(Math.abs(pl))+" ฿"+(cost>0?" ("+(pl/cost*100).toFixed(1)+"%)":"")));c.append(g);
                if(u>0)line("ถือ "+px(u)+" หน่วย · ต้นทุนเฉลี่ย",px(cost/u)+" ฿");
                const when=j.pt?" · "+new Date(j.pt).toLocaleTimeString("th-TH",{hour:"2-digit",minute:"2-digit"}):"";
                line(j.sym?"ราคา "+j.sym:"ราคาต่อหน่วย",j.price>0?px(j.price)+" ฿"+when:j.perr?"ดึงราคาไม่ได้":j.sym&&!hasA()?"ดึงได้เฉพาะในแอป":"ยังไม่มีราคา");
                line("DCA เดือนละ",fmt(monthly(j))+" ฿");
                line("ประมาณการอีก "+(j.yrs||0)+" ปี ที่ "+(j.rate||0)+"%/ปี",fmt(proj(b,monthly(j),j.rate,j.yrs))+" ฿");
                {
                  const mb=el("details");mb.open=MB===j.id;mb.style.marginTop="12px";mb.append(el("summary",null,"✍️ บันทึกซื้อ/ขาย/ราคาเอง"));
                  const mg=el("div","set");mg.addEventListener("focusin",()=>{OPEN=j.id;MB=j.id});
                  const sel=el("select");for(const [v,t] of [["b","ซื้อ"],["s","ขาย"]]){const o=el("option",null,t);o.value=v;sel.append(o)}
                  mg.append(el("label",null,"ประเภทรายการ"),sel);
                  const mk=(l,t,ph)=>{mg.append(el("label",null,l));const i=el("input");i.type=t==="number"?"text":t;if(t==="number"){i.inputMode="decimal";i.min="0";i.step="any"}if(ph)i.placeholder=ph;mg.append(i);return i};
                  const fd=mk("วันที่","date");fd.value=td();
                  const fa=mk("จำนวนเงิน (บาท)","number"),fp=mk("ราคาต่อหน่วย (บาท)","number"),fu=mk("จำนวนหน่วย","number","ใส่ 2 ใน 3 ช่อง อีกช่องคำนวณให้");
                  const msg=el("div","sub");
                  const ok=btn("p","บันทึกรายการ",()=>{
                    let a=pn(fa),pr=pn(fp),un=pn(fu);if([fa,fp,fu].some(e=>e._why==="bad"))return;
                    const h=x=>x>0;
                    if(h(a)&&h(pr)&&!h(un))un=a/pr;else if(h(a)&&h(un)&&!h(pr))pr=a/un;else if(h(pr)&&h(un)&&!h(a))a=pr*un;
                    const d=fd.value||td();
                    if(sel.value==="b"){
                      if(!h(a)){msg.textContent=FILLMSG;toast(FILLMSG);return}
                      j.tx.push({id:Date.now()+Math.random(),a,d,n:"ซื้อเอง"+(h(pr)?" @"+px(pr):""),u:h(un)?un:0,c:"",k:""});
                    }else{
                      if(!h(un)&&h(a)&&j.price>0)un=a/j.price;
                      if(!h(un)||un>u+1e-9){msg.textContent="ใส่จำนวนหน่วยที่ขาย (ถืออยู่ "+px(u)+")";toast(msg.textContent);return}
                      j.tx.push({id:Date.now()+Math.random(),a:-(u>0?cost/u*un:0),d,n:"ขาย "+px(un)+" หน่วย"+(h(pr)?" @"+px(pr):"")+(h(a)?" ได้ "+px(a):""),u:-un,c:"",k:""});
                    }
                    MB=j.id;OPEN=j.id;done()});
                  mg.append(ok,msg);
                  mg.append(el("label",null,"อัปเดตราคาปัจจุบันต่อหน่วย (บาท)"));
                  const np=el("input");np.type="text";np.inputMode="decimal";np.inputMode="decimal";np.step="any";np.placeholder=j.price>0?px(j.price):"";mg.append(np);
                  mg.append(btn("v","อัปเดตราคา (ใช้ราคาที่กรอกเอง)",()=>{const x=pn(np);if(!(x>0)){need(np);return}j.price=x;j.pt=Date.now();j.mp=true;MB=j.id;OPEN=j.id;done()}));
                  mb.append(mg);extra=mb;
                }
                inp.placeholder="จำนวนเงินที่ซื้อ (บาท)";
                q.append(inp,btn("p","+ ซื้อ",()=>{
                  const a=pn(inp);if(!(a>0)){need(inp);return}
                  if(j.price>0)j.tx.push({id:Date.now()+Math.random(),a,d:td(),n:"ซื้อ @"+px(j.price),u:a/j.price});
                  else if(j.sym){OPEN=j.id;inp.value="";inp.placeholder="ยังไม่มีราคา ดูในตั้งค่า";det.open=true;return}
                  else add(j,a,"ลงทุน");
                  done()}));
                q.append(j.sym?btn("v","รีเฟรช",refresh):btn("v","อัปเดตมูลค่า",()=>{const a=pn(inp);if(!(a>=0)){need(inp);return}add(j,a-b,"ปรับเป็นมูลค่าตลาด",true);done()}));
                fld(set,"สัญลักษณ์หุ้น (เช่น PTT.BK, AAPL, SPY)",j,"sym","t");
                fld(set,"ราคาต่อหน่วย (กรอกเองถ้าไม่ดึงอัตโนมัติ)",j,"price","n");
                {const l=el("label","chk"),c2=el("input");c2.type="checkbox";c2.checked=!!j.mp;c2.onchange=()=>{OPEN=j.id;j.mp=c2.checked;done()};l.append(c2,"ใช้ราคาที่กรอกเอง (ไม่ดึงอัตโนมัติ)");set.append(l)}
                fld(set,"DCA ต่อเดือน (บาท)",j,"mon","n");fld(set,"ผลตอบแทนคาดการณ์ (%/ปี)",j,"rate","n");fld(set,"ประมาณการอีกกี่ปี",j,"yrs","n");
              }
              if(j.type==="c"){
                const m=monthly(j),goal=j.goal||0,yrs=j.yrs||0;
                line("ฝากเดือนละ",fmt(m)+" ฿");
                if(goal>0){
                  const bar=el("div","bar"),i=el("i");i.style.width=Math.max(0,Math.min(100,b/goal*100))+"%";bar.append(i);
                  line("เป้าหมาย "+fmt(goal)+" ฿",(b/goal*100).toFixed(0)+"%");c.append(bar);
                  const r=(j.rate||0)/1200,n=yrs*12,g=Math.pow(1+r,n);
                  line("อีก "+yrs+" ปี ที่ "+(j.rate||0)+"%/ปี คาดว่าได้",fmt(proj(b,m,j.rate,yrs))+" ฿");
                  if(n>0)line("ต้องฝากเดือนละเท่าไรถึงเป้า",fmt(Math.max(0,r?(goal-b*g)*r/(g-1):(goal-b)/n))+" ฿");
                }else line("ประมาณการอีก "+yrs+" ปี",fmt(proj(b,m,j.rate,yrs))+" ฿");
                inp.placeholder="จำนวนเงิน";q.append(inp,btn("m","− ถอน",go(-1)),btn("p","+ ฝาก",go(1)));
                fld(set,"ฝากต่อเดือน (บาท)",j,"mon","n");fld(set,"เป้าหมาย เช่น ค่าเทอม (บาท)",j,"goal","n");
                fld(set,"ถึงเป้าหมายในอีกกี่ปี",j,"yrs","n");fld(set,"ผลตอบแทนคาดการณ์ (%/ปี)",j,"rate","n");
              }
              if(j.type==="f"){
                const bp=bk(j,"p"),bs=bk(j,"s"),yr=j.yrs||0,pm=S.salary*((j.emp||0)+(j.er||0))/100;
                line("🏦 PVD",fmt(bp)+" ฿");
                line("🤝 ออมทรัพย์/สหกรณ์",fmt(bs)+" ฿");
                line("สมทบ PVD เดือนละ",fmt(pm)+" ฿ (เรา "+fmt(S.salary*(j.emp||0)/100)+" + นายจ้าง "+fmt(S.salary*(j.er||0)/100)+")");
                line("ส่งหุ้นสหกรณ์เดือนละ",fmt(j.mon||0)+" ฿");
                line("อีก "+yr+" ปี คาดว่า PVD / ออมทรัพย์",fmt(proj(bp,pm,j.prate,yr))+" / "+fmt(proj(bs,j.mon||0,j.srate,yr))+" ฿");
                const ir=el("div","q one");ir.style.cssText="grid-template-columns:1fr 1fr;margin-top:12px";
                ir.append(btn("v","📄 นำเข้าจาก PDF",()=>pickPdf(j)),btn("v","✍️ กรอกจากสลิป",()=>{IMP={jid:j.id,file:"กรอกจากสลิ
