# App-schedule-
يمكنك من خلال هذا التطبيق جدولة فتح تطبيقات اخرى و قفل باقي التطبيقات اي وضع الهاتف في وضع التركيز 
// MainActivity.java
package com.example.appschedule;

import androidx.appcompat.app.AppCompatActivity;
import androidx.appcompat.app.AlertDialog;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import android.Manifest;
import android.app.AlarmManager;
import android.app.PendingIntent;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.content.res.Configuration;
import android.content.res.Resources;
import android.content.pm.PackageManager;
import android.content.pm.ResolveInfo;
import android.os.Bundle;
import android.util.DisplayMetrics;
import android.view.View;
import android.widget.ArrayAdapter;
import android.widget.Button;
import android.widget.Spinner;
import android.widget.TimePicker;
import android.widget.Toast;

import java.util.Calendar;
import java.util.List;
import java.util.Locale;

public class MainActivity extends AppCompatActivity {
    private TimePicker timePicker;
    private Spinner appSpinner;
    private SharedPreferences sharedPreferences;
    private static final String PREFS_NAME = "AppSchedulePrefs";
    private static final String LANGUAGE_KEY = "language";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        sharedPreferences = getSharedPreferences(PREFS_NAME, MODE_PRIVATE);
        String language = sharedPreferences.getString(LANGUAGE_KEY, "en");
        setAppLanguage(language);
        
        setContentView(R.layout.activity_main);

        requestPermissions();
        timePicker = findViewById(R.id.timePicker);
        appSpinner = findViewById(R.id.appSpinner);
        Button scheduleButton = findViewById(R.id.scheduleButton);
        Button shutdownButton = findViewById(R.id.shutdownButton);
        Button languageButton = findViewById(R.id.languageButton);

        populateAppSpinner();

        scheduleButton.setOnClickListener(v -> scheduleAppOpening());
        shutdownButton.setOnClickListener(v -> showShutdownConfirmation());
        languageButton.setOnClickListener(v -> showLanguageDialog());
    }

    private void setAppLanguage(String languageCode) {
        Resources res = getResources();
        DisplayMetrics dm = res.getDisplayMetrics();
        Configuration conf = res.getConfiguration();
        conf.setLocale(new Locale(languageCode));
        res.updateConfiguration(conf, dm);
    }

    private void showLanguageDialog() {
        new AlertDialog.Builder(this)
            .setTitle(R.string.select_language)
            .setItems(R.array.languages_array, (dialog, which) -> {
                String language = (which == 0) ? "en" : "ar";
                sharedPreferences.edit().putString(LANGUAGE_KEY, language).apply();
                recreate();
            }).show();
    }

    private void requestPermissions() {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.SET_ALARM)
                != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this,
                    new String[]{Manifest.permission.SET_ALARM}, 1);
        }
    }

    private void populateAppSpinner() {
        Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);
        mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);
        List<ResolveInfo> appList = getPackageManager().queryIntentActivities(mainIntent, 0);
        String[] appNames = new String[appList.size()];

        for (int i = 0; i < appList.size(); i++) {
            appNames[i] = appList.get(i).activityInfo.applicationInfo.loadLabel(getPackageManager()).toString();
        }

        ArrayAdapter<String> adapter = new ArrayAdapter<>(this,
                android.R.layout.simple_spinner_item, appNames);
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        appSpinner.setAdapter(adapter);
    }

    private void scheduleAppOpening() {
        int hour = timePicker.getCurrentHour();
        int minute = timePicker.getCurrentMinute();
        String selectedApp = appSpinner.getSelectedItem().toString();

        Calendar calendar = Calendar.getInstance();
        calendar.set(Calendar.HOUR_OF_DAY, hour);
        calendar.set(Calendar.MINUTE, minute);
        calendar.set(Calendar.SECOND, 0);

        Intent intent = new Intent(this, AlarmReceiver.class);
        intent.putExtra("appName", selectedApp);
        PendingIntent pendingIntent = PendingIntent.getBroadcast(this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);

        AlarmManager alarmManager = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
        alarmManager.setExact(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(), pendingIntent);

        Toast.makeText(this, getString(R.string.scheduled_success, selectedApp, hour, minute), Toast.LENGTH_LONG).show();
    }

    private void showShutdownConfirmation() {
        new AlertDialog.Builder(this)
            .setTitle(R.string.confirm_shutdown_title)
            .setMessage(R.string.confirm_shutdown_message)
            .setPositiveButton(R.string.yes, (dialog, which) -> shutdownOtherApps())
            .setNegativeButton(R.string.no, null)
            .show();
    }

    private void shutdownOtherApps() {
        sendBroadcast(new Intent(this, ShutdownReceiver.class));
        Toast.makeText(this, R.string.apps_closing, Toast.LENGTH_SHORT).show();
    }
}

