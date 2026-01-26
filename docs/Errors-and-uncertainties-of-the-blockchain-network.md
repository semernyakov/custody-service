---
created: 2026-01-25T14:46:00
tags:
  - blockchain
  - ошибки-блокчейн-сети
author: Иван Семерняков
---
# Ошибки и неопределённости сети блокчейн в контексте custody-service

## **Ключевые проблемы блокчейн-сетей**

### **1. Ненадёжность отправки транзакций:**

```
Ваша транзакция → Сеть → ?
            ↓
     Может зависнуть, потеряться,
     быть заменена, не дойти
```

### **2. Проблема finality (финальности):**

- **Proof of Work (Bitcoin, Ethereum 1.x):** probabilistic finality
- **Proof of Stake (Ethereum 2.0):** eventual finality
- **Некоторые блокчейны:** instant finality
- **Рекорганы:** могут откатывать историю
### **3. Динамическая стоимость газа:**

- **Сети перегружены** → газ дорожает в 100-1000 раз
- **Front-running:** майнеры/валидаторы выбирают дорогие транзакции
- **Gas wars:** конкуренция за место в блоке
## **Детальный разбор проблем MVP**

### **Проблема 1: Отсутствие мониторинга статуса транзакций**

**Что происходит в MVP:**

```python
# Упрощенный код MVP
def send_transaction(recipient, amount):
    tx_hash = wallet.send_transaction(recipient, amount)
    return tx_hash  # И всё! Что дальше?
```

**Риски:**

- Транзакция зависла в mempool
- Низкая комиссия → никогда не войдет в блок
- Конфликт nonce → блокировка последующих транзакций
- Двойная трата (unintentional)
### **Проблема 2: Нет логики работы с газом**

**Пример плохой ситуации:**

```python
# Фиксированная цена газа (катастрофа!)
GAS_PRICE = 50  # gwei

# В нормальное время: транзакция проходит за 30 секунд
# Во время NFT-дропа или DeFi-ажиотажа:
# - Средняя цена газа: 2000 gwei
# - Ваша транзакция: никогда не исполнится
# - Средства зависли на 24+ часа
```
### **Проблема 3: Нет finality проверок**

**Сценарий риска:**

```
1. Отправлена транзакция
2. Получено 1 подтверждение (confirmation)
3. Custody-service считает перевод завершенным
4. Происходит реорган блокчейна (reorg)
5. Транзакция исчезает из истории
6. Клиент получил средства, но они "вернулись"
```
### **Проблема 4: Нет обработки повторных отправок**

**Что происходит при ошибке:**

```
Попытка 1: Fail (low gas)
Попытка 2: Fail (network error)
Попытка 3: ... А её нет! Средства зависли
```
## **Решение в v2: Детальный разбор мер**

### **1. Мониторинг статуса транзакций и confirmations**

**Архитектура мониторинга:**

```python
class TransactionMonitor:
    def __init__(self):
        self.pending_txs = {}  # hash -> TransactionInfo
    
    async def monitor_transaction(self, tx_hash: str):
        """Мониторинг транзакции до финальности"""
        while True:
            status = await self.get_transaction_status(tx_hash)
            
            if status == "CONFIRMED":
                confirmations = await self.get_confirmations(tx_hash)
                
                # Ждем достаточное количество подтверждений
                if confirmations >= self.get_required_confirmations():
                    await self.mark_as_final(tx_hash)
                    break
                    
            elif status == "FAILED":
                await self.handle_failed_transaction(tx_hash)
                break
                
            elif status == "DROPPED":
                await self.handle_dropped_transaction(tx_hash)
                break
                
            await asyncio.sleep(15)  # Проверяем каждые 15 секунд
```

**Таблица required confirmations:**

| Блокчейн | Безопасные confirmations | Для больших сумм |
|----------|--------------------------|------------------|
| Bitcoin  | 6 подтверждений | 12+ подтверждений |
| Ethereum | 12 подтверждений | 30+ подтверждений |
| Polygon  | 60 подтверждений | 120+ подтверждений |
| BSC      | 15 подтверждений | 30+ подтверждений |
| Solana   | 1 подтверждение (finality) | 1 подтверждение |
### **2. RBF (Replace-By-Fee) / Retry Policy**

**RBF стратегии:**

