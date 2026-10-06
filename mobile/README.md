# Mobile — Aplikasi Ujian (Flutter, Android)

Proyek Flutter untuk **dua aplikasi Android** (bukan iOS):

| Aplikasi | Application ID | Flavor |
|---|---|---|
| Siswa | `id.smpn5tangerang.ujian_siswa` | `siswa` |
| Pengawas | `id.smpn5tangerang.ujian_pengawas` | `pengawas` |

Satu codebase, dua product flavor. Rilis ke **Google Play Store** via **fastlane + GitHub Actions** (internal → closed testing → production).

Ringkasan tech stack & konteks proyek: [`../CLAUDE.md`](../CLAUDE.md), [`../README.md`](../README.md).

---

## 1. Prasyarat

- **Flutter 3.x** (channel stable) + Dart 3.x — `flutter --version`
- **Android Studio** (SDK terbaru + SDK Build-Tools, emulator/api 34+ disarankan)
- **JDK 17**
- Perangkat/emulator Android dengan **minSdk 21** ke atas
- Akses ke server WebMin (base URL API dev/staging/prod)


```bash
flutter doctor   # pastikan toolchain Android siap
```

## 2. Scaffold proyek

Dari **root repo**:

```bash
flutter create --org id.smpn5tangerang --project-name ujian_mobile --platforms android mobile
cd mobile
```

## 3. Dependensi (`pubspec.yaml`)

```yaml
dependencies:
  flutter:
    sdk: flutter
  # State & navigasi
  flutter_riverpod: ^2.5.0
  riverpod_annotation: ^2.3.0
  go_router: ^14.0.0
  # Networking (interceptor: auth refresh, HMAC signature, retry backoff)
  dio: ^5.4.0
  # Model & serialisasi
  freezed_annotation: ^2.4.0
  json_annotation: ^4.9.0
  # Offline-first (local store + job queue sinkronisasi)
  drift: ^2.16.0
  sqlite3_flutter_libs: ^0.5.0
  path_provider: ^2.1.0
  # Keamanan
  flutter_secure_storage: ^9.0.0     # token JWT
  device_info_plus: ^10.1.0          # device binding
  # Notifikasi (pengawas)
  firebase_core: ^3.0.0
  firebase_messaging: ^14.7.0
  # Observability
  sentry_flutter: ^8.0.0

dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^2.5.0
  json_serializable: ^6.8.0
  riverpod_generator: ^2.4.0
  drift_dev: ^2.16.0
  flutter_lints: ^4.0.0
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
```

```bash
flutter pub get
```

## 4. Konfigurasi Android

### 4.1 Flavor (`android/app/build.gradle.kts`)

```kotlin
android {
    defaultConfig { minSdk = 21 }
    flavorDimensions += "app"
    productFlavors {
        create("siswa") {
            dimension = "app"
            applicationId = "id.smpn5tangerang.ujian_siswa"
        }
        create("pengawas") {
            dimension = "app"
            applicationId = "id.smpn5tangerang.ujian_pengawas"
        }
    }
}
```

`targetSdk` mengikuti kebijakan terbaru Play Store (cek requirement saat submit).

### 4.2 Permission — minimalis saja

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<!-- TIDAK ada akses lokasi/kamera kecuali fitur proktor memang membutuhkannya -->
```

### 4.3 Firebase (khusus pengawas — FCM)

Daftarkan **dua** aplikasi Firebase sesuai application ID flavor. Unduh `google-services.json` dari Firebase Console → taruh di `android/app/src/<flavor>/` (siswa tanpa FCM boleh kosong). Tambahkan plugin `com.google.gms.google-services` di `android/build.gradle.kts` dan aplikasi plugin di `android/app/build.gradle.kts`.

### 4.4 Root detection (aplikasi siswa)

Deteksi root di build siswa untuk menolak perangkat berisiko — gunakan `root_checker`, dikombinasikan dengan `device_info_plus` untuk device binding dan evaluasi risiko di server.

## 5. Environment via `--dart-define`

Buat file config (satu per env, **tidak commit secret**):

```jsonc
// env/dev.json
{ "BASE_URL": "http://10.0.2.2:3000/api/v1", "HMAC_KEY": "<key-dev>", "SENTRY_DSN": "" }
```

```bash
flutter run --flavor siswa -t lib/main_siswa.dart \
  --dart-define-from-file=env/dev.json

