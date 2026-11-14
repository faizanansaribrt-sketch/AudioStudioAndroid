import os, textwrap, zipfile

root = "/mnt/data/AudioStudioAndroid"
os.makedirs(root + "/app/src/main/java/com/example/audiostudio", exist_ok=True)
os.makedirs(root + "/app/src/main/res/values", exist_ok=True)

files = {
"settings.gradle.kts": """pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement { repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS); repositories { google(); mavenCentral() } }
rootProject.name = "AudioStudio"
include(":app")
""",
"build.gradle.kts": """plugins {
    id("com.android.application") version "8.7.3" apply false
    id("org.jetbrains.kotlin.android") version "2.0.21" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.0.21" apply false
}
""",
"gradle.properties": """org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
""",
"app/build.gradle.kts": """plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
}
android {
    namespace = "com.example.audiostudio"
    compileSdk = 35
    defaultConfig {
        applicationId = "com.example.audiostudio"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }
    buildFeatures { compose = true }
}
dependencies {
    implementation(platform("androidx.compose:compose-bom:2024.12.01"))
    implementation("androidx.activity:activity-compose:1.10.0")
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.material:material-icons-extended")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")
}
""",
"app/src/main/AndroidManifest.xml": """<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.RECORD_AUDIO"/>
    <application android:theme="@style/AppTheme" android:label="Audio Studio" android:allowBackup="false">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
""",
"app/src/main/res/values/styles.xml": """<resources>
    <style name="AppTheme" parent="android:style/Theme.Material.NoActionBar">
        <item name="android:fontFamily">sans</item>
        <item name="android:windowLightStatusBar">false</item>
        <item name="android:statusBarColor">#101116</item>
        <item name="android:navigationBarColor">#101116</item>
        <item name="android:windowBackground">#101116</item>
    </style>
</resources>
""",
"app/src/main/java/com/example/audiostudio/MainActivity.kt": r'''package com.example.audiostudio

import android.media.MediaPlayer
import android.net.Uri
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.background
import androidx.compose.foundation.horizontalScroll
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material.icons.filled.Pause
import androidx.compose.material.icons.filled.PlayArrow
import androidx.compose.material.icons.filled.Stop
import androidx.compose.material.icons.filled.VolumeUp
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.compose.ui.platform.LocalContext
import kotlinx.coroutines.delay

private val bg = Color(0xFF101116)
private val panel = Color(0xFF1B1D26)
private val muted = Color(0xFF9297A8)
private val cyan = Color(0xFF35D0C2)
private val purple = Color(0xFF9B86FF)

data class AudioTrackItem(
    val id: Long,
    val uri: Uri,
    val name: String,
    var muted: Boolean = false,
    var solo: Boolean = false,
    var volume: Float = 1f
)

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { AudioStudioApp() }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AudioStudioApp() {
    val context = LocalContext.current
    val tracks = remember { mutableStateListOf<AudioTrackItem>() }
    val players = remember { mutableStateMapOf<Long, MediaPlayer>() }
    var playing by remember { mutableStateOf(false) }
    var playhead by remember { mutableFloatStateOf(0f) }
    var master by remember { mutableFloatStateOf(0.85f) }
    var status by remember { mutableStateOf("Ready · Import audio to begin") }

    val picker = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenMultipleDocuments()
    ) { uris ->
        uris.forEach { uri ->
            runCatching { context.contentResolver.takePersistableUriPermission(uri, android.content.Intent.FLAG_GRANT_READ_URI_PERMISSION) }
            val name = uri.lastPathSegment?.substringAfterLast('/')?.substringAfterLast(':') ?: "Audio track ${tracks.size + 1}"
            tracks.add(AudioTrackItem(System.nanoTime(), uri, name))
        }
        if (uris.isNotEmpty()) status = "${uris.size} track(s) added"
    }

    fun stopAll() {
        players.values.forEach { runCatching { it.pause(); it.seekTo(0) } }
        playing = false
        playhead = 0f
        status = "Playback stopped"
    }

    fun startAll() {
        if (tracks.isEmpty()) { status = "Import an audio file first"; return }
        val anySolo = tracks.any { it.solo }
        tracks.forEach { track ->
            if (!track.muted && (!anySolo || track.solo)) {
                val player = players[track.id] ?: MediaPlayer().apply {
                    setDataSource(context, track.uri)
                    setOnPreparedListener { mp ->
                        mp.setVolume(track.volume * master, track.volume * master)
                        mp.start()
                    }
                    prepareAsync()
                    players[track.id] = this
                }
                if (player.isPlaying) return@forEach
                if (player.currentPosition > 0) player.seekTo(0)
                runCatching { player.start() }
            }
        }
        playing = true
        status = "Playing ${tracks.size} track(s)"
    }

    DisposableEffect(Unit) {
        onDispose { players.values.forEach { runCatching { it.release() } }; players.clear() }
    }

    LaunchedEffect(playing) {
        while (playing) {
            delay(100)
            playhead = (playhead + 0.1f) % 300f
            if (players.values.none { runCatching { it.isPlaying }.getOrDefault(false) }) {
                playing = false
                status = "Playback finished"
            }
        }
    }

    MaterialTheme(colorScheme = darkColorScheme(
        background = bg, surface = panel, primary = cyan,
        onBackground = Color.White, onSurface = Color.White
    )) {
        Scaffold(containerColor = bg) { padding ->
            Column(Modifier.fillMaxSize().padding(padding).padding(horizontal = 18.dp)) {
                Row(Modifier.fillMaxWidth().padding(top = 18.dp), verticalAlignment = Alignment.CenterVertically) {
                    Column(Modifier.weight(1f)) {
                        Text("AUDIO", color = cyan, fontSize = 11.sp, fontWeight = FontWeight.Bold, letterSpacing = 3.sp)
                        Text("STUDIO", color = Color.White, fontSize = 27.sp, fontWeight = FontWeight.ExtraBold)
                    }
                    Surface(color = Color(0xFF202B31), shape = RoundedCornerShape(50)) {
                        Text("MULTITRACK", Modifier.padding(horizontal = 12.dp, vertical = 7.dp), color = cyan, fontSize = 10.sp, fontWeight = FontWeight.Bold)
                    }
                }
                Text("Professional audio workspace", color = muted, fontSize = 12.sp, modifier = Modifier.padding(top = 2.dp, bottom = 18.dp))

                Row(horizontalArrangement = Arrangement.spacedBy(10.dp)) {
                    MetricCard("TRACKS", tracks.size.toString(), Modifier.weight(1f))
                    MetricCard("SESSION", "48 kHz*", Modifier.weight(1f))
                    MetricCard("MASTER", "${(master * 100).toInt()}%", Modifier.weight(1f))
                }

                Spacer(Modifier.height(16.dp))
                Surface(color = panel, shape = RoundedCornerShape(18.dp), modifier = Modifier.fillMaxWidth()) {
                    Column(Modifier.padding(14.dp)) {
                        Row(verticalAlignment = Alignment.CenterVertically) {
                            Text("TIMELINE", color = muted, fontSize = 11.sp, fontWeight = FontWeight.Bold, modifier = Modifier.weight(1f))
                            Text("00:${playhead.toInt().toString().padStart(2, '0')}", color = cyan, fontSize = 13.sp, fontWeight = FontWeight.Bold)
                        }
                        Spacer(Modifier.height(12.dp))
                        Canvas(Modifier.fillMaxWidth().height(88.dp)) {
                            val mid = size.height / 2
                            for (i in 0 until 100) {
                                val x = i * size.width / 100f
                                val h = (8 + ((i * 37) % 29)).toFloat()
                                drawLine(Color(0xFF2C5558), Offset(x, mid-h), Offset(x, mid+h), 2f)
                            }
                            val cursor = (playhead / 300f) * size.width
                            drawLine(cyan, Offset(cursor, 0f), Offset(cursor, size.height), 3f)
                        }
                        Row(verticalAlignment = Alignment.CenterVertically) {
                            Text("00:00", color = muted, fontSize = 10.sp)
                            Spacer(Modifier.weight(1f))
                            Text("05:00", color = muted, fontSize = 10.sp)
                        }
                    }
                }

                Spacer(Modifier.height(14.dp))
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Text("TRACKS", color = Color.White, fontWeight = FontWeight.Bold, fontSize = 14.sp, modifier = Modifier.weight(1f))
                    Text("${tracks.size} layers", color = muted, fontSize = 11.sp)
                    Spacer(Modifier.width(10.dp))
                    FilledTonalButton(onClick = { picker.launch(arrayOf("audio/*")) }, contentPadding = PaddingValues(horizontal = 12.dp, vertical = 6.dp)) {
                        Icon(Icons.Default.Add, null, Modifier.size(17.dp))
                        Spacer(Modifier.width(4.dp))
                        Text("Import", fontSize = 12.sp)
                    }
                }
                Spacer(Modifier.height(8.dp))

                if (tracks.isEmpty()) {
                    Surface(color = panel, shape = RoundedCornerShape(16.dp), modifier = Modifier.fillMaxWidth().weight(1f)) {
                        Column(Modifier.fillMaxSize(), horizontalAlignment = Alignment.CenterHorizontally, verticalArrangement = Arrangement.Center) {
                            Text("＋", color = cyan, fontSize = 38.sp)
                            Text("Your session is empty", color = Color.White, fontWeight = FontWeight.Bold)
                            Text("Import multiple audio files to create layers", color = muted, fontSize = 12.sp)
                        }
                    }
                } else {
                    Column(Modifier.weight(1f).verticalScroll(rememberScrollState()), verticalArrangement = Arrangement.spacedBy(9.dp)) {
                        tracks.forEachIndexed { index, track ->
                            TrackCard(
                                track = track, index = index,
                                onMute = { tracks[index] = track.copy(muted = !track.muted) },
                                onSolo = { tracks[index] = track.copy(solo = !track.solo) },
                                onVolume = { value ->
                                    tracks[index] = track.copy(volume = value)
                                    players[track.id]?.setVolume(value * master, value * master)
                                },
                                onRemove = {
                                    players.remove(track.id)?.release()
                                    tracks.removeAt(index)
                                }
                            )
                        }
                    }
                }

                Surface(color = Color(0xFF1C2029), shape = RoundedCornerShape(18.dp), modifier = Modifier.fillMaxWidth().padding(top = 12.dp, bottom = 8.dp)) {
                    Column(Modifier.padding(14.dp)) {
                        Row(verticalAlignment = Alignment.CenterVertically) {
                            Icon(Icons.Default.VolumeUp, null, tint = cyan, modifier = Modifier.size(18.dp))
                            Text(" MASTER OUTPUT", color = muted, fontSize = 10.sp, fontWeight = FontWeight.Bold, modifier = Modifier.weight(1f))
                            Text("${(master*100).toInt()}%", color = cyan, fontSize = 12.sp)
                        }
                        Slider(value = master, onValueChange = { master = it; players.values.forEach { p -> runCatching { p.setVolume(it, it) } } }, colors = SliderDefaults.colors(thumbColor = cyan, activeTrackColor = cyan))
                        Row(horizontalArrangement = Arrangement.spacedBy(10.dp), verticalAlignment = Alignment.CenterVertically) {
                            OutlinedButton(onClick = { stopAll() }, modifier = Modifier.weight(1f)) {
                                Icon(Icons.Default.Stop, null); Text(" Stop")
                            }
                            Button(onClick = { if (playing) { players.values.forEach { it.pause() }; playing = false; status = "Paused" } else startAll() }, modifier = Modifier.weight(1.4f), colors = ButtonDefaults.buttonColors(containerColor = cyan, contentColor = Color(0xFF071516))) {
                                Icon(if (playing) Icons.Default.Pause else Icons.Default.PlayArrow, null)
                                Text(if (playing) " Pause" else " Play mix", fontWeight = FontWeight.Bold)
                            }
                        }
                    }
                }
                Text(status, color = muted, fontSize = 10.sp, modifier = Modifier.padding(bottom = 8.dp))
                Text("*48 kHz is a session label; source files retain their original format.", color = Color(0xFF626879), fontSize = 9.sp, modifier = Modifier.padding(bottom = 10.dp))
            }
        }
    }
}

@Composable
fun MetricCard(label: String, value: String, modifier: Modifier = Modifier) {
    Surface(color = panel, shape = RoundedCornerShape(12.dp), modifier = modifier) {
        Column(Modifier.padding(11.dp)) {
            Text(label, color = muted, fontSize = 9.sp, fontWeight = FontWeight.Bold)
            Text(value, color = Color.White, fontSize = 16.sp, fontWeight = FontWeight.Bold)
        }
    }
}

@Composable
fun TrackCard(track: AudioTrackItem, index: Int, onMute: () -> Unit, onSolo: () -> Unit, onVolume: (Float) -> Unit, onRemove: () -> Unit) {
    Surface(color = panel, shape = RoundedCornerShape(15.dp), modifier = Modifier.fillMaxWidth()) {
        Column(Modifier.padding(12.dp)) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Surface(color = if (index % 2 == 0) Color(0xFF203D40) else Color(0xFF302A4A), shape = RoundedCornerShape(8.dp)) {
                    Text(" ${index + 1} ", color = if (index % 2 == 0) cyan else purple, modifier = Modifier.padding(6.dp), fontWeight = FontWeight.Bold, fontSize = 12.sp)
                }
                Spacer(Modifier.width(9.dp))
                Column(Modifier.weight(1f)) {
                    Text(track.name, color = Color.White, fontWeight = FontWeight.SemiBold, fontSize = 12.sp, maxLines = 1)
                    Text("AUDIO LAYER · ${(track.volume*100).toInt()}% gain", color = muted, fontSize = 9.sp)
                }
                TextButton(onClick = onRemove, contentPadding = PaddingValues(0.dp)) { Text("REMOVE", color = Color(0xFFFF7D88), fontSize = 9.sp) }
            }
            Canvas(Modifier.fillMaxWidth().height(38.dp).padding(top = 5.dp)) {
                val color = if (index % 2 == 0) cyan else purple
                val mid = size.height / 2
                for (i in 0 until 150) {
                    val x = i * size.width / 150f
                    val amp = (4 + ((i * (index + 13) * 17) % 23)).toFloat()
                    drawLine(color.copy(alpha = 0.25f + (i % 5) * 0.1f), Offset(x, mid-amp), Offset(x, mid+amp), 2f)
                }
            }
            Row(verticalAlignment = Alignment.CenterVertically) {
                FilterChip(selected = track.muted, onClick = onMute, label = { Text(if (track.muted) "MUTED" else "MUTE", fontSize = 9.sp) })
                Spacer(Modifier.width(5.dp))
                FilterChip(selected = track.solo, onClick = onSolo, label = { Text(if (track.solo) "SOLO ON" else "SOLO", fontSize = 9.sp) })
                Spacer(Modifier.width(8.dp))
                Slider(value = track.volume, onValueChange = onVolume, modifier = Modifier.weight(1f), colors = SliderDefaults.colors(thumbColor = cyan, activeTrackColor = cyan))
            }
        }
    }
}
''',
"README.md": """# Audio Studio (Android)

A dark-theme Android multitrack audio-session starter project built with Kotlin and Jetpack Compose.

## Included
- Import multiple audio files using Android's document picker
- Track list with individual volume, mute, solo and remove controls
- Simultaneous playback of eligible tracks
- Master output control, play, pause and stop
- Dark professional studio-style interface

## Build an APK
1. Install Android Studio (Ladybug or newer) and Android SDK Platform 35.
2. Open this folder in Android Studio.
3. Allow Gradle sync to finish.
4. Select **Build > Build Bundle(s) / APK(s) > Build APK(s)**.
5. Find the debug APK under `app/build/outputs/apk/debug/app-debug.apk`.

## Important limitations
This is a functional starter, not a finished DAW. The waveform display is a visual placeholder, not a decoded audio waveform. Playback uses Android MediaPlayer and does not yet provide sample-accurate sync, trimming, effects, automation, recording, or a rendered/exported mix. Track playback begins when each player is prepared, so precise synchronization is not guaranteed. A production studio app needs a dedicated low-latency audio engine and an audio-mix/export pipeline.
"""
}

for rel, content in files.items():
    path = os.path.join(root, rel)
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        f.write(content)

zip_path = "/mnt/data/AudioStudioAndroid_Project.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for folder, _, names in os.walk(root):
        for name in names:
            full = os.path.join(folder, name)
            z.write(full, os.path.relpath(full, root))
print(zip_path)
# AudioStudioAndroid
