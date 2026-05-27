# lig-observability

Моніторинг і структуроване логування для LIG-системи.
Єдиний контракт полів → OTEL Collector → VictoriaLogs HA.

---

## З чого починати

| Я... | Читай |
|------|-------|
| Розробник ліби | `docs/architecture.md` → `TECHNICAL_REQUIREMENTS_DEV.md` |
| Розробник вендорного адаптера | `contracts/[vendor]/commands.md` → `contracts/[vendor]/mapping.md` → `src/adapters/_Template/` |
| QA | `contracts/[vendor]/mapping.md` (тест-кейси в кожній секції) |
| DevOps | `infra/` → `tools/LogGenerator/README.md` |
| Хочу додати нового вендора | Читай нижче |

---

## Структура

```
docs/                          # Архітектура, вимоги до полів VL, ADR
contracts/                     # Каталоги команд і маппінги по кожному вендору
src/
  StructureLogger.Abstractions/ # Контракт (ILigVendor, ILigLogger, моделі)
  StructureLogger.Core/         # Pipeline, Channel, BatchWorker, OTLP sink
  StructureLogger.Vendors/      # Вбудовані маркери вендорів
  adapters/                     # Вендорні адаптери (по одному проекту)
tests/                         # Unit і integration тести
tools/LogGenerator/            # HTTP API генератор логів для тестування VL
infra/                         # OTEL Collector конфіги, docker-compose
```

---

## Як додати нового вендора

### Крок 1 — Контракт (аналітик + розробник адаптера)

```bash
cp -r contracts/_template contracts/[vendor-name]
```

Заповни `contracts/[vendor-name]/commands.md` і `contracts/[vendor-name]/mapping.md`.
Погодь з tech lead перед реалізацією.

### Крок 2 — Маркер вендора

Якщо вендор тимчасовий (до наступного релізу ліби) — оголоси маркер у своєму проекті:

```csharp
public sealed class MyVendor : ILigVendor
{
    public string Name     => "my_vendor";
    public string Protocol => "SIP";
}
```

Якщо вендор офіційний — додай у `StructureLogger.Vendors/` через PR.

### Крок 3 — Адаптер

```bash
cp -r src/adapters/_Template src/adapters/StructureLogger.Adapters.[VendorName]
```

Реалізуй маппінг відповідно до `contracts/[vendor-name]/mapping.md`.
Кожне поле з mapping.md — окремий рядок у адаптері.

### Крок 4 — Реєстрація в застосунку

```csharp
services.AddStructureLogger()
    .AddVendor<MyVendor>()
    .AddVendorAdapter<MyVendor, MyMessage, MyMessageAdapter>()
    .WriteToOtlp(o => o.OtlpEndpoint = "http://otel-collector:4318");
```

### Крок 5 — Тести і верифікація

1. Unit тест для кожного адаптера (`tests/StructureLogger.Adapters.[VendorName].Tests/`)
2. Запусти `LogGenerator` і перевір що поля присутні у VL
3. Заповни колонку "VL перевірено" у `contracts/[vendor-name]/mapping.md`

---

## Вимоги до полів у VictoriaLogs

→ `docs/victoria-logs-fields.md`

**Коротко:**
- `correlationId` — завжди string
- `mcc`, `mnc` — string (ведучий нуль)
- `lac`, `cellId` — число
- `_msg` — формується тільки лібою, адаптер не впливає
- Вендор-специфічне → `extra.[key]` (атомарні значення)

---

## Локальне середовище

```bash
# Запустити OTEL Collector + VictoriaLogs
docker-compose -f infra/docker-compose.yml up

# Запустити генератор логів
cd tools/LogGenerator/src/LogGenerator
dotnet run

# Swagger: http://localhost:5000
# VictoriaLogs UI: http://localhost:9428
```
