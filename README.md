# gtps-BALLS-PS
#!/usr/bin/env bash
set -euo pipefail

########## CONFIG (ubah bila perlu) ##########
INPUT_APK="$1"                           # arg1: APK asli yang kamu punya
OUT_DIR="out_grow_decompiled"
UNSIGNED_APK="unsigned_grow.apk"
ALIGNED_APK="aligned_grow.apk"
SIGNED_APK="grow_custom_signed.apk"

# keystore (jika belum ada, script akan buat)
KEYSTORE="grow_keystore.jks"
KEY_ALIAS="growkey"
KEYSTORE_PASS="password123"              # ubah ke password aman
KEY_PASS="$KEYSTORE_PASS"

# daftar penggantian host/domain -> target IP/domain
# format: "search|replace"
REPLACES=(
"growtopia1.com|91.134.85.13"
"growtopia2.com|91.134.85.13"
"www.growtopia1.com|91.134.85.13"
"www.growtopia2.com|91.134.85.13"
)

# path ke tools (bila tidak di PATH, ganti sesuai)
APKTOOL="apktool"         # apktool harus ada di PATH
ZIPALIGN="zipalign"       # dari Android SDK build-tools
APKSIGNER="apksigner"     # dari Android SDK build-tools
##############################################

if [ -z "$INPUT_APK" ] || [ ! -f "$INPUT_APK" ]; then
  echo "Usage: $0 path/to/input.apk"
  exit 1
fi

echo "[*] membersihkan direktori lama..."
rm -rf "$OUT_DIR" "$UNSIGNED_APK" "$ALIGNED_APK" "$SIGNED_APK"

echo "[*] decompile APK dengan apktool..."
$APKTOOL d -f "$INPUT_APK" -o "$OUT_DIR"

echo "[*] mencari file yang mengandung domain asli..."
# cari file yang mengandung domain / host; simpan daftar file
MATCH_FILES=$(grep -RIlE "growtopia1\.com|growtopia2\.com|www\.growtopia1\.com|www\.growtopia2\.com" "$OUT_DIR" || true)

if [ -z "$MATCH_FILES" ]; then
  echo "[!] Tidak menemukan domain di teks smali/ressources. Mencari string biner di lib/..."
  # cari string di lib native
  for lib in $(find "$OUT_DIR" -type f -name "*.so" -o -name "*.bin" 2>/dev/null); do
    strings "$lib" | grep -E "growtopia1\.com|growtopia2\.com" && echo "Found in $lib"
  done || true
fi

echo "[*] melakukan penggantian pada file teks (smali/resources/assets)..."
for rep in "${REPLACES[@]}"; do
  SEARCH="${rep%%|*}"
  REPL="${rep##*|}"
  # escape untuk perl
  ESC_SEARCH=$(printf '%s\n' "$SEARCH" | sed -e 's/[]\/$*.^[]/\\&/g')
  ESC_REPL=$(printf '%s\n' "$REPL" | sed -e 's/[]\/$*.^[]/\\&/g')
  # replace di semua file teks dalam out dir
  perl -i -pe "s/$ESC_SEARCH/$ESC_REPL/g" $(grep -RIl --binary-files=without-match "$SEARCH" "$OUT_DIR" || true) 2>/dev/null || true
done

# Double-check occurrences left
echo "[*] cek apakah masih ada referensi lama..."
if grep -RInE "growtopia1\.com|growtopia2\.com|www\.growtopia1\.com|www\.growtopia2\.com" "$OUT_DIR"; then
  echo "[!] Warning: Masih ada referensi domain lama. Mungkin tersimpan di binary (.so). Perlu editing manual atau rebuild native."
else
  echo "[*] Semua referensi server lama seharusnya sudah terganti (jika hanya di teks)."
fi

echo "[*] build ulang APK..."
$APKTOOL b "$OUT_DIR" -o "$UNSIGNED_APK"

echo "[*] membuat keystore jika belum ada..."
if [ ! -f "$KEYSTORE" ]; then
  echo "[*] membuat keystore self-signed (keytool)..."
  keytool -genkeypair -v \
    -keystore "$KEYSTORE" \
    -alias "$KEY_ALIAS" \
    -keyalg RSA -keysize 2048 -validity 10000 \
    -storepass "$KEYSTORE_PASS" -keypass "$KEY_PASS" \
    -dname "CN=GrowCustom, OU=Dev, O=You, L=City, S=State, C=ID"
fi

echo "[*] zipalign (jika tersedia)..."
if command -v "$ZIPALIGN" >/dev/null 2>&1; then
  $ZIPALIGN -v -p 4 "$UNSIGNED_APK" "$ALIGNED_APK"
else
  echo "[!] zipalign tidak ditemukan, melewati langkah zipalign."
  mv "$UNSIGNED_APK" "$ALIGNED_APK"
fi

echo "[*] sign APK..."
if command -v "$APKSIGNER" >/dev/null 2>&1; then
  $APKSIGNER sign --ks "$KEYSTORE" --ks-key-alias "$KEY_ALIAS" --ks-pass "pass:$KEYSTORE_PASS" --key-pass "pass:$KEY_PASS" --out "$SIGNED_APK" "$ALIGNED_APK"
else
  echo "[!] apksigner tidak ditemukan, mencoba jarsigner..."
  jarsigner -keystore "$KEYSTORE" -storepass "$KEYSTORE_PASS" -keypass "$KEY_PASS" "$ALIGNED_APK" "$KEY_ALIAS"
  mv "$ALIGNED_APK" "$SIGNED_APK"
fi

echo "[*] selesai. APK signed berada di: $SIGNED_APK"

echo "Opsional: install ke device via adb:"
echo "  adb install -r $SIGNED_APK"

exit 0
