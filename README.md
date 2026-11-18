# ИИ-сервис обнаружения скрытых промптов в изображениях

Практическая часть магистерской работы **«Разработка модели обнаружения
скрытых текстовых инструкций в изображениях при изменении масштаба»**.

Сервис обнаруживает **image-scaling attack** (Trail of Bits, Anamorpher):
инструкцию, невидимую в полном разрешении и проявляющуюся после даунскейла,
и **отличает вредоносную инструкцию от безобидного скрытого текста**.

Атаки для обучения/проверки генерируются **настоящими солверами Trail of
Bits** (`vendor/anamorpher/`, Apache-2.0).

---

## 1. Архитектура — двухстадийный детектор

```
            ┌─────────────── СТАДИЯ 1 ───────────────┐   ┌──────── СТАДИЯ 2 ────────┐
изображение → признаки масштабирования → есть скрытый?  → OCR проявленного текста →
            → (RandomForest, detector.joblib)          → классификатор вредоносности
                                                          (text_intent.joblib)
                                              │
                                              ▼
                        Итоговый вердикт (3 класса):
                        • clean            — скрытого текста нет
                        • hidden_benign    — скрытый текст есть, но не вредоносный
                        • hidden_malicious — скрытая вредоносная инструкция
```

- **Стадия 1** отвечает «есть ли скрытый при масштабировании текст» — по
  признакам расхождения ядер при даунскейле (не читает смысл).
- **Стадия 2** читает проявленный текст (OCR по красному каналу) и классифицирует
  его как вредоносную инструкцию или безобидный текст (TF-IDF + LogReg + правила).

Так решается требование: отличать надпись «HELLO WORLD» от вредоносной команды.

---

## 2. Структура

```
hpdetector/
├── vendor/anamorpher/          # НАСТОЯЩИЙ код Trail of Bits (Apache-2.0)
│   ├── bicubic/bilinear/nearest_gen_payload.py, decoy_images/, LICENSE
├── dataset/
│   ├── anamorpher_attack.py    # обёртка: генерация атак, скрытность, проявление
│   ├── fast_embed.py           # векторизованный bicubic-солвер (4000px за ~2с)
│   ├── prompts.py              # реалистичные промпты + отрисовка мелкого текста
│   └── build_dataset.py        # сборка датасета (3072px, тёмные decoy, гейт)
├── research/
│   ├── benchmarks.py           # независимый генератор + LSB/alpha/метаданные
│   └── run_studies.py          # что детектор ловит, а что нет (отчёт)
├── detector/
│   ├── features.py             # 81 признак сигнатуры масштабирования
│   ├── train.py                # обучение стадии 1 + честная CV-оценка
│   ├── text_intent.py          # стадия 2: классификатор вредоносности текста
│   ├── infer.py                # двухстадийный инференс (3 класса) + OCR проявления
│   ├── evaluate_robustness.py  # устойчивость (JPEG/шум/размытие/рескейл)
│   └── cnn_vit_train.py        # ⚠ обучение CNN/ViT для GPU (здесь НЕ запускалось)
├── scripts/make_holdout.py     # независимый набор (decoy 'кот') + 3-классовая оценка
├── service/app.py              # веб-интерфейс + REST API
├── models/                     # detector.joblib, text_intent.joblib
└── reports/                    # метрики, графики, галереи
```

---

## 3. Быстрый старт

```bash
sudo apt install -y tesseract-ocr libgl1 libglib2.0-0
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

python dataset/build_dataset.py --out data/dataset --n 250 --size 3072  # 1) датасет 3072px
python detector/text_intent.py                               #    стадия 2 (классификатор текста)
python detector/train.py --jobs 2                            # 2) стадия 1 (детектор)
python scripts/make_holdout.py --n 40 --seed 2025            # 3) независимая проверка
python research/run_studies.py --n 20                        #    исследование границ (стего и др.)
python detector/evaluate_robustness.py                       #    устойчивость
python service/app.py                                        # 4) веб-сервис :8000
```

Исследование результатов и ограничений — в **`RESEARCH.md`** (положительные и
отрицательные результаты, графики в `reports/`). Для источников 4000×4000:
`--size 4000`.

```text
Коротко из RESEARCH.md: детекция scaling-атак — CV 0.998, независимый decoy 1.0,
обобщение на другой генератор 100%; НЕ ловит LSB/alpha/метаданные (0%);
стадия 2 (вредоносность) на пережатых данных ~0.70 (зависит от OCR).
```

Проверка одного изображения (3-классовый вердикт + распознанный текст):

```bash
python detector/infer.py data/holdout/attack/ho_attack_000_nearest.png
```

---

## 4. Веб-интерфейс

Открыть `http://127.0.0.1:8000`. Загрузите изображение — страница показывает:

- **загруженное фото** (что видит человек) и **результат даунскейла 4:1**
  (что получает ИИ-модель — там проявляется payload), рядом;
