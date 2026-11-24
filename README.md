# Akıllı Gıda Etiketi Analizi

> **Durum: Devam ediyor.** Veri hazırlama ve sızıntısız veri bölme aşamaları tamamlandı. Model aşamaları sırada.

Ambalajın arkasındaki içindekiler listesi küçük puntoyla yazılır, E-kodlu katkı maddeleri anlaşılmaz, alerjen uyarıları gözden kaçabilir. Bu projede bir ürünün içindekiler metni ve ambalaj görsellerinden yola çıkarak:

1. **Nutri-Score** sınıfını (a–e) tahmin eden,
2. **alerjenleri ve katkı maddelerini** çıkaran,
3. sonucu **sade Türkçe** ile açıklayan,
4. tek komutla ayağa kalkan bir **servis** olarak çalışan

uçtan uca bir sistem kuruyorum. Projenin odağı Türkiye pazarındaki ürünler.

| | |
|---|---|
| **Veri** | [Open Food Facts](https://world.openfoodfacts.org/data) (~4,5 milyon ürün, ODbL lisansı) |
| **Kapsam** | Türkiye ürünleri + karşılaştırma ve eğitim için 28.300 Avrupa ürünü |
| **Birincil ölçüt** | Makro-F1 (Nutri-Score sınıfları dengesiz) |
| **Ortam** | Google Colab, DuckDB, pandas, scikit-learn, lingua |

## Depo içeriği

| Dosya | İçerik |
|---|---|
| `notebooks/01_veri_hazirlama.ipynb` | Ham dökümün Parquet'e çevrilmesi, Türkiye ürünlerinin bulunması, veri kümesinin oluşturulması ve otomatik kontrolleri |
| `notebooks/02_dil_ve_bolme.ipynb` | İçindekiler metinlerinde dil tespiti, marka bazlı ve rastgele eğitim/doğrulama/test bölmesi, sızıntı kontrolleri |
| `data/dil_dogrulama_etiketli.csv` | Dil tespitini elle kontrol etmek için her gruptan 20 olmak üzere etiketlenmiş 60 örnek |

## Şimdiye kadar yapılanlar

### 1. Büyük veriyi belleğe yüklemeden işlemek
Open Food Facts dökümü sıkıştırılmış hâliyle ~1,2 GB, açık hâliyle ~9 GB ve 211 sütun. Bu dosyayı pandas ile Colab belleğine yüklemek yerine **DuckDB** ile bir kez sütun bazlı **Parquet**'e çevirdim. Sonraki bütün sorgular dakikalar yerine saniyeler sürüyor. Sütun seçmeden önce 211 sütunun hepsi için doluluk oranını (tüm veri ve Türkiye ürünleri ayrı ayrı) tek taramada hesapladım.

### 2. Türkiye ürünlerini doğru bulmak
- Ülke etiketi olarak yalnızca `en:turkey` aranınca **601 ürünün kaçtığını** fark ettim. Ülkenin uluslararası adı 2022'de değiştiği için veritabanında `en:turkiye` diye ikinci bir etiket oluşmuş.
- Ülke etiketi olmayan Türk ürünlerini yakalamak için **GS1 barkod öneklerini (868/869)** kullandım. Bu ürünlerin çoğu Fransa, Almanya ve Irak'ta kayıtlı **ihracat ürünleri** çıktı. Etiketleri farklı dillerde olduğu için bunları ayrı bir grup (`TR_barkod`) olarak tuttum.

### 3. Etiket sızıntısını baştan önlemek
Nutri-Score, besin değerlerinden (`*_100g`) doğrudan hesaplanıyor. Bu sütunları model girdisi yapmak, modele cevabı vermek demek. Bu yüzden hangi sütunun girdi, hangisinin yalnızca analiz amaçlı olduğunu veri hazırlama aşamasında açıkça ayırdım.

### 4. Kendi hatalarımı yakalayan kontroller
- DuckDB'deki `USING SAMPLE`, `WHERE` filtresinden **önce** çalışıyor. İlk denemede 28.300 Avrupa ürünü yerine yalnızca 1.931 ürün geldi. Seçimi tohum değerli bir hash sıralamasıyla (`ORDER BY hash(code || '-42')`) yeniden yazdım. Sonuç rastgele ama her çalıştırmada aynı.
- Tekrarlanan barkodlar seçimden sonra silinince örnek sayısı hedefin altına düşüyordu. Tekilleştirmeyi seçimden önceye aldım.
- Bu hataların tekrar gözden kaçmaması için not defteri `assert` kontrolleriyle bitiyor: satır sayısı, barkod tekilliği, hedef değişkenin dolu olması.

### 5. Marka bazlı, sızıntısız veri bölme
Aynı markanın ürünleri birbirine çok benziyor ("Çikolatalı Gofret" / "Fındıklı Gofret"). Bir marka hem eğitim hem test kümesine düşerse model markayı ezberleyip gerçekte olduğundan iyi görünür. Bu yüzden:
- `StratifiedGroupKFold` ile **marka bazlı** %70 / %15 / %15 bölme yaptım: bir markanın bütün ürünleri tek bir kümede. Tabaka etiketi `grup + Nutri-Score`, yani hem Türkiye/Avrupa oranı hem sınıf dağılımı kümelerde korunuyor.
- Karşılaştırma için bir de **rastgele bölme** oluşturdum. İki bölmedeki skor farkı, marka sızıntısının etkisini ölçecek.
- Bölmeden sonra marka sızıntısı ve aynı ürün adının farklı kümelerde geçip geçmediği otomatik olarak kontrol ediliyor.
- **Bilinen sınırlama:** Aynı marka farklı yazımlarla geçebiliyor (`marks-spencer`, `m-s`). Bu yüzden marka sızıntısı tamamen sıfırlanamıyor.

### 6. Dil tespiti
İçindekiler metinleri Türkçe, Fransızca, Almanca ve Arapça karışık. Her metnin dilini ve güven skorunu `lingua` ile buldum. Dedektörü yalnızca veride beklenen dillerle sınırladım, güveni 0,5'in altındaki metinleri `belirsiz` saydım. Otomatik sonuca körü körüne güvenmemek için 60 örneği elle etiketledim (`data/dil_dogrulama_etiketli.csv`).

## Yol haritası

- [x] Veri hazırlama ve veri kümesi oluşturma
- [x] Dil tespiti ve marka bazlı bölme
- [ ] Keşifsel veri analizi (sınıf dağılımı, metin uzunlukları, görsel örnekleri)
- [ ] Nutri-Score sınıflandırma: klasik ML temel modelleri ve Türkçe dil modeli (BERTurk)
- [ ] Ambalaj görsellerinden OCR ile içindekiler metni çıkarma
- [ ] Alerjen ve katkı maddesi çıkarımı
- [ ] Büyük dil modeliyle sade Türkçe açıklama
- [ ] FastAPI servisi, Docker ve GitHub Actions

## Nasıl çalıştırılır?

Not defterleri Google Colab için hazırlandı. Dosyalar Google Drive'da `MyDrive/off` klasöründe tutuluyor. **01 → 02** sırasıyla çalıştırın. Uzun süren adımlar (indirme, dönüştürme) dosya zaten varsa atlanıyor. Kullanılan kütüphaneler `requirements.txt` dosyasında.

Open Food Facts verisi her gün güncellendiği için sonuçlar indirme tarihine göre biraz değişebilir.
