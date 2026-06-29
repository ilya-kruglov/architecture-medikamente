# Механизм тегирования данных

## 1. Цели тегирования
- Автоматическая классификация данных при создании
- Контроль доступа на основе тегов (ABAC)
- Аудит операций с тегированными данными
- Автоматическое применение политик (шифрование, маскирование, удаление)

## 2. Архитектура тегирования
```
┌─────────────────────────────────────────────────────────────┐
│ Data Tagging Layer                                          │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐     │
│ │ Tag Engine  │ │ Policy Enf. │ │ Audit & Monitoring  │     │
│ └─────────────┘ └─────────────┘ └─────────────────────┘     │
│ ┌─────────────────────────────────────────────────────┐     │
│ │ Metadata Repository                                 │     │
│ │ (Tags, Policies, Lineage, Retention Rules)          │     │
│ └─────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

## 3. Таксономия тегов

### 3.1. Теги чувствительности (Sensitivity)
| Тег | Значение | Пример применения |
|-----|----------|-------------------|
| `SENSITIVITY::PII_SENSITIVE` | Критические ПДн/PII | ФИО, адрес, телефон |
| `SENSITIVITY::PII_GENERAL` | Общие ПДн/PII | Дата рождения |
| `SENSITIVITY::MEDICAL` | Медицинские данные | Диагноз, анализы |
| `SENSITIVITY::FINANCIAL` | Финансовые данные | Сумма оплаты |
| `SENSITIVITY::OPERATIONAL` | Операционные данные | Статистика |
| `SENSITIVITY::PUBLIC` | Публичные данные | Режим работы |

### 3.2. Теги обработки (Processing)
| Тег | Значение | Пример применения |
|-----|----------|-------------------|
| `PROCESS::ENCRYPT_AT_REST` | Шифровать при хранении | Все ПДн/PII и Medical |
| `PROCESS::ENCRYPT_IN_TRANSIT` | Шифровать при передаче | Все данные при передаче вне периметра ЦОДа (внешние API, интернет, партнёры) |
| `PROCESS::MASK_DISPLAY` | Маскировать при отображении | ПДн/PII-Sensitive |
| `PROCESS::ANONYMIZE` | Обезличить перед использованием | Для аналитики |
| `PROCESS::RETENTION_5Y` | Хранить 5 лет | Financial |
| `PROCESS::RETENTION_25Y` | Хранить 25 лет | Medical |

### 3.3. Теги доступа (Access)
| Тег | Значение | Пример применения |
|-----|----------|-------------------|
| `ACCESS::RESTRICTED` | Ограниченный доступ | Только владелец |
| `ACCESS::ROLE_ADMIN` | Доступ у администраторов | Данные ресепшена |
| `ACCESS::ROLE_DOCTOR` | Доступ у врачей | Медицинские данные |
| `ACCESS::ROLE_CASHIER` | Доступ у кассиров | Финансовые данные |
| `ACCESS::ROLE_ACCOUNTANT` | Доступ у бухгалтеров | Бухгалтерские данные |

### 3.4. Теги происхождения (Lineage)
| Тег | Значение | Пример применения |
|-----|----------|-------------------|
| `SOURCE::PATIENT` | Данные от пациента | Регистрационная форма |
| `SOURCE::DOCTOR` | Данные от врача | Диагноз |
| `SOURCE::LABORATORY` | Данные из лаборатории | Результаты анализов |
| `SOURCE::SYSTEM` | Системные данные | Логи, метрики |

## 4. Механизм применения тегов

### 4.1. Автоматическое тегирование при создании
```yaml
# Пример политики автоматического тегирования
policies:
  - name: "Patient Registration"
    trigger: "on_create /patients"
    tags:
      - "SENSITIVITY::PII_SENSITIVE"
      - "PROCESS::ENCRYPT_AT_REST"
      - "PROCESS::ENCRYPT_IN_TRANSIT"
      - "ACCESS::ROLE_ADMIN"
      - "SOURCE::PATIENT"
    fields:
      - field: "full_name" -> tag: "PII_SENSITIVE"
      - field: "phone" -> tag: "PII_SENSITIVE"
      - field: "diagnosis" -> tag: "MEDICAL"

  - name: "Laboratory Results"
    trigger: "on_upload /laboratory/*"
    tags:
      - "SENSITIVITY::MEDICAL"
      - "PROCESS::ENCRYPT_AT_REST"
      - "PROCESS::RETENTION_25Y"
      - "ACCESS::ROLE_DOCTOR"
      - "SOURCE::LABORATORY"
```

### 4.2. Ручное тегирование (интерфейс администратора)
- Web-интерфейс для просмотра и редактирования тегов
- Bulk-операции для существующих данных
- История изменений тегов (audit)

### 4.3. Интеграция с DLP-системой
- Автоматическое обнаружение ПДн/PII в неструктурированных данных
- Предложение тегов на основе анализа содержимого
- Автоматическое применение тегов при соответствии паттернам

## 5. Инструменты реализации
| Компонент | Инструмент | Назначение |
|-----------|------------|------------|
| Tag Engine | Apache Atlas / OpenMetadata | Управление метаданными и тегами |
| Policy Enforcement | Open Policy Agent (OPA) | Применение политик доступа |
| Metadata Repository | PostgreSQL + Elasticsearch | Хранение и поиск метаданных |
| DLP Integration | Custom ML models | Обнаружение ПДн/PII |
| Audit | ELK Stack + Custom | Логирование операций с тегами |

## 6. Пример использования в API
```http
POST /api/v1/patients
Content-Type: application/json
X-Tags: SENSITIVITY::PII_SENSITIVE, ACCESS::ROLE_ADMIN

{
  "full_name": "Иванов Иван Иванович",
  "phone": "+7-999-123-45-67",
  "diagnosis": "Грипп"
}
```
Response:
```json
{
  "id": "pat-123",
  "tags": ["SENSITIVITY::PII_SENSITIVE", "SENSITIVITY::MEDICAL", ...],
  "data": { ... }
}
```