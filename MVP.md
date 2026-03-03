# HairGrade AI — MVP Specification

React Native (Expo) ile geliştirilecek. Foundation model tabanlı saç dökülmesi analizi.

## Temel Akış

```
Onboarding → Fotoğraf Çek (2 açı) → AI Analiz → Sonuç Ekranı → Tedavi Planı → İlerleme Takibi
```

---

## 1. Onboarding (3 ekran swipe)

1. "AI ile saç dökülmesi seviyeni öğren" + hero görsel
2. "Kişisel tedavi planı al" + before/after mockup
3. Cinsiyet seç (Erkek → Norwood, Kadın → Ludwig) + Yaş aralığı + Ne zamandır dökülüyor (opsiyonel)

Onboarding sonrası doğrudan kamera ekranına yönlendir. Kayıt zorunlu değil (anonymous-first).

## 2. Fotoğraf Çekimi

### Gerekli Açılar
- **Ön yüz** (alın + saç çizgisi görünecek şekilde)
- **Tepe** (crown/vertex, yukarıdan çekilmiş)

### Guided Capture UI
- Ekranda yüz/baş outline overlay göster (alignment guide)
- Kalite kontrolleri (client-side):
  - Bulanıklık kontrolü (Laplacian variance < 100 → "Daha net çek")
  - Aydınlatma kontrolü (ortalama brightness < 40 → "Daha aydınlık ortam")
  - Yüz/baş tespiti yoksa → "Başını kareye hizala"
- Her açı için ayrı çekim ekranı, aralarında "Şimdi tepeden çek" yönlendirmesi

### Teknik
- `expo-camera` ile çekim
- JPEG, max 1024x1024 resize (upload öncesi client-side compress)
- Fotoğraflar analiz sonrası sunucuda 30 gün tutulur, sonra silinir (GDPR)

## 3. AI Analiz

### Mimari
```
Client (React Native)
  → POST /api/analyze { photos: [base64], gender, age?, duration? }
  → Backend (Node.js/Fastify)
    → OpenRouter API (paralel):
        1. Gemini 2.0 Flash ($0.0003/analiz)
        2. GPT-4o ($0.007/analiz)
    → Consensus logic
    → JSON response
  → Client sonuç ekranı
```

### Consensus Logic
- 2 model aynı stage → kesin sonuç, yüksek confidence
- 1 stage fark (ör. biri 3, biri 4) → GPT-4o'nun sonucunu al, orta confidence
- 2+ stage fark → "Sonuç belirsiz, daha net fotoğraf çek veya dermatolog önerisi" göster

### System Prompt (Norwood)
```
You are an experienced trichologist. Analyze these hair loss photos using the
Norwood-Hamilton scale. You will receive a frontal photo and a crown/vertex photo.

Patient info: Gender: {gender}, Age: {age}, Duration: {duration}

Norwood-Hamilton Scale:
- Stage 1: No hair loss, full juvenile hairline
- Stage 2: Slight temporal recession (mature hairline). NOT clinically significant
- Stage 3: Deep symmetrical recession at temples. First clinically significant stage
- Stage 3V: Stage 2 hairline + significant vertex (crown) thinning
- Stage 4: Severe frontotemporal recession + vertex thinning, connected by hair band
- Stage 5: Front and vertex areas enlarging, connecting band narrowing
- Stage 6: Bridge between front and vertex lost. Horseshoe pattern
- Stage 7: Most severe. Only narrow band of hair on sides and back

Respond ONLY in this JSON format:
{
  "stage": "3V",
  "confidence": 82,
  "observations": [
    "Temporal recession visible bilaterally, approximately 2cm above original hairline",
    "Crown area shows moderate thinning with visible scalp"
  ],
  "severity": "moderate",
  "hair_density_estimate": 65
}

Rules:
- confidence: 0-100 integer
- severity: "minimal" | "mild" | "moderate" | "severe" | "extensive"
- hair_density_estimate: 0-100 (100 = full density, 0 = completely bald)
- Be conservative — when uncertain between two stages, choose the lower one
- This is for educational/wellness purposes only, not medical diagnosis
```

Ludwig prompt aynı yapıda, scale tanımları:
- **Grade I**: Minimal diffuse thinning on crown, wider part line
- **Grade II**: Moderate thinning, scalp visible through hair on crown
- **Grade III**: Severe diffuse thinning, extensive scalp visibility

