# Gap – Power BI Uçtan Uca Eğitim Veri Seti

> **KURGUSAL VERİDİR.** Proje adları, müşteriler, tedarikçiler, personel adları, vergi numaraları ve döviz kurları rastgele üretilmiştir. Gerçek Gap İnşaat kayıtlarıyla ilgisi yoktur. Kurlar **TCMB kuru değildir**.

Dönem: **01.01.2023 – 30.09.2026** · Raporlama para birimi: **USD** · 14 proje, 6 bölge, ~98 bin maliyet satırı, ~100 bin ekipman günü

## Klasör yapısı

| Klasör / Dosya | Tür | Satır (yaklaşık) | Eğitimde kullanım |
|---|---|---|---|
| **01_Ana_Veriler** | | | **Boyut tabloları** |
| Projeler.xlsx | Boyut | 14 | Proje, ülke, bölge, sözleşme tipi/bedeli, proje müdürü |
| Maliyet_Kodlari.xlsx | Boyut (parent-child) | 38 | 3 seviyeli maliyet kodu ağacı → PATH / PATHITEM |
| Tedarikciler.csv | Boyut (kirli) | 39 | Mükerrer tedarikçi temizliği, eşleme tablosu |
| Malzemeler.xlsx | Boyut | 35 | Malzeme grubu, birim, standart fiyat |
| Personel.xlsx | Boyut (hiyerarşi) | 664 | YoneticiID ile organizasyon hiyerarşisi |
| Ekipmanlar.xlsx | Boyut | 91 | Ekipman tipi, mülkiyet, saatlik maliyet/yakıt |
| **02_Hareketler** | | | **Olgu (fact) tabloları** |
| Maliyetler\ (46 CSV) | Olgu, klasör | 98.290 | Klasörden birleştirme, şema değişikliği, para birimi çevrimi |
| Butce_2023_2026.xlsx | Olgu, geniş format | 2 sayfa | Unpivot, başlık satırı atlama, versiyon karşılaştırma |
| Hakedisler.xlsx | Olgu | 348 | Gelir, kesintiler, tahsilat / DSO / yaşlandırma |
| Satinalma_Siparisleri.xlsx | Olgu | 7.420 | Tedarik süresi, zamanında teslim (OTD), SoD kontrolü |
| Puantaj.xlsx | Olgu, proje başına sayfa, geniş format | 14 sayfa | Sayfaları birleştirme, 31 gün kolonunu unpivot, metin/sayı karışık |
| Ekipman_Kullanim.csv | Olgu | 100.622 | Kullanım oranı, yakıt verimi, arıza |
| ISG.xlsx | Olgu (2 sayfa) | 394 olay + adam-saat | LTIFR, olay sıklığı |
| Proje_Ilerleme.xlsx | Olgu | 366 | S-eğrisi, kazanılmış değer (EVM: SPI, CPI) |
| **03_Referans** | | | |
| Doviz_Kurlari.json | Referans | ~2.000 gün × 2 | JSON ayrıştırma, hafta sonu boşluklarını doldurma |
| RLS_Kullanici_Bolge.xlsx | Güvenlik | 9 | Dinamik RLS (USERPRINCIPALNAME) |
| Bolge_Hedefleri.csv | Hedef | 24 | Hedef / gerçekleşen KPI kartları |
| **04_Uretici_Script** | | | Veriyi yeniden üretmek ya da büyütmek için Python betiği |

## Anahtar ilişkiler (önerilen yıldız şema)

```
Takvim[Tarih] (DAX ile kurulacak) 1─* Maliyet[BelgeTarihi], Hakedis[Donem sonu], PO[SiparisTarihi], Ekipman[Tarih], ISG[OlayTarihi]
Projeler[ProjeKodu]       1─* Maliyet, Butce, Hakedis, PO, Puantaj, Ekipman, ISG, Ilerleme
MaliyetKodlari[MaliyetKodu] 1─* Maliyet, Butce
Tedarikciler[TedarikciID] 1─* Maliyet, PO       (mükerrer ID'ler temizlenmeli!)
Malzemeler[MalzemeKodu]   1─* PO
Personel[PersonelID]      1─* Puantaj[Sicil No], PO[TalepEden/Onaylayan] (pasif ilişki → USERELATIONSHIP)
Ekipmanlar[EkipmanKodu]   1─* Ekipman_Kullanim
Kur tablosu  → ilişki kurmadan ölçü içinde LOOKUPVALUE / TREATAS
RLS[Bolge] → Projeler[Bolge]
```

