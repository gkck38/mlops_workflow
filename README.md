# Banka Kampanyası MLOps Workflow

Bu proje, bir banka müşterisinin telefon görüşmesi öncesinde vadeli mevduat
(`term deposit`) satın alma olasılığını tahmin eden bir makine öğrenmesi
akışını uçtan uca gösterir:

```text
veri -> keşif ve hazırlık -> model eğitimi -> Docker/FastAPI servisi
     -> canlı tahmin -> istek logları -> veri kayması (drift) analizi
```

## Proje içeriği

| Dosya | Açıklama |
| --- | --- |
| [`lesson.ipynb`](./lesson.ipynb) | Önerilen uçtan uca akış: EDA, leakage önleme, model eğitimi, test, servis ve PSI ile drift analizi |
| [`main.ipynb`](./main.ipynb) | Duplicate/leakage kontrolü, farklı modellerin karşılaştırılması ve optimize edilmiş model denemeleri |
| [`data/bank-additional-full.csv`](./data/bank-additional-full.csv) | UCI Bank Marketing verisi; `;` ayraçlı, yaklaşık 41 bin telefon görüşmesi |
| [`serve/app.py`](./serve/app.py) | FastAPI tahmin servisi |
| [`serve/model.joblib`](./serve/model.joblib) | Eğitilmiş scikit-learn pipeline'ı |
| [`serve/reference.json`](./serve/reference.json) | Eğitim dağılımı; canlı veriyle drift karşılaştırmasının referansı |
| [`serve/Dockerfile`](./serve/Dockerfile) | Servis imajının tanımı |
| [`logs/`](./logs/) | Tahmin isteklerinin JSONL formatındaki logları |

## Temel kararlar

- Hedef değişken `y`, `yes` ve `no` değerlerinden 0/1 olarak üretilir.
- `duration`, görüşme bitmeden bilinmediği için modelden çıkarılır; böylece
  data leakage önlenir.
- Veri dosya sırasına göre bölünür: %70 train, %15 test ve %15 live. Live
  bölümü eğitim ve test sırasında kullanılmaz.
- Sayısal özellikler ölçeklenir, kategorik özellikler one-hot encoding ile
  dönüştürülür. Dönüşümler yalnızca train bölümünde fit edilir.
- `serve/app.py`, olasılığı [`THRESHOLD`](./serve/app.py) değeri olan `0.5884`
  ile karşılaştırarak `yes`/`no` etiketi üretir.
- Her `/predict` isteği `logs/requests.jsonl` dosyasına yazılır. Eğitim
  dağılımı [`reference.json`](./serve/reference.json) içinde saklanır ve PSI
  (Population Stability Index) ile canlı dağılım karşılaştırılır.

## Kurulum

Python 3.10+ ve Docker gereklidir.

### Windows PowerShell

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

PowerShell script çalıştırma politikası nedeniyle aktivasyon engellenirse,
venv'i aktive etmeden de `.\.venv\Scripts\python.exe` ve
`.\.venv\Scripts\pip.exe` komutlarını kullanabilirsiniz.

### macOS/Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

## Notebook'ları çalıştırma

1. VS Code veya Jupyter'de notebook'u açın.
2. Kernel olarak `.venv` ortamını seçin.
3. [`lesson.ipynb`](./lesson.ipynb) hücrelerini yukarıdan aşağıya çalıştırın.
   Bu notebook, `serve/` altındaki model ve servis dosyalarını yeniden
   oluşturur.
4. Test bölümünde classification report, confusion matrix ve ROC-AUC
   metriklerini inceleyin.

`main.ipynb`, alternatif bir analiz akışıdır. Duplicate kayıtları temizler,
`pdays` üzerinden `previously_contacted` özelliği üretir ve logistic
regression, gradient boosting ve benzeri modelleri karşılaştırır.

## Servisi Docker ile çalıştırma

Notebook'un Docker bölümünü çalıştırdıktan sonra proje kök dizininde:

### Windows PowerShell

```powershell
docker build -t bank-campaign .\serve
New-Item -ItemType Directory -Force .\logs | Out-Null
docker run --rm -p 8000:8000 -v "${PWD}\logs:/logs" bank-campaign
```

### macOS/Linux

```bash
docker build -t bank-campaign ./serve
mkdir -p logs
docker run --rm -p 8000:8000 -v "$PWD/logs:/logs" bank-campaign
```

Container çalışırken servisin durumunu kontrol edin:

```text
GET http://localhost:8000/health
```

Beklenen yanıt:

```json
{"status": "ok"}
```

Tek bir müşteri için tahmin:

```powershell
@{
  age = 35
  job = "technician"
  marital = "single"
  education = "university.degree"
  default = "no"
  housing = "yes"
  loan = "no"
  contact = "cellular"
  month = "may"
  day_of_week = "mon"
  campaign = 1
  pdays = 999
  previous = 0
  poutcome = "nonexistent"
  "emp.var.rate" = 1.1
  "cons.price.idx" = 93.994
  "cons.conf.idx" = -36.4
  euribor3m = 4.857
  "nr.employed" = 5191.0
} | ConvertTo-Json | Invoke-RestMethod -Method Post `
  -Uri http://localhost:8000/predict -ContentType "application/json"
```

Yanıt biçimi:

```json
{"label": "no", "probability": 0.1234}
```

`lesson.ipynb` içindeki canlı inference bölümünü çalıştırmak, live
bölümündeki müşterileri servise gönderir ve istekleri `logs/requests.jsonl`
dosyasına kaydeder. Ardından notebook, sayısal ve kategorik özellikler için
PSI hesaplar:

- `< 0.10`: stable
- `0.10 - < 0.20`: small shift
- `>= 0.20`: drift

İşiniz bittiğinde container'ı `Ctrl+C` ile durdurun.

## Gereksinimler

Ana ortam bağımlılıkları [`requirements.txt`](./requirements.txt), yalnızca
servis imajında kullanılan bağımlılıklar ise
[`serve/requirements.txt`](./serve/requirements.txt) içindedir.
