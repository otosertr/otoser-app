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
```
### 3.2. Araç ve Sahiplik
```python
class Vehicle(models.Model):
    id: UUID
    vin_hash: CharField(unique=True)       # HMAC-SHA256
    vin_ciphertext: TextField()            # encrypted VIN
    make: CharField()                      # Marka
    model: CharField()
    year: IntegerField(null=True, blank=True)
    engine: CharField(null=True, blank=True)
    created_at: DateTimeField(auto_now_add=True)

class VehicleOwnership(models.Model):
    id: UUID
    vehicle: ForeignKey(Vehicle)
    user: ForeignKey(User)
    start_at: DateTimeField(auto_now_add=True)
    end_at: DateTimeField(null=True, blank=True)  # null = aktif sahip
```
### 3.3. Usta Profili
```python
class MechanicProfile(models.Model):
    user: OneToOneField(User, primary_key=True)
    display_name: CharField()
    shop_name: CharField()
    specialties: JSONField()               # ['mechanic', 'electric', 'body', 'tyres', ...]
    verified_level: IntegerField(default=0)  # 0=doğrulanmamış, 1=kimlik, 2=saha vb.
    shop_lat: DecimalField()
    shop_lng: DecimalField()
    on_site_service: BooleanField(default=False)
    towing_available: BooleanField(default=False)
    rating_avg: FloatField(default=0)
    rating_count: IntegerField(default=0)
    created_at: DateTimeField(auto_now_add=True)
```
### 3.4. Arıza Kaydı ve Medya
```python
class RepairCase(models.Model):
    id: UUID
    vehicle: ForeignKey(Vehicle)
    opened_by: ForeignKey(User, related_name='opened_cases')
    title: CharField()
    description: TextField()
    location_lat: DecimalField()
    location_lng: DecimalField()
    location_area_text: CharField()        # örn. 'Kartal, İstanbul'
    service_mode: CharField(choices=['bring_to_shop', 'on_site', 'towing'])
    status: CharField(choices=['open', 'in_offers', 'accepted', 'closed', 'cancelled'])
    created_at: DateTimeField(auto_now_add=True)
    updated_at: DateTimeField(auto_now=True)

class RepairMedia(models.Model):
    id: UUID
    repair_case: ForeignKey(RepairCase, related_name='media')
    media_type: CharField(choices=['photo', 'video'])
    url: TextField()
    created_at: DateTimeField(auto_now_add=True)
```
### 3.5. Teklifler
```python
class Offer(models.Model):
    id: UUID
    repair_case: ForeignKey(RepairCase, related_name='offers')
    mechanic: ForeignKey(User, related_name='offers_made')  # mechanic user
    min_price: DecimalField()
    max_price: DecimalField()
    currency: CharField(default='TRY')
    eta_days: IntegerField()               # tahmini süre (gün)
    notes: TextField()
    status: CharField(choices=['sent', 'withdrawn', 'accepted', 'rejected'])
    created_at: DateTimeField(auto_now_add=True)
    updated_at: DateTimeField(auto_now=True)
```

### 3.6. Mesajlaşma
```python
class Thread(models.Model):
    id: UUID
    repair_case: ForeignKey(RepairCase, related_name='threads')
    customer: ForeignKey(User, related_name='threads_as_customer')
    mechanic: ForeignKey(User, related_name='threads_as_mechanic')
    created_at: DateTimeField(auto_now_add=True)

class Message(models.Model):
    id: UUID
    thread: ForeignKey(Thread, related_name='messages')
    sender: ForeignKey(User)
    message_text: TextField()
    created_at: DateTimeField(auto_now_add=True)
    read_at: DateTimeField(null=True, blank=True)
```

### 3.7. Yorumlar
```python
class Review(models.Model):
    id: UUID
    repair_case: OneToOneField(RepairCase, related_name='review')
    mechanic: ForeignKey(User, related_name='reviews_received')
    customer: ForeignKey(User, related_name='reviews_given')
    stars: IntegerField()                  # 1-5
    tags: JSONField()                      # örn. ['şeffaf fiyat', 'zamanında teslim']
    comment: TextField()
    created_at: DateTimeField(auto_now_add=True)
