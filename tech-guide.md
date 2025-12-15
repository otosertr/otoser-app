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