## Bilerek bırakılan "kirli veri" tuzakları (Power Query alıştırmaları)

| # | Nerede | Sorun | Beklenen çözüm |
|---|---|---|---|
| 1 | Maliyetler klasörü | `Maliyet_2024_07 - Kopya.csv` ve `Okubeni.txt` var | Dosya adına / uzantısına göre filtre |
| 2 | Maliyetler 2025-01 sonrası | `Tutar`→`BelgeTutari`, `ParaBirimi`→`BelgeParaBirimi` adı değişti; `KaydedenKullanici` eklendi | Örnek dosya dönüşümünde kolon adlarını standartlaştırma |
| 3 | Maliyetler | 2024 ve öncesinde tarih `gg.aa.yyyy`, 2025'ten sonra `yyyy-aa-gg` | Yerel ayar (tr-TR) ile tür dönüştürme, koşullu ayrıştırma |
| 4 | Maliyetler, Ekipman | Ondalık ayırıcı virgül, binlik ayırıcı nokta, `;` ayraç | Yerel ayara göre tür değiştirme |
| 5 | Maliyetler | ~%1 küçük harf proje kodu, ~%0,6 sonda boşluk | Trim + Upper |
| 6 | Maliyetler | 60 satırda boş maliyet kodu | "Tanımsız" üyesi oluşturma, veri kalitesi raporu |
| 7 | Maliyetler | ~90 tam mükerrer satır | Remove Duplicates (BelgeNo bazında) |
| 8 | Maliyetler | ~%1,5 negatif tutar (iade) | İşaretin korunması; iade analizi |
| 9 | Tedarikciler.csv | Aynı firma farklı yazımla, farklı ID ile (T2101–T2104); boş ülke; " türkiye" | Bulanık birleştirme (fuzzy merge) → eşleme tablosu |
| 10 | Projeler.xlsx | "türkiye ", "IRAK", " Türkmenbaşı" | Trim, Capitalize, değer değiştirme |
| 11 | Butce_2023_2026.xlsx | İlk 3 satır rapor başlığı, aylar kolonda (`Oca.23`…), en altta "GENEL TOPLAM" satırı | Skip rows, Promote headers, Unpivot, Türkçe ay adından tarih üretme |
| 12 | Hakedisler.xlsx | `Proje` = "P-TR-01 - Ankara Lojistik Merkezi" | Ayraçla kolon bölme |
| 13 | Puantaj.xlsx | 14 proje sayfası + 1 açıklama sayfası; 1–31 gün kolonları; hücrede sayı ya da `H/İ/R/Mz` | Excel.Workbook ile sayfaları birleştirme, Unpivot, gün tarihini oluşturma, saat/izin ayrıştırma |
| 14 | Doviz_Kurlari.json | Sadece iş günleri var; uzun format | JSON → tablo, Pivot, takvimle birleştirip Fill Down |

## Kontrol ve denetim senaryoları (iç denetim gözüyle)

- **SoD ihlali:** `Satinalma_Siparisleri` içinde ~125 satırda `TalepEdenPersonelID = OnaylayanPersonelID`; birkaçında da onaylayan proje müdürü değil, satınalmacı. İstisna raporu kurulabilir.
- **Kayıt gecikmesi:** Maliyetlerde `KayitTarihi – BelgeTarihi` farkı 0–8 gün; ay sonuna sarkan belgeler dönem kesmesi (cut-off) tartışması için uygun.
- **Açık hakedişler:** `TahsilatTarihi` boş olanlar → alacak yaşlandırma.
- **Mükerrer tedarikçi kartları:** Ana veri yönetişimi tartışması.

## Önerilen ölçü listesi (DAX modülleri)

