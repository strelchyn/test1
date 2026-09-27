# Дослідження: підключення банків до застосунку особистих фінансів

Дата: 2026-09-27. Банки: **Monobank (UA)**, **Revolut**, **Raiffeisenbank a.s. (CZ)**.

## TL;DR

| Банк | Пряме API для приватної особи | Через агрегатор (PSD2) | Вигрузка файлів | Рекомендація |
|---|---|---|---|---|
| **Monobank** | ✅ **Так**, безкоштовно — Personal API з токеном | не потрібно (Україна не під PSD2) | PDF / XLS / CSV з застосунку | **Personal API + webhook** |
| **Revolut** | ❌ (API є лише для Business та ліцензованих TPP) | ✅ Enable Banking (безкоштовний "restricted" режим для власних рахунків) — *але див. застереження щодо резидентів ЧР* | CSV/Excel/PDF з app та web | **Enable Banking**, fallback — імпорт CSV/XLSX |
| **Raiffeisenbank CZ** | ❌ для особистих рахунків. ✅ **Premium API** — лише для бізнес-/ОСВЧ-рахунків (безкоштовна, сертифікат з інтернет-банкінгу) | ⚠️ Enable Banking **не підтримує** RB CZ; є Salt Edge, Finbricks — але це B2B, платно, з договором | CSV, ABO/GPC (GPC треба активувати, платно) | **Імпорт CSV**; якщо є рахунок підприємця — Premium API |

Ліцензія AISP (PSD2) для власного pet-проєкту **не потрібна**, якщо використовувати: Monobank Personal API, Enable Banking restricted mode та файловий імпорт.

---

## 1. Monobank — найпростіше

**Monobank Personal API** (`https://api.monobank.ua`) — офіційне, безкоштовне, для власного рахунку.

- **Отримання токена**: зайти на https://api.monobank.ua/, відсканувати QR у застосунку monobank → отримати `X-Token`. Токен, яким довго не користуються, може анулюватись.
- **Ендпоінти**:
  - `GET /personal/client-info` — клієнт, рахунки (картки, валюти, IBAN, баланси), "банки" (джари). Ліміт: 1 запит / 60 с.
  - `GET /personal/statement/{account}/{from}/{to}` — виписка. Ліміти: **1 запит / 60 с**, вікно **≤ 31 доба + 1 год (2 682 000 с)**, **≤ 500 транзакцій** за відповідь (нові зверху; якщо прийшло 500 — дотягувати з меншим `to`).
  - `POST /personal/webhook` — реєстрація URL; банк шле `{"type":"StatementItem","data":{"account":"…","statementItem":{…}}}` на кожну нову операцію → **real-time** синхронізація.
  - `GET /bank/currency` — курси, без авторизації.
- **Поля транзакції**: `id`, `time`, `description`, `mcc`, `originalMcc`, `hold`, `amount`, `operationAmount`, `currencyCode` (ISO 4217, числовий), `commissionRate`, `cashbackAmount`, `balance`, `comment`, `counterIban`, `counterName` тощо. Суми — у копійках.
- **Бонус**: MCC-код дає автоматичну категоризацію "з коробки".
- **Готові бібліотеки**: Python `monobank` (vitalik/python-monobank), `monobankua`; Go `vtopc/go-monobank`; Ruby `monobank`.
- Для сторонніх застосунків з багатьма користувачами є "корпоративний" варіант API (з заявкою до банку) — для особистого використання не потрібен.

**Історичний імпорт**: цикл по 31-денних вікнах з паузою 60 с між запитами (рік історії ≈ 12+ хвилин на рахунок).

## 2. Revolut

### 2.1 Пряме API — немає для особистих рахунків
- `developer.revolut.com` → **Open Banking API** лише для зареєстрованих TPP з eIDAS/OBIE сертифікатом (тобто ліцензія AISP).
- **Business API** — тільки для Revolut Business.