### Loading UX
- Analiz süresi: 3-8 saniye
- "Fotoğrafın analiz ediliyor..." animasyonu (pulsing brain/scan icon)
- Progress steps göster: "Saç çizgisi inceleniyor" → "Yoğunluk hesaplanıyor" → "Sonuç hazırlanıyor"

## 4. Sonuç Ekranı

### Ana Bileşenler
1. **Stage circle** — büyük, ortada (ör. "Stage 3V")
2. **Confidence bar** — %0-100, renk kodlu (<%60 kırmızı, %60-80 sarı, %80+ yeşil)
3. **Severity badge** — "Moderate" gibi tek kelime, renk kodlu
4. **Density estimate** — %65 gibi, bar ile
5. **Observations** — AI'ın gözlemleri, bullet list (Türkçe/İngilizce locale'e göre)
6. **CTA**: "Tedavi Planını Gör" butonu

### Disclaimer (her sonuç ekranının altında)
> "Bu bir tıbbi teşhis değildir. Eğitim ve takip amaçlıdır. Tedaviye başlamadan önce bir dermatoloğa danışın."

## 5. Tedavi Planı

Stage'e göre otomatik oluşturulan, statik içerik (AI üretmiyor, önceden hazırlanmış).

### Stage → Tedavi Mapping

#### Norwood 1-2 (Erken — Koruma)
- **Birincil**: Finasteride 1mg/gün (reçeteli, doktora danış)
- **Destekleyici**: Minoxidil %5 (OTC), günde 2x
- **Supplement**: Saw Palmetto 320mg, D3 vitamini (eksikse)
- **Yaşam tarzı**: Stres yönetimi, beslenme
- **Beklenti**: %80-90 dökülme durdurma, %60 yeni çıkma (6-12 ay)

#### Norwood 3-4 (Orta — Kombinasyon)
- **Birincil**: Finasteride + Minoxidil kombine (%94.1 etkinlik)
- **İkincil**: PRP değerlendirmesi, LLLT cihazı
- **Cerrahi**: Saç ekimi değerlendirmesi (2500-3500 greft)
- **Supplement**: Saw Palmetto, Zinc, Iron (eksikse)
- **Beklenti**: İlaçla 3-6 ayda yanıt değerlendir, cerrahi opsiyonu

#### Norwood 5-7 (İleri — Cerrahi Öncelikli)
- **Birincil**: Saç ekimi (FUE/FUT)
- **Destek**: Post-op Finasteride + Minoxidil (ömür boyu)
- **Alternatif**: Scalp mikropigmentasyon, saç protezi
- **Beklenti**: Cerrahi + medikal bakım, Norwood 7'de donör alan sınırlı

#### Ludwig I-III (Kadın)
- **Grade I**: Minoxidil %2, Iron/D3/Zinc takviyesi, beslenme
- **Grade II**: Minoxidil %5, Spironolactone (reçeteli), PRP
- **Grade III**: Yoğun medikal tedavi + saç ekimi değerlendirmesi

### Tedavi Kartı UI
Her tedavi bir kart:
- İsim + kısa açıklama
- Etkinlik (%) + kanıt seviyesi (Güçlü/Orta/Zayıf)
- Tahmini süre (ilk etki → tam etki)
- Aylık maliyet aralığı
- Yan etkiler (collapsible)
- "Daha Fazla Bilgi" → detay sayfası
- Affiliate link (uygunsa) → "Satın Al" butonu

### Supplement Kanıt Seviyeleri (sadece bunları öner)
| Supplement | Kanıt | Günlük Doz |
|-----------|-------|------------|
| Saw Palmetto | Güçlü (5 RCT) | 320mg |
| Iron | Güçlü (eksikse) | Doktora danış |
| Vitamin D | Güçlü (eksikse) | 2000-4000 IU |
| Zinc | Orta (eksikse) | 15-30mg |
| Rosemary Oil | Orta (1 RCT) | Topikal |
| Biotin | Zayıf (eksiklik yoksa etkisiz) | Önerme |

## 6. İlerleme Takibi (Progress Tracker)

### MVP Scope
- Takvim view — her ay fotoğraf çekme hatırlatması (push notification)
- Fotoğraf karşılaştırma — yan yana slider (before/after)
- Manuel log:
  - Hangi tedavileri uyguluyor (checkbox list)
  - Compliance tracking (bugün minoxidil sürdün mü? evet/hayır)