| Konu | Ölçüler |
|---|---|
| Temel | Fiili Maliyet USD, Bütçe USD, Sapma, Sapma %, Hakediş Geliri, Brüt Kâr, Kâr Marjı % |
| Para birimi | Belge tarihindeki kurla USD/TRY/EUR çevrimi (SUMX + LOOKUPVALUE), dönem sonu kuru ile çevrim, para birimi seçici (field parameter) |
| Zaman zekâsı | YTD, Geçen Yıl Aynı Dönem, YoY %, Kümülatif (S-eğrisi), Hareketli 3 ay, Bütçe kalan |
| EVM | PV = Plan% × Bütçe, EV = Gerçekleşen% × Bütçe, AC = Fiili; **SPI = EV/PV, CPI = EV/AC**, EAC = BAC/CPI, VAC |
| Satınalma | Ortalama tedarik süresi, OTD %, gecikme gün dağılımı, açık sipariş tutarı |
| İnsan kaynağı | Adam-saat, fazla mesai saati (>8), devamsızlık %, aktif personel sayısı (tarih aralığı mantığı) |
| Ekipman | Kullanım oranı = Çalışma saati / Mevcut saat, arıza oranı, litre/saat |
| İSG | LTIFR = Kayıp günlü kaza × 1.000.000 / Toplam adam-saat; ramak kala oranı |
| Finans | DSO, tahsilat süresi, teminat kesintisi bakiyesi, yaşlandırma kovaları (0-30/31-60/61-90/90+) |
| İleri düzey | Maliyet kodu hiyerarşisi (PATH), personel hiyerarşisi, RANKX ile en çok sapan projeler, what-if parametresi (kur/CPI senaryosu), calculation groups |

## 35 saat için modül eşlemesi (öneri)

| Modül | Saat | Ana dosyalar |
|---|---|---|
| 1. Power BI'a giriş, arayüz, ilk rapor | 3 | Projeler, Hakedisler |
| 2. Power Query temelleri (tür, yerel ayar, temizlik) | 4 | Projeler, Tedarikciler, Ekipman_Kullanim |
| 3. Power Query ileri (klasör, unpivot, sayfa birleştirme, JSON, fuzzy merge, parametre, M fonksiyonu) | 6 | Maliyetler klasörü, Butce, Puantaj, Doviz_Kurlari |
| 4. Veri modelleme (yıldız şema, takvim, ilişkiler, aktif/pasif ilişki, hiyerarşi) | 4 | Tümü |
| 5. DAX temelleri (hesaplanmış kolon / ölçü, CALCULATE, filtre bağlamı) | 5 | Maliyet, Butce, Hakedis |
| 6. DAX ileri (zaman zekâsı, kur çevrimi, EVM, RANKX, PATH, what-if) | 6 | Maliyet, Ilerleme, Kur, Maliyet_Kodlari |
| 7. Görselleştirme ve rapor tasarımı (drill-through, tooltip, bookmark, field parameter) | 4 | Tümü |
| 8. Yayınlama, RLS, workspace, zamanlanmış yenileme, gateway | 2 | RLS_Kullanici_Bolge |
| 9. Bitirme projesi: Proje Kontrol Kokpiti | 1 | Tümü |

## KVKK notu

Personel tablosunda yalnızca kurgusal ad soyad, unvan ve ücret **kademesi** var; T.C. kimlik no ya da gerçek ücret yok. Eğitimde gerçek şirket verisine geçilirse puantaj ve personel tablolarında veri minimizasyonu, RLS ile erişim kısıtı ve maskeleme ayrıca ele alınmalıdır.

## Notlar

- Bütçe dosyası 2023'ten itibaren olan dönemi içerir. 2023 öncesi başlayan projelerde (P-TR-01, P-TM-01, P-RU-01) `Projeler[OnaylıButceUSD]` toplamı, bütçe dosyasının toplamından büyüktür. Fark, 2023 öncesi dönemden gelir ve "devreden" tartışması için kullanılabilir.
- Revize-1 bütçesi, gerçekleşen ilerleme eğrisine göre yayılmıştır; Orijinal ile kıyaslanınca plan kayması görülür.
- KDV %20, teminat kesintisi %5 ve avans mahsubu %10 **örnek oranlardır**. Güncel mevzuat için GİB'e bakılmalıdır.
- Gerçek kurla çalışmak isteyenler için ek alıştırma: TCMB `today.xml` / EVDS'den Power Query ile kur çekmek.
