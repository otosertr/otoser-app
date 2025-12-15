# Sanayi Rehberi & Parça Dedektifi (Faz 1 – Teknik Tasarım)

## Kapsam (Faz 1 / MVP)

Bu doküman, Faz 1 için teknik tasarımı tanımlar.  
Faz 1'in ana bileşenleri:

- Araç Kimliği ve Geçmişi (VIN)
- Arıza Kaydı Açma (Kullanıcı)
- Teklif Verme (Usta)
- Konuma Göre Eşleştirme (yakından uzağa)
- Inbox (kullanıcı–usta mesajlaşma)
- Basit abonelik + teklif limiti modeli (usta için)

**Not:** Parça Dedektifi, pazaryeri, ekspertiz vb. modüller bu dokümanın kapsamı dışındadır (Faz 2+).

---

## 1. Mimari Genel Bakış

- **Backend:** Django + Django REST Framework
- **DB:** PostgreSQL
- **Deployment (öneri):**
  - Başlangıçta tek uygulama (monolith)
  - İlerleyen fazlarda mikroservislere bölünebilir (örn. scraping, analitik)
- **Kimlik Doğrulama:**
  - OTP (telefon tabanlı) – bir SMS sağlayıcı üzerinden
- **Mobil Uygulama:**
  - Flutter (öneri) – API'lerle haberleşecek
- **Admin Panel:**
  - Django Admin kullanılacak
  - Usta doğrulama, arıza/yorum moderasyonu, abonelik/limit yönetimi

---

## 2. Ana Kavramlar ve İş Kuralları

### 2.1. Araç Kimliği (Vehicle) ve VIN

- VIN/Şase zorunlu (Faz 1'de)
- VIN, araç kimliğinin ana anahtarıdır:
  - Aynı VIN tekrar girilirse, aynı `Vehicle` kaydı kullanılır.
  - Böylece araç el değiştirse bile, geçmiş kayıtlar araç üzerinde birleşir.

**Güvenlik ve KVKK yaklaşımı:**

- DB'de VIN düz metin tutulmaz:
  - `vin_ciphertext`: uygulama seviyesinde şifreli veri
  - `vin_hash`: HMAC-SHA256 ile tek yönlü hash (VIN tabanlı; araç eşleştirme için)
- UI'da VIN maskeli gösterilir (örn. `***************1234`).
- Eski kayıtlar yeni kullanıcıya anonim görünür:
  - Tarih, km, yapılan işlem bilgisi kalır.
  - Önceki sahibin kişisel verisi (ad, tel, adres) tutulmaz/gösterilmez.

### 2.2. Konum (Location)

- Kullanıcı arıza açarken konum pinlemek zorunda.
- Usta profilinde sabit konum (iş yeri).
- Konuma göre:
  - Ustanın arıza havuzunda arızalar: yakından uzağa sıralanır.
  - Kullanıcının teklif listesindeki ustalar: mesafe bilgisi gösterilir.

**Mahremiyet:**

- Ustaya tam adres gösterilmez; sadece:
  - İlçe / semt bilgisi (`location_area_text`)
  - Haritada yaklaşık pin (zoom limitiyle; ayrıntı kısıtlanabilir)

### 2.3. Arıza Kaydı ve Teklif Akışı

- Kullanıcı arıza kaydı açar (foto zorunlu, video opsiyonel, konum zorunlu).
- Uygun ustalar (branş + konum) arızayı havuzlarında görür.
- Ustalar fiyat aralığı ile teklif verebilir (`min_price`, `max_price`).
- Kullanıcı teklifler listesini görür, ustalarla inbox üzerinden konuşur.
- Kullanıcı bir teklifi kabul eder:
  - Diğer teklifler otomatik reddedilir.
  - Arıza kaydı durumu: `accepted`.
- İş bitince:
  - Usta "iş tamamlandı" der.
  - Kullanıcı onaylar ve yorum + derece bırakır.
  - Arıza kaydı `closed` olur.

### 2.4. Mesajlaşma (Inbox)

- Her kabul edilen teklif için bir `Thread` açılır (veya teklif sonrası kullanıcı isterse).
- **Thread:**
  - 1 müşteri, 1 usta, 1 arıza kaydı
- **Message:**
  - Gönderen kullanıcı (customer veya mechanic)
  - Metin (zorunlu)
  - Opsiyonel medya (ileriki fazlarda)

### 2.5. Spam Kontrol ve Limitler

- **Yeni kullanıcı:**
  - Günde 1 arıza kaydı limiti.
- **Usta:**
  - 1 ücretsiz deneme teklifi.
  - Sonrası: abonelik veya teklif kredisi gerektirir.
- **Usta tekliflerinde:**
  - Çok hızlı seri teklif (örn. 1 dk'da 3'ten fazla) rate-limit ile engellenebilir.

---

## 3. Veri Modeli (DB Şeması Taslağı)

Aşağıdaki yapı PostgreSQL üzerinde Django ORM ile tasarlanacak. Tipler illustrative.

### 3.1. User ve Roller

```python
class User(AbstractBaseUser):
    id: UUID
    phone: CharField(unique=True)
    role: CharField(choices=['customer', 'mechanic', 'admin'])
    is_active: BooleanField
    created_at: DateTimeField