### 2.2 Через агрегатор — Enable Banking (рекомендовано)
GoCardless Bank Account Data (колишній Nordigen, був безкоштовним) **закрив нові реєстрації**. Найближча заміна — **Enable Banking** (Фінляндія, ліцензований AISP):
- Безкоштовний **"restricted production"** режим: створюєш production-застосунок у Control Panel, активуєш кнопкою "Activate by linking accounts", прив'язуєш **лише свої** рахунки.
- Автентифікація API: `application_id` + RSA-ключ (JWT).
- Ліміт PSD2 для фонового доступу — ~**4 запити на добу** на рахунок.
- Згода (consent) діє **до 180 днів**, потім потрібно пройти SCA знову (в застосунку Revolut).
- Revolut підтримується (використовується, наприклад, у Firefly III Data Importer, Home Assistant інтеграціях).

⚠️ **Застереження**: у документації Enable Banking по Чехії сказано, що вони **не надають послуги користувачам (PSU), які є резидентами ЧР**. Якщо ти резидент Чехії, Revolut-з'єднання теж може бути відхилене (Revolut Bank UAB — литовський, тож варто просто спробувати). Якщо не спрацює — див. п. 2.3 або платні Salt Edge / Tink / Yapily / TrueLayer (усі підтримують Revolut, але це B2B-контракти).

### 2.3 Вигрузка файлів
- App: рахунок → `…` → **Statement** → формат **Excel / CSV / PDF**, період — місяць або довільний діапазон. Web (app.revolut.com) — аналогічно.
- Виписка генерується **окремо для кожного валютного рахунку**.
- Колонки CSV: `Type, Product, Started Date, Completed Date, Description, Amount, Fee, Currency, State, Balance`.
- Є готовий парсер `ofxstatement-revolut` (Python), корисний як референс.

## 3. Raiffeisenbank a.s. (Чехія) — найскладніше

### 3.1 Пряме API
- **PSD2 API** (`developer.rb.cz`, Czech Open Banking Standard) — sandbox відкритий для всіх, **production — лише з ліцензією** від ČNB/регулятора та кваліфікованим eIDAS сертифікатом. Для приватної особи нереально.
- **Premium API** — власне API банку для підключення облікових систем:
  - доступне **тільки для бізнес-рахунків** (фірми та **podnikatelé/ОСВЧ**), **не** для osobní účet;
  - сам сервіс безкоштовний, але потрібні права "stahování výpisů" (за прайсом);
  - автентифікація — клієнтський сертифікат, який генерується в інтернет-банкінгу (Nastavení → Správa certifikátů), дійсний 5 років з поновленням щороку;
  - дає рахунки, баланси, історію транзакцій, виписки.
  - 👉 **Якщо в тебе є рахунок підприємця в RB — це найкращий шлях.**

### 3.2 Агрегатори
- **Enable Banking — RB CZ не підтримується** (станом на їхню документацію по CZ).
- **Salt Edge** (саме його використовує чеський Wallet by BudgetBakers), **Finbricks** (дочка Komerční banka, підтримує RB) — підтримують RB CZ, але це платні B2B-контракти, не для pet-проєкту.
- GoCardless — нові реєстрації закриті.

### 3.3 Вигрузка файлів (рекомендовано для особистого рахунку)
- Інтернет-банкінг: **Účty → Pohyby → Export** → **CSV** (без додаткової активації).
- **ABO/GPC** (чеський стандарт, підтримується всіма банками ЧР) — через "Výpisy", але **потрібна активація** (Nastavení → Nastavení pro účetní systémy, ABO1/ABO2), може бути платною.
- PDF-виписки — як запасний варіант (парсинг PDF ненадійний, краще уникати).

## 4. Інші варіанти (fallback / доповнення)

1. **Універсальний імпорт файлів** — CSV/XLSX/OFX/QIF/GPC/MT940/camt.053 з маппінгом колонок (як у Zenmoney). Має бути в застосунку в будь-якому разі.
2. **Парсинг email-сповіщень** — банки (зокрема RB) можуть слати e-mail про кожну операцію; можна парсити через IMAP/Gmail API. Хитко, але дає майже real-time.
3. **Автоматизація браузера** (Playwright) для вигрузки CSV — порушує умови банків і ламається через SCA. **Не рекомендую.**
4. **Отримати власну ліцензію AISP** — місяці роботи й великі витрати; має сенс тільки якщо застосунок стане продуктом для інших людей.

