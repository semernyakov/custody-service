---
created: 2026-01-24 14:33
tags:
  - Vault
  - tasks-service-models
  - HashiCorpVault
author: Иван Семерняков
---
# **Базовая концепция**

### **Проблема, которую решает Auto-Unseal:**

В обычном режиме HashiCorp Vault требует **ручного распечатывания (unseal)** после каждого перезапуска. Для этого нужно:

- 3-5 операторов с разными ключами распечатки
- Каждый вводит свою часть ключа (shard)
- Только при наличии достаточного количества ключей Vault запускается

**Auto-unseal** автоматизирует этот процесс, используя внешний KMS для хранения ключа шифрования.
## **Как это работает**

### **Архитектура:**

```
┌─────────────────┐     ┌─────────────────────┐     ┌─────────────────┐
│   HashiCorp     │     │     Внешний KMS     │     │   KMS Provider  │
│     Vault       │────▶│  (AWS KMS, GCP KMS, │────▶│    (HSM под     │
│                 │     │    Azure Key Vault) │     │     капотом)    │
└─────────────────┘     └─────────────────────┘     └─────────────────┘
        │                         │                           │
        │ Master Key              │ Encryption Key            │ Root Key
        │ (encrypted)             │ (managed by KMS)          │ (in HSM)
        ▼                         ▼                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     Криптографический поток                          │
└──────────────────────────────────────────────────────────────────────┘
```
### **Детальный процесс:**

1. **Инициализация Vault:** 
	 - `vault operator init -recovery-shares=5 -recovery-threshold=3`
	 - Генерируются **recovery keys** (вместо unseal keys)
	 - **Root token** создается как обычно
	 - **Master key** шифруется с помощью KMS и хранится зашифрованным
2. **Запуск Vault с auto-unseal:**
3. Vault запускается
4. Запрашивает у KMS расшифровку master key
5. KMS проверяет права доступа (IAM, сервисный аккаунт)
6. Возвращает расшифрованный master key
7. Vault использует master key для расшифровки своих данных
8. Vault готов к работе (автоматически!)
## **Конфигурация**

### **Пример конфигурации Vault (HCL):**

```hcl
# config.hcl
storage "raft" {
  path = "/vault/data"
  node_id = "vault_1"
}

listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = true
}

# Конфигурация Auto-Unseal через AWS KMS
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "alias/vault-auto-unseal"
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

# Или через Google Cloud KMS
seal "gcpckms" {
  project    = "my-project"
  region     = "global"
  key_ring   = "vault-keyring"
  crypto_key = "vault-auto-unseal-key"
  credentials = "/etc/vault/gcp-service-account.json"
}

# Или через Azure Key Vault
seal "azurekeyvault" {
  tenant_id      = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  client_id      = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  client_secret  = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  vault_name     = "my-vault-keyvault"
  key_name       = "vault-auto-unseal-key"
}
```
## **Типы ключей в системе**

| Ключ | Где хранится | Назначение | Кто имеет доступ |
|------|-------------|------------|------------------|
| **KMS Key** | Внешний KMS/HSM | Шифрует Master Key | Vault (через IAM/SA) |
| **Master Key** | Vault storage (зашифрованный) | Шифрует Root Key | Никто (расшифровывается только в памяти) |
| **Root Key** | Vault storage (зашифрованный) | Шифрует все данные Vault | Vault после unseal |
| **Recovery Keys** | У операторов (на бумаге) | Для disaster recovery | Операторы безопасности |
## **Преимущества**

### **1. Операционная простота:**

- **Автоматический запуск** после перезагрузки сервера
- **Нет необходимости** в ручном вводе unseal keys
- **Быстрое масштабирование** кластера
### **2. Безопасность:**

- **Разделение обязанностей** (separation of duties)
- **Ключи никогда не хранятся** на дисках Vault в открытом виде
- **Аудит доступа** через логи KMS
- **Ротация ключей** управляется KMS
### **3. Recovery-ориентированный дизайн:**

- **Recovery keys** остаются у людей
- **Можно отключить** auto-unseal при компрометации
- **План восстановления** через recovery keys
## **Недостатки и риски**

### **1. Зависимость от внешнего сервиса:**

- **KMS становится SPOF** (Single Point of Failure)
- **Сетевая зависимость** - если KMS недоступен, Vault не запустится
- **Региональные риски** - при падении региона облачного провайдера
### **2. Безопасность KMS:**

- **Компрометация IAM роли** = компрометация Vault
- **Чрезмерные права** у сервисного аккаунта
- **Отсутствие human oversight** - автоматизация может скрыть инциденты
### **3. Стоимость:**

- **Дополнительные расходы** на KMS ($1-3 за ключ в месяц + операции)
- **Исходящий трафик** на запросы к KMS
## **Рекомендации по реализации**

### **1. Безопасная конфигурация:**

```hcl
# Использование IAM Instance Profile вместо статических ключей
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "alias/vault-auto-unseal"
  # Не указывать access_key/secret_key - использовать IAM роль EC2
}

# Включить envelope encryption
seal "awskms" {
  # ... другие параметры ...
  endpoint = "kms.us-east-1.amazonaws.com"  # Использовать VPC endpoint
}
```
### **2. Disaster Recovery Plan:**

```
1. Хранить recovery keys в физическом сейфе
2. Регулярно тестировать recovery без auto-unseal
3. Иметь backup KMS ключа в другом регионе
4. Настроить кросс-регионную репликацию для критичных данных
```
### **3. Мониторинг:**
- **CloudTrail logs** за доступом к KMS
- **Метрики Vault** за unseal операциями
- **Alert при неудачном** auto-unseal
- **Периодическая проверка** recovery процесса
## **Практический пример восстановления**

### **Если KMS компрометирован:**

```bash
# 1. Остановить все инстансы Vault
systemctl stop vault

# 2. Изменить конфигурацию на manual unseal
# config.hcl
# seal "awskms" { ... }  # Закомментировать или удалить

# 3. Перезапустить Vault
systemctl start vault

# 4. Использовать recovery keys для ручного unseal
vault operator unseal -recovery-key <key1>
vault operator unseal -recovery-key <key2>
vault operator unseal -recovery-key <key3>
```
## **Сравнение с другими подходами**

| Метод | Автоматизация | Безопасность | Сложность | Recovery |
|-------|--------------|--------------|-----------|----------|
| **Manual Unseal** | Нет | Высокая | Высокая | Простой |
| **Auto-Unseal с KMS** | Полная | Средне-высокая | Средняя | Требует планирования |
| **Shamir's Secret Sharing** | Частичная | Высокая | Высокая | Гибкий |
| **Transit Auto-Unseal** | Полная | Высокая | Высокая | Зависит от другого Vault |
## **Вывод**

**Vault с auto-unseal через KMS** — это баланс между безопасностью и операционной эффективностью. Он идеально подходит для:

- **Продакшен сред**, где доступность критически важна
- **Организаций с DevOps** культурой
- **Систем с частыми деплоями** и перезапусками
- **Команд, где безопасность** делегирована облачному провайдеру

**Но требует:**
- Тщательного проектирования DRP
- Мониторинга зависимостей
- Регулярного тестирования recovery процедур

Это решение перекладывает ответственность за защиту корневого ключа шифрования на облачного провайдера и его HSM, что может быть приемлемым компромиссом для многих организаций.