```python
class RBFManager:
    def __init__(self):
        self.max_retries = 3
        self.gas_bump_percentage = 110  # +10% каждый раз
        self.max_gas_multiplier = 2.0  # Не более 2x от начального
    
    async def replace_transaction(self, old_tx, reason):
        """Замена зависшей транзакции"""
        
        strategies = {
            "low_gas": self._bump_gas_strategy,
            "stuck": self._accelerate_strategy,
            "nonce_conflict": self._cancel_strategy
        }
        
        strategy = strategies.get(reason, self._default_strategy)
        new_tx = await strategy(old_tx)
        
        return await self.send_with_retry(new_tx)
    
    def _bump_gas_strategy(self, tx):
        """Увеличение цены газа"""
        new_gas_price = tx.gas_price * self.gas_bump_percentage / 100
        new_gas_price = min(new_gas_price, 
                           tx.gas_price * self.max_gas_multiplier)
        return tx.copy(gas_price=new_gas_price)
```

**Retry Policy Matrix:**

| Ошибка | Действие | Макс. попыток | Таймаут между |
|--------|----------|---------------|---------------|
| Nonce too low | Ждать + retry | 5 | 30 сек |
| Insufficient funds | Стоп, алерт | 1 | - |
| Gas too low | Bump gas + retry | 3 | 60 сек |
| Network error | Exponential backoff | 7 | 2^n сек |
### **3. Gas Estimation и алерты**

**Умный gas estimation:**

```python
class GasEstimator:
    def __init__(self):
        self.history_window = 100  # блоков
        self.percentile = 70  # 70-й перцентиль
    
    async def estimate_optimal_gas(self):
        """Адаптивная оценка газа"""
        
        # 1. Берем исторические данные
        history = await self.get_gas_history()
        
        # 2. Учитываем время суток (ночью дешевле)
        hour = datetime.now().hour
        time_multiplier = self._get_time_multiplier(hour)
        
        # 3. Учитываем день недели (пн/пт дороже)
        weekday = datetime.now().weekday()
        weekday_multiplier = self._get_weekday_multiplier(weekday)
        
        # 4. Учитываем спец-события (NFT дропы и т.д.)
        events = await self.check_calendar_events()
        event_multiplier = self._get_event_multiplier(events)
        
        # 5. Вычисляем оптимальную цену
        base_gas = np.percentile(history, self.percentile)
        optimal_gas = (base_gas * time_multiplier 
                      * weekday_multiplier * event_multiplier)
        
        # 6. Добавляем safety margin
        optimal_gas *= 1.15  # +15% запаса
        
        return {
            "slow": optimal_gas * 0.8,
            "standard": optimal_gas,
            "fast": optimal_gas * 1.3,
            "urgent": optimal_gas * 2.0
        }
```

**Система алертов:**

```yaml
alerts:
  gas_price_spike:
    condition: "gas_price > threshold * 5"
    channels: ["slack", "pagerduty"]
    severity: "HIGH"
    
  transaction_stuck:
    condition: "tx_pending_time > 1h"
    channels: ["slack"]
    severity: "MEDIUM"
    
  low_confirmations_rate:
    condition: "confirmations_per_hour < 10"
    channels: ["email", "slack"]
    severity: "LOW"
```

### **4. Reconciliation Job (Сверочное задание)**

**Архитектура reconciliation:**

```python
class ReconciliationJob:
    def __init__(self):
        self.cron_schedule = "*/30 * * * *"  # Каждые 30 минут
        self.max_discrepancy = 0.001  # 0.1% допустимое расхождение
    
    async def run_reconciliation(self):
        """Сверка балансов и транзакций"""
        
        # 1. Получаем балансы из разных источников
        db_balance = await self.get_database_balance()
        blockchain_balance = await self.get_blockchain_balance()
        accounting_balance = await self.get_accounting_balance()
        
        # 2. Сравниваем
        discrepancies = self.find_discrepancies(
            db_balance, blockchain_balance, accounting_balance
        )
        
        # 3. Если есть расхождения > допустимого
        if discrepancies > self.max_discrepancy:
            await self.trigger_investigation(discrepancies)
            
            # 4. Пытаемся автоматически исправить
            if self.can_auto_fix(discrepancies):
                await self.auto_reconcile()
            else:
                await self.notify_humans()
        
        # 5. Логируем результат
        await self.log_reconciliation_result(
            discrepancies, 
            "auto_fixed" if fixed else "needs_manual"
        )
```

**Reconciliation Report:**

