I. Kodun Çalışma Sistemi (Adım Adım Mekanizma)

Algoritma, borsadaki gürültüyü temizlemek ve en pürüzsüz yükselen 5 atı seçmek için 6 aşamalı bir filtreleme ve puanlama hattı (pipeline) işletir:

Adım 1: Likidite ve Geçmiş Veri Duvarı (The Gatekeeper)
Kod, BIST_TUM_SYMBOLS listesindeki ~500 hisseyi ThreadPoolExecutor kullanarak 16 paralel işçi ile hızlıca yfinance üzerinden indirir. Ardından şu filtreleri uygular:
Minimum Bar Kıstası (MIN_BARS = 210): Son 1 yılda halka arz olmuş, geçmiş verisi sığ olan şirketleri eler. 200 günlük ortalamanın sağlıklı hesaplanması için bu şarttır.
Likidite Filtresi (MIN_AVG_TL_VOLUME = 50_000_000): Son 20 günlük ortalama işlem hacmi 50 Milyon TL'nin altında olan sığ, spekülatif, tahtacının kolayca manipüle edebileceği yan tahtaları tamamen sistemin dışına iter.
Fiyat Filtresi (MIN_PRICE = 2.0): Kuruşluk hisseleri (penny stocks) eler.

Adım 2: Zorunlu Boğa Rejimi FiltresiAlgoritma akıntıya karşı kürek çekmeyi reddeder. 
Bir hissenin momentum listesine girebilmesi için temel şart:Fiyat>EMA50>EMA200
Bu dizilim yoksa, hisse ne kadar çok kazandırmış olursa olsun anında elenir. Bu filtre, düşen bıçakları (falling knives) veya geçici dipten dönüş (dead cat bounce) hareketlerini yakalamanızı engeller; yalnızca kurulu ve tescilli boğa trendindeki hisseleri içeri alır.

Adım 3: Risk-Ayarlı Momentum (RAM) Motoru
Kodun kalbi burasıdır. Klasik tarayıcılar "son 3 ayda en çok gideni" seçer. Bu kod ise Sharpe Oranı benzeri bir Risk-Adjusted Momentum (ram_blend) hesaplar:
Son 6 aylık (window=126) ve 3 aylık (window=63) dönemler için ham getiriyi hesaplar.
Bu getiriyi, ilgili dönemin standart sapmasına (volatilitesine) böler ve karekök 252 ile yıllıklandırır.Ağırlıklandırma: Toplam momentum skorunun %60'ını uzun vadeye (126G), %40'ını kısa vadeye (63G) vererek dengeli bir harman oluşturur.
Anlamı: Çok sert zıplayıp çok sert düşen oynak bir hisse yerine, az dalgalanarak istikrarlı yükselen hisse daha yüksek RAM skoru alır.

Adım 4: Trend Kalitesi ve Pürüzsüzlük (ADX & Rkare)
Sistem, trendin sadece yönüyle değil, "kalitesiyle" de ilgilenir:
ADX (14): Trendin gücünü ölçer. 25'in üzerindeki ADX, trendin güçlü olduğunu gösterir.
Doğrusal Regresyon (Rkare): Son 90 günün kapanış fiyatlarının logaritmasını alarak doğrusal bir regresyon çizgisi çizer. Hesaplanan Rkare (Belirlilik Katsayısı), fiyatın bu çizgiye ne kadar sadık kaldığını ölçer. Eğer fiyat zikzak çizmeden, dümdüz bir cetvel gibi yükseliyorsa Rkare 1.0'a yaklaşır.
Eğim Filtresi: reg_slope_ann <= 0 ise (yani pürüzsüz ama aşağı yönlü bir düşüş varsa) hisse elenir.

Adım 5: Cross-Sectional (Evren İçi) Puanlama
Kodun en profesyonel kısmı pct = lambda col: df[col].rank(pct=True) satırıdır. Algoritma sabit eşik değerleri kullanmaz. Filtreyi geçen tüm hisseleri kendi içinde yarıştırır ve her metriği 0 ile 1 arasında bir yüzdeliğe (percentile) çevirir.
Ana Puanlama Ağırlıkları: %40 RAM Gücü + %15 ADX Gücü + %15 Rkare Pürüzsüzlüğü + %10 Kısa Vade Getiri + %10 Hacim İvmesi + %10 EMA50 Eğimi.
Bu sayede farklı ölçekteki metrikler (0-100 arası ADX ile 1-5x arası hacim ivmesi) birbiri içinde kusursuzca standardize edilir.

Adım 6: Kauçuk Bant (Mean-Reversion) Cezaları
Trendi güçlü olan bir hisse aşırı şişmiş ve düzeltme (pullback) vaktine gelmiş olabilir. Kod bunu engellemek için üç farklı aşırılık cezası keser:
RSI Cezası: RSI > 85 ise skordan direkt 15 puan siler.
Kauçuk Bant Cezası (Stretch ATR): Fiyat, 21 günlük ortalamadan (EMA21) çok uzaklaşmışsa ve bu mesafe ATR'ın 3 katını aşmışsa, fiyatın ortalamaya geri döneceğini (mean-reversion) varsayar ve skordan 20 puana kadar acımasız kesinti yapar.
Bollinger İhlali: Fiyat Bollinger Üst Bandını %2'den fazla aşmışsa 10 puan siler.
Sonuç olarak tüm cezalardan arınan evrendeki en yüksek puanlı ilk 5 hisse (TOP 5) eşit ağırlıklı (%20'şer) portföy olarak ekrana yazdırılır.