// AlarmReceiver.java
package com.example.appschedule;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.widget.Toast;

public class AlarmReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        String appName = intent.getStringExtra("appName");
        try {
            PackageManager pm = context.getPackageManager();
            Intent launchIntent = pm.getLaunchIntentForPackage(getPackageName(context, appName));
            if (launchIntent != null) {
                launchIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                context.startActivity(launchIntent);
                Toast.makeText(context, context.getString(R.string.opening_app, appName), Toast.LENGTH_SHORT).show();
            }
        } catch (Exception e) {
            Toast.makeText(context, context.getString(R.string.app_open_error, appName), Toast.LENGTH_SHORT).show();
        }
    }

    private String getPackageName(Context context, String appName) {
        PackageManager pm = context.getPackageManager();
        Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);
        mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);
        for (ResolveInfo info : pm.queryIntentActivities(mainIntent, 0)) {
            if (appName.equals(info.loadLabel(pm).toString())) {
                return info.activityInfo.packageName;
            }
        }
        return "";
    }
}

// ShutdownReceiver.java
package com.example.appschedule;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.os.Build;
import android.app.ActivityManager;

public class ShutdownReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            ((ActivityManager)context.getSystemService(Context.ACTIVITY_SERVICE))
                    .killBackgroundProcesses(null);
        }
    }
}

// strings.xml (English)
<resources>
    <string name="app_name">App Scheduler</string>
    <string name="select_language">Select Language</string>
    <string name="scheduled_success">Scheduled to open %1$s at %2$d:%3$d</string>
    <string name="confirm_shutdown_title">Close Other Apps</string>
    <string name="confirm_shutdown_message">Are you sure you want to close all other apps?</string>
    <string name="yes">Yes</string>
    <string name="no">No</string>
    <string name="apps_closing">Closing other apps...</string>
    <string name="opening_app">Opening %s</string>
    <string name="app_open_error">Failed to open %s</string>
    <string name="schedule_button">Schedule App</string>
    <string name="shutdown_button">Close Other Apps</string>
    <string-array name="languages_array">
        <item>English</item>
        <item>Arabic</item>
    </string-array>
</resources>

// strings.xml (Arabic - values-ar)
<resources>
    <string name="app_name">جدولة التطبيقات</string>
    <string name="select_language">اختر اللغة</string>
    <string name="scheduled_success">تم جدولة فتح %1$s في الساعة %2$d:%3$d</string>
    <string name="confirm_shutdown_title">إغلاق التطبيقات الأخرى</string>
    <string name="confirm_shutdown_message">هل أنت متأكد أنك تريد إغلاق جميع التطبيقات الأخرى؟</string>
    <string name="yes">نعم</string>
    <string name="no">لا</string>
    <string name="apps_closing">جاري إغلاق التطبيقات الأخرى...</string>
    <string name="opening_app">جاري فتح %s</string>
    <string name="app_open_error">فشل فتح %s</string>
    <string name="schedule_button">جدولة التطبيق</string>
    <string name="shutdown_button">إغلاق التطبيقات الأخرى</string>
    <string-array name="languages_array">
        <item>الإنجليزية</item>
        <item>العربية</item>
    </string-array>
</resources>

// activity_main.xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <Button
        android:id="@+id/languageButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="end"
        android:text="@string/select_language"/>

    <TimePicker
        android:id="@+id/timePicker"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_horizontal"/>

    <Spinner
        android:id="@+id/appSpinner"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="20dp"/>

    <Button
        android:id="@+id/scheduleButton"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="20dp"
        android:text="@string/schedule_button"/>

    <Button
        android:id="@+id/shutdownButton"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="10dp"
        android:text="@string/shutdown_button"/>
</LinearLayout>

// AndroidManifest.xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.appschedule">

    <uses-permission android:name="android.permission.SET_ALARM"/>
    <uses-permission android:name="android.permission.KILL_BACKGROUND_PROCESSES"/>
    <uses-permission android:name="android.permission.REORDER_TASKS"/>

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <receiver android:name=".AlarmReceiver"/>
        <receiver android:name=".ShutdownReceiver"/>
    </application>
</manifest>
