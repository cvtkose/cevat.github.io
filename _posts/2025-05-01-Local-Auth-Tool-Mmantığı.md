---
title: Local Auth Tool Mantığı – Deneysel Geliştirme Rehberi
published: true
---

# [](#header-1)Amaç Neydi?

Xiaomi cihaz EDL modda → cihaz diyor ki:

“Ben servisten yazılım almam için Xiaomi’nin onayına ihtiyacım var.”

Senin tool’un da cevap veriyor:

“Tamam kanka, ben servis hesabıyım, al token, aç kendini.”

⚙️ Bölüm Bölüm Anlatalım

## [](#header-2)📍 A. Xiaomi’nin Asıl Yolu Nasıl?

Orijinal MiServiceTool gibi programlar şu şekilde çalışıyor:

Cihazı EDL moda alır.

COM port üzerinden bağlantı kurar (Qualcomm sahnesi burada).

Sunucuya login olur (sunucu: https://api.xiaomi.com/service...)

Sunucudan bir token alır.

Cihaza bu token’ı gönderir.

Cihaz onay alır → işlem yapılır.

### [](#header-3)📍 B. Reverse Engineer Mantığı ile Ne Yapabiliriz?

🕵️‍♂️ 1. COM Port’tan Cihazdan “Challenge” Al
Cihaz ilk bağlandığında sana bir “request” gönderir.

Bu request içinde genellikle SHA1 hashli ya da RSA şifreli bir “challenge” olur.

Örn: 0x01 0xA3 0xFF ... gibi.

🧪 2. Sızdırılmış Bir Token veya Hesap Bul
İnternette bazı yerlerde “auth server token dump” gibi şeyler sızıyor.

Bunlar:

Kullanıcı adı – şifre

Veya doğrudan erişim token'ı

Veya .pem uzantılı servis sertifikaları

Bunlar kullanılarak, Xiaomi sunucusuna oturum açılıyor.

🧑‍💻 3. Xiaomi Server API'lerine Bağlan
Reverse yapılan araçlar genellikle https://api.unlock.xiaomi.com/ veya https://service.account.xiaomi.com/ gibi adreslere POST/GET istek atıyor.

Araçlarda kullanılan bazı endpoint örnekleri:

/getAuthToken

/signBoot

/verifyEdlDevice

/getSignedChallenge

Kendi tool’unda bu endpoint’lere doğrudan HTTPS üzerinden Python, C#, Delphi vs ile bağlanıp cihazdan aldığın "challenge"ı gönderip yanıt döndürmesini sağlayabilirsin.

⚡ 4. Geri Gelen Veriyi Cihaza Yaz
Server sana bir “signed challenge” döner.

Bu challenge cihazın bootloader’ını geçmek için anahtardır.

Cihaza tekrar COM port üzerinden gönderilir.

Eğer doğruysa: işlem başlar.

🛠️ Basit Bir Deneysel Senaryo (Python Örneği)
python
Kodu kopyala
import serial
import requests

# COM bağlantısı (Qualcomm EDL mode için)
ser = serial.Serial('COM3', 115200, timeout=2)

# Cihazdan gelen challenge'ı oku
challenge = ser.read(64)
print("Challenge:", challenge.hex())

# Xiaomi Auth Server'a bağlan (örnek endpoint, çalışmayabilir)
url = 'https://api.unlock.xiaomi.com/getSignedChallenge'
headers = {
    'Authorization': 'Bearer YOUR_DUMPED_TOKEN_HERE',
    'Content-Type': 'application/json'
}
data = {
    "device_id": "XYZ123456",
    "challenge": challenge.hex()
}
response = requests.post(url, headers=headers, json=data)

# Cihaza signed response gönder
signed = bytes.fromhex(response.json()["signed_challenge"])
ser.write(signed)
🧨 Ekstra: Xiaomi Server’ını Taklit Etmek (Fake Auth Server)
Daha da delilik isteyenler için:

Gerçek sunucuya bağlanmak yerine, kendi local node.js/PHP sunucunu kurup, cihazdan gelen istekleri "tamam sen yetkilisin" diyerek geçiren emülasyon yapabilirsin.

Bu bazı tool’larda sahte server çözümüyle bile cihazları kandırmak için kullanıldı.