- цветной вердикт: зелёный (чисто) / оранжевый (скрытый текст, не вредоносный) /
  красный (скрытая вредоносная инструкция);
- распознанный скрытый текст и две шкалы вероятности (скрытость, вредоносность).

REST: `curl -F "file=@img.png" http://127.0.0.1:8000/api/detect`.

---

## 5. Результаты (честная оценка)

| Оценка | Значение | Файл |
|---|---|---|
| Стадия 1, кросс-валидация по всем данным | accuracy **0.998** (1 ошибка/500), AUC 1.0 | `cv_metrics.json` |
| Классификатор текста (стадия 2), CV | F1 **1.0** на корпусе фраз | — |
| Независимый набор (decoy 'кот'), стадия 1 | 100% (обобщение на невиданный decoy) | `holdout/predictions.csv` |
| Независимый набор, **3 класса** | accuracy **0.99** | `holdout/gallery.png` |
| Устойчивость: JPEG q75–q30, рескейл | 1.0 | `robustness.json` |
| Устойчивость: **шум** / **размытие** | acc **0.72** / **0.93** | `robustness.json` |

Почему стадия 1 даёт почти 100% и это НЕ переобучение: признаки 4:1 напрямую
ловят определяющее свойство атаки (зависимость даунскейла от ядра), а
независимый набор на другом decoy это подтверждает. Реальные границы видны при
**сдвиге распределения**: аддитивный шум даёт ложные срабатывания (acc 0.72),
размытие сглаживает ВЧ-сигнатуру и снижает recall (bicubic 0.77). Это материал
для раздела «ограничения».

Чтобы оценка была честной, в датасет добавлены **трудные негативы** (муар,
ВЧ-текстуры, пережатые JPEG, уже уменьшенные фото — расхождение ядер БЕЗ
payload) и **скрытные/пережатые** атаки.

---

## 6. Скрытность атак

Payload прячется в тёмных текстурированных зонах. Качество скрытия зависит от
decoy и ядра:

- **bicubic, nearest** — скрытные (краснота в полном разрешении ≈0–5, текст
  невидим). Используются в датасете.
- **bilinear** — принципиально заметнее (широкое ядро), поэтому в обучающий
  набор не включён; поддерживается в обёртке как «видимый» случай.

`build_dataset.py` выбирает **самый тёмный** кроп реального decoy и применяет
**гейт скрытности**: атака принимается, только если payload невидим в полном
разрешении И проявляется при даунскейле.

---

## 7. Переход на CNN/ViT (GPU)

`detector/cnn_vit_train.py` — готовый скрипт обучения CNN (ResNet-18) или ViT.

> ⚠ Требует GPU и PyTorch и **не запускался** в среде сборки проекта (там нет
> torch/GPU). Запускать на вашей машине с GPU.

На вход сети подаётся 6-канальный тензор, явно несущий сигнатуру атаки:
каналы 0–2 — bicubic-даунскейл 4:1 (проявляет payload), каналы 3–5 — разность
(bicubic − area) (area payload не проявляет → разность ≈ скрытый контент).

```bash
pip install torch torchvision timm
python detector/cnn_vit_train.py --data data/dataset --arch resnet18 --epochs 20
python detector/cnn_vit_train.py --data data/dataset --arch vit --epochs 20
```

---

## 8. Развёртывание (VirtualBox, Ubuntu Server 22.04)

```bash
sudo apt install -y git python3-venv tesseract-ocr libgl1 libglib2.0-0
cd ~/hpdetector && python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python detector/text_intent.py && python dataset/build_dataset.py --out data/dataset --n 250
python detector/train.py --jobs 2
```

systemd-служба `/etc/systemd/system/hpdetect.service` (замените `ilya` на своего
пользователя; при ошибке `status=217/USER` — неверный `User=`):

```ini
[Unit]
Description=Hidden Prompt Detector
After=network.target
[Service]
User=ilya
WorkingDirectory=/home/ilya/Desktop/hpdetector
Environment=PORT=8000
ExecStart=/home/ilya/Desktop/hpdetector/.venv/bin/python service/app.py
Restart=on-failure
[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl reset-failed hpdetect
sudo systemctl enable --now hpdetect && sudo systemctl status hpdetect
```

---

## 9. Ограничения

- Атаки заточены под отношение **4:1** и конкретное ядро; иные коэффициенты —
  направление расширения.
- Классический ML (стадия 1) силён на своём семействе атак, но деградирует при
  шуме и размытии (`robustness.json`). Переход на CNN/ViT (раздел 7) — путь к
  устойчивости.
- Стадия 2 зависит от качества OCR: при сильной деградации текст может читаться
  с ошибками (единичные промахи malicious/benign на независимом наборе).
- Образцы синтетические; для продакшена нужен набор реальных атак и фото.