flutter run --flavor pengawas -t lib/main_pengawas.dart \
  --dart-define-from-file=env/staging.json
```

Env: `dev` / `staging` / `prod`. Baca dengan `String.fromEnvironment`.

## 6. Codegen (freezed, json_serializable, riverpod, drift)

```bash
dart run build_runner build --delete-conflicting-outputs   # sekali
dart run build_runner watch --delete-conflicting-outputs   # saat develop
```

## 7. Struktur folder

```
mobile/
├── android/
│   └── app/src/{siswa,pengawas}/     # manifest & google-services per flavor
├── env/                              # dev.json, staging.json, prod.json
├── lib/
│   ├── main_siswa.dart
│   ├── main_pengawas.dart
│   ├── app/            # router (go_router), tema, bootstrap Riverpod
│   ├── core/
│   │   ├── api/        # dio + interceptor (auth refresh, HMAC X-Signature/X-Nonce/X-Timestamp, retry)
│   │   ├── db/         # drift: local store + sync job queue (server-wins)
│   │   └── security/   # secure storage, device binding, root detection
│   ├── features/       # ujian_siswa, pemantauan_pengawas, laporan, notifikasi
│   └── shared/         # widgets, utils
└── test/  integration_test/
```

## 8. Testing

```bash
flutter test                                   # unit & widget
flutter test integration_test                  # E2E (perangkat/emukator tersambung)
flutter analyze
```

---

## 9. Build untuk Google Play

Play Store **wajib AAB** (`appbundle`), bukan APK.

```bash
# Siswa — production
flutter build appbundle --release --flavor siswa   -t lib/main_siswa.dart   --dart-define-from-file=env/prod.json

# Pengawas — production
flutter build appbundle --release --flavor pengawas -t lib/main_pengawas.dart --dart-define-from-file=env/prod.json
```

Output: `build/app/outputs/bundle/<flavor>Release/app-<flavor>-release.aab`.

## 10. Signing (Play App Signing + keystore release)

Kita pakai **Play App Signing** (Google pegang App Signing Key; kita pegang **Upload Key**).

1. Generate upload keystore (sekali, simpan aman — **di luar repo**):

   ```bash
   keytool -genkeypair -v -storetype PKCS12 \
     -keystore webmin-upload-keystore.p12 \
     -alias webmin-upload -keyalg RSA -keysize 2048 -validity 10000
   ```

2. Simpan kredensial di `android/key.properties` — **jangan commit** (masuk `.gitignore`), supply lewat **CI secret**:

   ```properties
   storePassword=...
   keyPassword=...
   keyAlias=webmin-upload
   storeFile=/run/secrets/webmin-upload-keystore.p12
   ```

3. Daftarkan kunci upload ke Play Console (App signing → Upload key certificate).

4. Konfigurasi `signingConfigs` di `android/app/build.gradle.kts` membaca `key.properties` untuk build release.

> Lost upload key? Revoke & upload key baru via Play Console. Jangan simpan keystore di laptop tunggal — minimal di password manager + backup offline.

## 11. Alur rilis (fastlane + GitHub Actions)

Tahap wajib per aplikasi:

```
internal (1–2 hari uji)  →  closed testing (≥12 tester aktif selama 14 hari)  →  production
```

- **Closed testing** untuk aplikasi baru harus lolos review akses Play: sediakan **≥12 tester email list** (guru/pengawas tetap) dan biarkan aktif 14 hari.
- Setelan **testing track** harus memuat info privasi & deskripsi uji.

Struktur fastlane:

```yaml
# mobile/fastlane/Appfile — per aplikasi (dua Play Console app)
android_package_name: "id.smpn5tangerang.ujian_siswa"

# mobile/fastlane/Snapfile / metadata/
metadata/
  android/siswa/en-US/, android/siswa/id-ID/
  android/pengawas/en-US/, android/pengawas/id-ID/