```

### 3.8. Abonelik ve Kredi (Basit MVP Seviyesi)
```python
class SubscriptionPlan(models.Model):
    id: UUID
    name: CharField()
    monthly_price: DecimalField()
    monthly_offer_quota: IntegerField()    # ayda kaç teklif hakkı
    is_active: BooleanField(default=True)

class Subscription(models.Model):
    id: UUID
    mechanic: ForeignKey(User, related_name='subscriptions')
    plan: ForeignKey(SubscriptionPlan)
    start_at: DateTimeField()
    end_at: DateTimeField()

class CreditTransaction(models.Model):
    id: UUID
    mechanic: ForeignKey(User, related_name='credit_transactions')
    change: IntegerField()                 # + veya - teklif hakkı
    reason: CharField()
    created_at: DateTimeField(auto_now_add=True)
```
### 4. API Tasarımı (Endpoint'ler ve Örnek Gövdeler)
##### Base URL: /api/v1/

#### 4.1. Auth (OTP)
POST /auth/otp/request
Body:
```json
{ "phone": "+905xxxxxxxxx" }
```
Response: 200 OK (her zaman generic cevap – güvenlik için)

POST /auth/otp/verify
Body:
```json
{ "access_token": "jwt_or_token", "user": { "id": "...", "role": "customer" } }
```
#### 4.2. Araçlar
POST /vehicles
Body:
```json
{
  "vin": "WVWZZZ...1234",
  "make": "Volkswagen",
  "model": "Golf",
  "year": 2015,
  "engine": "1.6 TDI"
}
```
İş mantığı:

VIN doğrula (format)
vin_hash ile Vehicle ara:
varsa: o kaydı dön (ve yeni ownership oluştur)
yoksa: yeni Vehicle oluştur + ownership

Response:
```json
{
  "id": "vehicle_uuid",
  "masked_vin": "***************1234",
  "make": "Volkswagen",
  "model": "Golf",
  "year": 2015
}
```
GET /vehicles
Kullanıcının sahip olduğu aktif araçları listeler.

#### 4.3. Arıza Kayıtları
POST /repair-cases
Body:
```json
{
  "vehicle_id": "vehicle_uuid",
  "title": "Motor sesi var",
  "description": "Soğukken çalıştırınca metalik ses geliyor...",
  "location_lat": 40.9,
  "location_lng": 29.1,
  "location_area_text": "Kartal, İstanbul",
  "service_mode": "bring_to_shop"
}
```
Response:
```json
{
  "id": "case_uuid",
  "status": "open",
  ...
}
```
GET /repair-cases/:id
Arıza detayları + medya + (kullanıcı ise) gelen teklifler.

GET /repair-cases?mine=true
Kullanıcının açtığı arızaları listeler.

#### 4.4. Medya Yükleme
Tercihen pre-signed URL:

POST /repair-cases/:id/media-upload-url
Body:
```json
{ "media_type": "photo", "file_extension": "jpg" }
```
Response:
```json
{ "upload_url": "https://...", "final_url": "https://..." }
```
POST /repair-cases/:id/media
Body:
```json
{ "media_type": "photo", "url": "https://..." }
```
#### 4.5. Usta Profil
GET /mechanic/me
PUT /mechanic/me
Body:
```json
{
  "display_name": "Ahmet Usta",
  "shop_name": "Ahmet Oto",
  "specialties": ["mechanic", "electric"],
  "shop_lat": 40.9,
  "shop_lng": 29.1,
  "on_site_service": true,
  "towing_available": false
}
```
#### 4.6. Ustanın Arıza Havuzu
GET /mechanic/repair-cases?lat=40.9&lng=29.1&page=1
Sunucu:

Aynı şehirdeki arızaları getirir.
Mesafeye göre sıralar (yakından uzağa).
İleriki fazda km filtresi eklenebilir.
Response (özet):
```json
[
  {
    "id": "case_uuid",
    "title": "Motor sesi var",
    "distance_km": 3.2,
    "service_mode": "bring_to_shop",
    "created_at": "..."
  },
  ...
]
```
#### 4.7. Teklifler
POST /repair-cases/:id/offers (usta)
Body:
```json
{
  "min_price": 1500,
  "max_price": 2500,
  "currency": "TRY",
  "eta_days": 2,
  "notes": "Muhtemelen triger gergisi vs. Aracı görmem lazım."
}
```
İş mantığı:

Ustanın teklif hakkı var mı? (abonelik / kredi kontrolü)
Aynı usta aynı case'e daha önce teklif verdiyse engelle.
Response: 201 Created

GET /repair-cases/:id/offers (kullanıcı)
Teklifler listesi:
```json
[
  {
    "id": "offer_uuid",
    "mechanic": {
      "id": "user_uuid",
      "display_name": "Ahmet Usta",
      "shop_name": "Ahmet Oto",
      "rating_avg": 4.8,
      "rating_count": 23,
      "distance_km": 3.2
    },
    "min_price": 1500,
    "max_price": 2500,
    "eta_days": 2,
    "notes": "...",
    "status": "sent"
  }
]
```
POST /offers/:id/accept (kullanıcı)
RepairCase.status → accepted
İlgili Offer.status → accepted, diğerleri → rejected
Gerekirse otomatik Thread açılır.

#### 4.8. Mesajlaşma (Inbox)
POST /threads
Veya teklif kabul edilirken otomatik de oluşturulabilir.

Body:
```json
{
  "repair_case_id": "case_uuid",
  "mechanic_id": "user_uuid"
}
```
GET /threads
Kullanıcı veya usta için kendi thread'lerini listeler.

GET /threads/:id/messages
POST /threads/:id/messages
Body:
```json
{ "message_text": "Merhaba, yarın sabah gelebilir miyim?" }
```
#### 4.9. Yorumlar
POST /repair-cases/:id/review
Sadece closed ve iş tamamlandı olan case'ler için.

Body:
```json
{
  "stars": 5,
  "tags": ["şeffaf fiyat", "zamanında teslim"],
  "comment": "İlgili ve açıklayıcıydı, teşekkürler."
}
```
### 5. Validasyon ve Edge-Case Kuralları
##### Araç Ekleme
VIN: 17 karakter + temel format kontrolü
Aynı kullanıcı aynı aracı tekrar eklemek isterse, eski sahiplik end_at doluysa yeni ownership başlatılır.
##### Arıza Açma
Aynı gün içinde aynı kullanıcı + aynı araç için birden fazla case açılırsa uyarı.
Yeni kullanıcılar için günlük 1 case limiti (yapılandırılabilir).
##### Teklif Verme
min_price <= max_price zorunlu.
Usta aynı case'e bir kez teklif verebilir.
Ustanın aktif aboneliği yoksa ve ücretsiz hakkını kullandıysa: hata döner (veya "ödeme gerekli" kodu).
Case Durum Geçişleri
open → in_offers (ilk teklif geldiğinde)
in_offers → accepted (teklif seçildiğinde)
accepted → closed (iş tamamlandı + kullanıcı onayı)
* → cancelled (kullanıcı iptal ederse; belirli kurallarla)
  
6. Geliştirme Yol Haritası (İskelet)
Bu kısım GitHub issue'larına sprint task'ları olarak da bölünebilir.

Sprint 1 (Altyapı + Temel Modeller)
 Django + DRF kurulum
 PostgreSQL bağlantısı
 User modeli (role alanıyla)
 OTP auth entegrasyonu (stub bile olabilir)
 Vehicle + VehicleOwnership modelleri
 VIN hash/encrypt yardımcı fonksiyonları
Sprint 2 (Arıza ve Usta Temeli)
 MechanicProfile modeli + basit CRUD API
 RepairCase + RepairMedia modelleri ve API'leri
 Arıza açma akışı (+ konum zorunlu)
 Medya upload iskeleti (şimdilik basit bir dosya servisi bile olabilir)
 Ustanın arıza havuzu endpoint'i (mesafeye göre sıralama)
Sprint 3 (Teklif + Inbox + Yorum)
 Offer modeli + teklif API'si
 Teklif kabul akışı + case status geçişleri
 Thread + Message modelleri + inbox API'leri
 Review modeli + yorum API'si
 Admin panelde:
usta doğrulama
şikayet/yorum yönetimi için alanlar
Sprint 4 (Abonelik/Kredi Basit MVP)
 SubscriptionPlan, Subscription modelleri
 CreditTransaction modeli
 Teklif verirken hak kontrolü
 Admin'den ustaya manuel hak tanımlama (pilot aşama için yeterli)
