# README: Fine-tuning моделей эмбеддингов для улучшения векторного поиска

## Обзор

Данный документ описывает процесс дообучения (fine-tuning) модели эмбеддингов на специфических данных каталога товаров для улучшения качества семантического поиска. Это особенно полезно, когда общедоступные модели не достаточно хорошо справляются с конкретной предметной областью или языковыми нюансами.

## Содержание

1. [Проблемы стандартных моделей эмбеддингов](#проблемы-стандартных-моделей-эмбеддингов)
2. [Преимущества fine-tuning](#преимущества-fine-tuning)
3. [Подготовка данных для обучения](#подготовка-данных-для-обучения)
4. [Процесс fine-tuning](#процесс-fine-tuning)
5. [Оценка и тестирование модели](#оценка-и-тестирование-модели)
6. [Интеграция с векторным поиском](#интеграция-с-векторным-поиском)
7. [Риски и лучшие практики](#риски-и-лучшие-практики)
8. [Асинхронное обучение для больших наборов данных](#асинхронное-обучение-для-больших-наборов-данных)

## Проблемы стандартных моделей эмбеддингов

Предобученные модели эмбеддингов часто сталкиваются с ограничениями:

1. **Недостаточное понимание специфической терминологии** – модели могут не улавливать семантическую близость между специализированными терминами в определенной области.
2. **Проблемы с обработкой опечаток** – например, "дымо ход" vs "дымоход".
3. **Слабая дифференциация между похожими, но разными категориями товаров**.
4. **Неправильное взвешивание важности слов** в конкретной предметной области.

## Преимущества fine-tuning

Дообучение моделей эмбеддингов позволяет:

1. **Повысить точность поиска** для специфичных запросов.
2. **Улучшить устойчивость к опечаткам** и вариациям написания.
3. **Сократить зависимость от внешних API** (например, OpenAI).
4. **Снизить стоимость** по сравнению с использованием коммерческих API.
5. **Обеспечить более быстрый поиск** благодаря локальному размещению модели.

## Подготовка данных для обучения

Ключевой этап – создание качественных обучающих пар. Существует несколько типов таких пар:

### 1. Пары "запрос-товар"

Это основной тип обучающих данных, где:
- Первый элемент: пользовательский запрос или его вариация
- Второй элемент: релевантное наименование товара
- Третий элемент: оценка релевантности (от 0.0 до 1.0)

Примеры:
```python
["дымоход", "Сэндвич дымоход из нержавеющей стали", 1.0]
["дымоход утепленный", "Сэндвич дымоход из нержавеющей стали с утеплителем", 1.0]
["труба для отвода дыма", "Сэндвич дымоход из нержавеющей стали", 0.9]
```

### 2. Обучающие пары с негативными примерами

Важно включать примеры нерелевантных пар с низкими оценками:

```python
["дымоход утепленный", "Дрель шуруповерт с аккумулятором", 0.1]
["аккумуляторная дрель", "Сэндвич дымоход из нержавеющей стали", 0.1]
```

### 3. Источники данных для обучения

Данные можно получить из:
- Логов поиска пользователей
- Экспертной разметки данных
- Автоматически сгенерированных вариаций запросов
- Списков синонимов и связанных терминов

## Процесс fine-tuning

Процесс дообучения модели включает следующие шаги:

### 1. Установка необходимых зависимостей

```bash
pip install sentence-transformers torch scikit-learn
```

### 2. Базовый код для fine-tuning

```python
import torch
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader
import os
import time

# Определение устройства для обучения
device = 'mps' if torch.backends.mps.is_available() else 'cuda' if torch.cuda.is_available() else 'cpu'

# Загрузка базовой модели
base_model = "sberbank-ai/sbert_large_mt_nlu_ru"  # Для русского языка
model = SentenceTransformer(base_model)
model.to(device)

# Преобразование обучающих пар в формат InputExample
train_examples = []
for query, item, score in training_pairs:
    train_examples.append(InputExample(texts=[query, item], label=score))

# Создание DataLoader
batch_size = 8  # Оптимальное значение зависит от доступной памяти
train_dataloader = DataLoader(train_examples, shuffle=True, batch_size=batch_size)

# Настройка функции потерь
train_loss = losses.CosineSimilarityLoss(model)

# Обучение модели
model.fit(
    train_objectives=[(train_dataloader, train_loss)],
    epochs=3,  # Обычно достаточно 3-5 эпох
    warmup_steps=50,
    show_progress_bar=True,
    output_path="custom_catalog_model"  # Путь для сохранения модели
)
```

### 3. Параметры обучения

- **Эпохи (epochs)**: Обычно 3-5 эпох достаточно для дообучения. Большее количество может привести к переобучению.
- **Размер батча (batch_size)**: Зависит от доступной памяти GPU/CPU. Чем больше, тем быстрее обучение.
- **Функция потерь**: `CosineSimilarityLoss` оптимальна для задач векторного поиска.
- **Устройство обучения**: Обучение на GPU (CUDA) или MPS (для Mac с M-чипами) значительно быстрее.

## Оценка и тестирование модели

После обучения необходимо протестировать модель на типичных запросах:

```python
def test_model(model, test_queries, catalog_items):
    for query in test_queries:
        # Векторизация запроса и товаров
        query_embedding = model.encode(query, convert_to_tensor=True)
        item_embeddings = model.encode(catalog_items, convert_to_tensor=True)
        
        # Добавление размерности для сравнения
        query_embedding = query_embedding.unsqueeze(0)
        
        # Вычисление косинусной близости
        similarities = torch.nn.functional.cosine_similarity(
            query_embedding, item_embeddings
        )
        
        # Вывод топ-3 результатов
        top_indices = torch.argsort(similarities, descending=True)[:3]
        
        for i, idx in enumerate(top_indices):
            similarity = similarities[idx].item() * 100
            print(f"{i+1}. Схожесть: {similarity:.2f}% - {catalog_items[idx]}")
```

## Интеграция с векторным поиском

Пример интеграции с LangChain для векторного поиска:

```python
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_community.vectorstores import FAISS

# Использование дообученной модели для эмбеддингов
embedding_model = HuggingFaceEmbeddings(model_name="custom_catalog_model")

# Создание векторного хранилища
texts = [
    "Сэндвич дымоход из нержавеющей стали",
    "Сэндвич дымоход из нержавеющей стали с утеплителем",
    # Другие товары...
]

vector_store = FAISS.from_texts(texts, embedding_model)

# Выполнение поиска
query = "дымо ход с утеплением"
found_docs = vector_store.similarity_search_with_score(query, k=3)

# Вывод результатов
for i, (doc, score) in enumerate(found_docs):
    similarity = (1 - score) * 100
    print(f"{i+1}. Схожесть: {similarity:.2f}% - {doc.page_content}")
```

## Риски и лучшие практики

### Риски

1. **Переобучение (Overfitting)**:
   - Модель может слишком сильно адаптироваться к обучающим данным и плохо работать с новыми запросами.
   - Признаки: очень высокие показатели схожести на обучающих данных и низкие на тестовых.

2. **Дисбаланс данных**:
   - Если в обучающих данных преобладает одна категория товаров, модель может быть смещена в её сторону.

3. **Забывание (Catastrophic forgetting)**:
   - Модель может "забыть" общие языковые паттерны, фокусируясь только на специфичных данных.

4. **Вычислительные требования**:
   - Дообучение требует значительных вычислительных ресурсов для больших наборов данных.

### Лучшие практики

1. **Сбалансированный набор данных**:
   - Включайте примеры из всех категорий товаров.
   - Используйте как положительные, так и отрицательные примеры.

2. **Регуляризация**:
   - Не обучайте слишком много эпох (3-5 обычно достаточно).
   - Используйте небольшой learning rate для fine-tuning.

3. **Валидация**:
   - Отделите часть данных для валидации результатов.
   - Регулярно проверяйте модель на новых запросах, не входивших в обучение.

4. **Итеративное улучшение**:
   - Начните с малого набора данных и постепенно добавляйте новые примеры.
   - Сохраняйте промежуточные версии модели для сравнения.

## Асинхронное обучение для больших наборов данных

Для обработки больших наборов данных (например, 10,000+ товаров) полезно использовать асинхронные подходы:

```python
import asyncio
import torch
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader
import os
import time
import concurrent.futures

async def load_training_data(data_file):
    """Асинхронная загрузка данных из файла"""
    training_pairs = []
    # Загрузка данных...
    return training_pairs

async def create_batch_examples(training_pairs, start_idx, batch_size):
    """Создание батча обучающих примеров"""
    examples = []
    end_idx = min(start_idx + batch_size, len(training_pairs))
    
    for i in range(start_idx, end_idx):
        query, item, score = training_pairs[i]
        examples.append(InputExample(texts=[query, item], label=score))
        
    return examples

async def train_model_async(model, training_pairs, epochs=3, batch_size=32):
    """Асинхронное обучение модели по батчам"""
    train_loss = losses.CosineSimilarityLoss(model)
    
    for epoch in range(epochs):
        print(f"Эпоха {epoch+1}/{epochs}")
        
        # Создаем задачи для батчей
        tasks = []
        for i in range(0, len(training_pairs), batch_size):
            task = create_batch_examples(training_pairs, i, batch_size)
            tasks.append(task)
            
        # Обрабатываем батчи асинхронно
        batches = await asyncio.gather(*tasks)
        
        # Обучаем на каждом батче
        for batch_examples in batches:
            train_dataloader = DataLoader(batch_examples, shuffle=True, batch_size=len(batch_examples))
            model.fit(
                train_objectives=[(train_dataloader, train_loss)],
                epochs=1,
                warmup_steps=0,
                show_progress_bar=True
            )
    
    return model

# Пример использования
async def main():
    model = SentenceTransformer("sberbank-ai/sbert_large_mt_nlu_ru")
    device = 'mps' if torch.backends.mps.is_available() else 'cuda' if torch.cuda.is_available() else 'cpu'
    model.to(device)
    
    training_pairs = await load_training_data("data.json")
    model = await train_model_async(model, training_pairs)
    model.save("custom_catalog_model")

if __name__ == "__main__":
    asyncio.run(main())
```

## Заключение

Fine-tuning модели эмбеддингов позволяет существенно повысить качество векторного поиска для специфических предметных областей. Правильно настроенная модель обеспечивает более релевантные результаты, устойчивость к опечаткам и лучшее понимание терминологии.

При дообучении важно соблюдать баланс между специализацией модели и сохранением её общих языковых способностей, а также следить за рисками переобучения и дисбаланса данных.

## Дополнительные ресурсы

- [Документация Sentence Transformers](https://www.sbert.net/)
- [Руководство по fine-tuning в HuggingFace](https://huggingface.co/docs/transformers/main/en/training)
- [Документация LangChain по векторным хранилищам](https://python.langchain.com/docs/modules/data_connection/vectorstores/)