```json
{
  "timestamp": "2024-01-24T10:30:00Z",
  "status": "discrepancy_found",
  "balances": {
    "database": "100.5 ETH",
    "blockchain": "100.0 ETH",
    "accounting": "100.5 ETH"
  },
  "discrepancy": {
    "amount": "0.5 ETH",
    "percentage": "0.5%",
    "suspected_reason": "unconfirmed_transaction",
    "transactions_in_dispute": ["0xabc..."]
  },
  "action_taken": "alert_sent",
  "next_check": "2024-01-24T11:00:00Z"
}
```

## **Реализация в custody-service v2**

### **Архитектура системы:**

```
┌─────────────────────────────────────────────────────┐
│                 Transaction Manager                 │
├─────────────────────────────────────────────────────┤
│  Gas Estimator → RBF Engine → Monitor → Reconciler  │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│                State Machine (транзакции)           │
├─────────────────────────────────────────────────────┤
│ PENDING → MINED → CONFIRMING → FINALIZED / FAILED   │
└─────────────────────────────────────────────────────┘
```
### **Пример конфигурации:**

```yaml
transaction_policy:
  default:
    required_confirmations: 12
    max_gas_price_gwei: 200
    rbf_enabled: true
    retry_policy:
      max_retries: 3
      backoff_factor: 2
      
  critical:
    required_confirmations: 30
    max_gas_price_gwei: 1000  # Готовы заплатить за скорость
    rbf_enabled: true
    retry_policy:
      max_retries: 5
      backoff_factor: 1.5
      
  low_value:
    required_confirmations: 6
    max_gas_price_gwei: 50
    rbf_enabled: false  # Экономим на комиссиях
    retry_policy:
      max_retries: 1
```
### **Безопасные timeouts:**

```python
# Время ожидания перед эскалацией
TIMEOUTS = {
    "tx_mined": timedelta(minutes=30),      # 30 минут для попадания в блок
    "tx_confirmed": timedelta(hours=2),     # 2 часа для подтверждений
    "reconciliation": timedelta(hours=6),   # 6 часов для сверки
    "full_finality": timedelta(days=1)      # 1 день для полной финальности
}
```
## **Метрики и мониторинг для v2**

### **Key Performance Indicators:**

1. **Transaction Success Rate**: % успешных транзакций
2. **Average Confirmation Time**: среднее время подтверждения
3. **Gas Efficiency**: цена газа / байт данных
4. **Reconciliation Discrepancy Rate**: % расхождений при сверке
5. **RBF Success Rate**: % успешных замен транзакций

### **Дашборд мониторинга:**

```
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  TX Success:    │ │ Avg Confirm:    │ │ Gas Price:      │
│     98.7%       │ │   45s           │ │   42 gwei       │
└─────────────────┘ └─────────────────┘ └─────────────────┘
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  Pending TXs:   │ │ RBF Attempts:   │ │ Discrepancies:  │
│      3          │ │     12          │ │     0.1%        │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```
## **Риски, которые остаются даже в v2**

### **1. Long-range attacks:**

- Атаки на Proof-of-Stake сети
- Требует социального консенсуса
### **2. MEV (Miner Extractable Value):**

- Фронтраннинг ваших транзакций
- Sandwich attacks для DeFi
### **3. Протокольные риски:**

- Хардфорки без обратной совместимости
- Изменения в EIP (Ethereum)
### **Меры:**

```python
# Защита от MEV
def apply_mev_protection(tx):
    # Использование приватных mempool (Flashbots)
    # Установка maxPriorityFee и maxFee
    # Пакетные транзакции (bundle)
    return protected_tx
```
## **Вывод**

**v2 custody-service** должен реализовать:

1. **Полный цикл мониторинга** транзакций от отправки до финальности
2. **Адаптивную логику газа** с учетом времени, событий и истории
3. **Умные retry-стратегии** с RBF поддержкой
4. **Автоматическую сверку** (reconciliation) для обнаружения расхождений
5. **Многоуровневую систему алертов** для оперативного реагирования

**Без этих механизмов:**

- Риск потери средств из-за зависших транзакций
- Невозможность обработки пиковых нагрузок сети
- Расхождения в учете (discrepancies)
- Ручная работа операторов 24/7

**С этими механизмами:**

- Автоматическое восстановление при сбоях
- Оптимизация комиссий на 30-50%
- Гарантированная доставка транзакций
- Прозрачный аудит всех операций

> Это не просто улучшение UX, а критически важные функции для любого production-ready custody-сервиса, обрабатывающего реальные средства клиентов.