## 5. Рекомендована архітектура імпорту

```
Sources (адаптери)                  Core
┌───────────────────────────┐
│ MonobankApiSource (webhook│──┐
│   + polling 31d windows)  │  │    ┌──────────────────────────────┐
│ EnableBankingSource       │──┼──▶ │ Normalizer → Transaction     │
│   (Revolut, ≤4×/day)      │  │    │ (id, account, date, amount,  │
│ FileImportSource          │──┘    │  currency, payee, mcc, raw)  │
│   (RB CSV/GPC, Revolut    │       ├──────────────────────────────┤
│    CSV/XLSX, generic)     │       │ Dedup (bank id або hash      │
│ RbPremiumApiSource (опц.) │       │  date+amount+desc)           │
└───────────────────────────┘       │ Categorizer (MCC + правила)  │
                                    │ Transfers matcher (між       │
                                    │  своїми рахунками)           │
                                    └──────────────────────────────┘
```

Ключові моменти:
- Зберігати сирий `raw` payload кожної транзакції — для повторної категоризації/дебагу.
- Суми — цілі числа в мінорних одиницях + ISO-валюта; курси — з `/bank/currency` Monobank або ECB/ČNB.
- Дедуплікація обов'язкова: один і той самий рахунок можуть наповнювати і API, і файл.
- Розпізнавання переказів між своїми рахунками (напр., Revolut → RB) — щоб не рахувати їх як витрати/доходи.
- Секрети (токен Monobank, RSA-ключ Enable Banking, сертифікат RB) — тільки в зашифрованому сховищі, не в репозиторії.

## 6. План дій

1. **Monobank** — отримати токен на api.monobank.ua, перевірити `client-info` через curl. (15 хв)
2. **Enable Banking** — зареєструватися на enablebanking.com, створити production-застосунок, спробувати прив'язати Revolut. (30 хв) Якщо відмовлять через резидентство ЧР — Revolut через CSV.
3. **Raiffeisenbank** — з'ясувати, чи є рахунок підприємця (→ Premium API); інакше — вивантажити тестовий CSV з інтернет-банкінгу та надіслати мені (можна знеособлений), щоб написати парсер.
4. Надіслати інтерфейс застосунку → проєктуємо модель даних і MVP.

## Джерела

- Monobank Open API: https://api.monobank.ua/docs/ ; https://github.com/vitalik/python-monobank ; https://pkg.go.dev/github.com/OlexiyOdarchuk/go-monobank-sdk/personal ; https://github.com/siomochkin/monobank-open-api-documentation
- Revolut: https://developer.revolut.com/docs/guides/build-banking-apps/introduction-to-the-open-banking-api/who-can-access-the-api ; https://www.openbankingtracker.com/provider/revolut ; https://pypi.org/project/ofxstatement-revolut/
- Enable Banking: https://enablebanking.com/docs/markets/cz ; https://enablebanking.com/docs/api/linked-accounts/ ; https://enablebanking.com/docs/faq/ ; https://docs.firefly-iii.org/tutorials/data-importer/eb/ ; https://github.com/pschmitt/ha-enablebanking
- GoCardless: https://bankaccountdata.gocardless.com/new-signups-disabled
- Raiffeisenbank CZ: https://developer.rb.cz/start ; https://developer.rb.cz/psd2 ; https://www.rb.cz/podnikatele/ucty-a-platebni-styk/prime-bankovnictvi/premium-api ; https://www.rb.cz/attachments/podnikatele/vytvoreni-certifikatu-premium-api-rbcz.pdf ; https://www.rb.cz/attachments/elektronicke-bankovnictvi/IB-micro-formaty-vypisu.pdf ; https://github.com/JirkaChadima/cz-banking-psd2
- Агрегатори CZ: https://www.saltedge.com/products/account_information/partner_program ; https://www.abra.eu/napojeni-na-bankovni-api-pres-finbricks-nove-podporuje-raiffeisenbank/ ; https://www.finance.cz/clanky/503051-osobni-finance-appka/