- Stage geçmişi grafiği (zaman → stage değişimi)

### Veri Modeli
```
UserProfile {
  id, gender, age_range, hair_loss_duration,
  created_at, locale
}

Analysis {
  id, user_id, photos[], stage, confidence,
  severity, density, observations[], model_responses[],
  created_at
}

TreatmentLog {
  id, user_id, treatment_name, started_at,
  is_active, notes
}

ComplianceEntry {
  id, user_id, treatment_log_id, date,
  completed: boolean
}
```

## 7. Hesap & Auth

- **MVP**: Anonymous kullanım (device ID ile). Kayıt opsiyonel.
- Kayıt: Email + password (Supabase Auth)
- Kayıt teşviki: "Sonuçlarını kaybet me — ücretsiz hesap oluştur"
- Kayıt olmadan: son 1 analiz lokalde tutulur (AsyncStorage)
- Kayıt olunca: tüm geçmiş sunucuda

## 8. Tech Stack

| Katman | Teknoloji |
|--------|-----------|
| Framework | React Native (Expo SDK 52+) |
| Navigation | Expo Router (file-based) |
| State | Zustand |
| Backend | Supabase (Auth + PostgreSQL + Storage) |
| AI API | OpenRouter (Gemini 2.0 Flash + GPT-4o) |
| Kamera | expo-camera |
| Bildirimler | expo-notifications |
| Analytics | Umami (web), expo-analytics (app) |
| Styling | NativeWind (Tailwind for RN) |

## 9. API Endpoints

```
POST /api/analyze
  Body: { photos: string[], gender, age?, duration? }
  Response: { stage, confidence, severity, density, observations[] }
  Auth: optional (anonymous OK, rate limit: 2/ay free)

GET /api/analyses
  Response: Analysis[]
  Auth: required

POST /api/treatments/log
  Body: { treatment_name, started_at }
  Auth: required

POST /api/compliance
  Body: { treatment_log_id, date, completed }
  Auth: required
```

Backend Supabase Edge Functions veya ayrı Node.js/Fastify servis olabilir. MVP'de Supabase Edge Functions yeterli.

## 10. Monetizasyon (MVP)

| Tier | Fiyat | Özellikler |
|------|-------|------------|
| **Free** | $0 | 2 analiz/ay, temel sonuç, tedavi kartları, manuel takip |
| **Premium** | $9.99/ay | Sınırsız analiz, detaylı rapor, compliance tracker, before/after slider |

Affiliate linkler tedavi kartlarında (Saw Palmetto, Minoxidil vb. Amazon/iHerb linkleri). RevenueCat ile in-app purchase yönetimi.

## 11. Ekranlar (MVP)

```
app/
├── (onboarding)/
│   └── index.tsx          # 3-step swipe onboarding
├── (tabs)/
│   ├── index.tsx          # Home — son analiz özeti + "Yeni Analiz" CTA
│   ├── progress.tsx       # İlerleme takibi — takvim + before/after
│   └── profile.tsx        # Hesap, ayarlar, premium
├── analyze/
│   ├── capture.tsx        # Guided kamera (2 açı)
│   ├── loading.tsx        # AI analiz animasyonu
│   └── result.tsx         # Sonuç ekranı + tedavi planı CTA
├── treatment/
│   ├── plan.tsx           # Stage bazlı tedavi kartları
│   └── [id].tsx           # Tedavi detay sayfası
└── auth/
    ├── login.tsx
    └── register.tsx
```

## 12. MVP Dışı (Phase 2+)

- AI before/after karşılaştırma (density heatmap)
- Telemedicine entegrasyonu
- Klinik referral pipeline (Türkiye saç ekimi)
- Topluluk/forum
- Savin/BASP scale desteği
- Custom CNN model (300K+ image ile eğitim)
- Push notification ile tedavi hatırlatma
- Çoklu dil desteği (TR/EN/DE/AR)

## 13. Disclaimer & Legal

Her sonuç ekranı, tedavi sayfası ve app açılışında:

> "HairGrade is an educational wellness tool, not a medical device. Results are AI-generated assessments and do not constitute medical diagnosis or treatment advice. Always consult a qualified dermatologist before starting any treatment. Individual results may vary."

Terms of Service ve Privacy Policy (web sayfası linki) zorunlu.