```

```ruby
# mobile/fastlane/Fastfile
platforms :android do
  desc "Rilis aplikasi siswa"
  lane :siswa do |options|
    gradle(task: "bundle", build_flavor: "siswa", build_type: "release")
    upload_to_play_store(
      track: options[:track] || "internal",   # internal | closed | production
      aab: "build/app/outputs/bundle/siswaRelease/app-siswa-release.aab",
      metadata_path: "fastlane/metadata/android/siswa"
    )
  end

  desc "Rilis aplikasi pengawas"
  lane :pengawas do |options|
    gradle(task: "bundle", build_flavor: "pengawas", build_type: "release")
    upload_to_play_store(
      track: options[:track] || "internal",
      aab: "build/app/outputs/bundle/pengawasRelease/app-pengawas-release.aab",
      metadata_path: "fastlane/metadata/android/pengawas"
    )
  end
end
```

GitHub Actions (ringkas):

```yaml
# .github/workflows/release-mobile.yml (potongan)
on:
  workflow_dispatch:
    inputs:
      app:   { type: choice, options: [siswa, pengawas] }
      track: { type: choice, options: [internal, closed, production] }
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4      # JDK 17
      - uses: subosito/flutter-action@v2 # Flutter stable
      - run: cd mobile && flutter pub get && dart run build_runner build --delete-conflicting-outputs
      - name: Pasang keystore & kredensial (dari secrets)
        run: |
          echo "${{ secrets.UPLOAD_KEYSTORE_B64 }}" | base64 -d > webmin-upload-keystore.p12
          printf "storePassword=%s\nkeyPassword=%s\nkeyAlias=webmin-upload\nstoreFile=%s\n" \
            "${{ secrets.KEYSTORE_PASSWORD }}" "${{ secrets.KEY_PASSWORD }}" \
            "$(pwd)/webmin-upload-keystore.p12" > mobile/android/key.properties
      - uses: ruby/setup-ruby@v1
      - run: gem install fastlane
      - name: Build & upload ke Play
        env:
          SUPPLY_JSON_KEY_DATA: "${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}"
        run: cd mobile && fastlane android ${{ inputs.app }} track:${{ inputs.track }}
```

Secrets CI: `UPLOAD_KEYSTORE_B64`, `KEYSTORE_PASSWORD`, `KEY_PASSWORD`, `PLAY_SERVICE_ACCOUNT_JSON` (service account Play Console, role *Release manager*).

## 12. Kepatuhan Play Store (sebelum submit)

- [ ] **Privacy policy** (URL publik) untuk kedua aplikasi — wajib, meskipun data hanya disimpan di sekolah.
- [ ] **Data safety form** diisi akurat: data siswa (identitas, jawaban ujian, bukti pelanggaran) → jelaskan pengumpulan, tujuan, retensi; tandai *not shared* kecuali ke processor (Sentry/Firebase).
- [ ] **Permission minimal**: hanya INTERNET (+ POST_NOTIFICATIONS untuk pengawas). Location/kamera hanya jika fitur proktor aktif — jika tidak, hapus dari manifest.
- [ ] **Account sekolah (G Suite)** untuk Play Console; satu developer account untuk kedua aplikasi.
- [ ] **Target API level** mengikuti kebijakan Play terbaru saat submit (naikkan targetSdk tiap tahun kebijakan berubah).
- [ ] **AAB** (`flutter build appbundle`), bukan APK.
- [ ] Halaman Play: screenshot (≥2), deskripsi id-ID, kategori *Education*, kontak dukungan.
- [ ] Aplikasi siswa: konten sesuai usia anak (Data Safety + family policy bila ditargetkan ke anak — perhatikan *Designed for Families*).

## 13. Troubleshooting rilis

| Masalah | Solusi |
|---|---|
| `Upload certificate does not match` | Pastikan build CI memakai upload keystore yang sama dengan yang terdaftar di Play Console. |
| Closed testing tidak lolos review | Tambah tester aktif ≥12 dan pertahankan 14 hari sebelum ajukan production. |
| Ditolak: permission berlebihan | Hapus permission tak terpakai dari manifest & Data safety. |
| `minSdk` warning Google Play | Naikkan ke angka terbaru yang diminta Play saat itu. |
