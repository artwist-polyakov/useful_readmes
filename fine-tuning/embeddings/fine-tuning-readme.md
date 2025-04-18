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

## Масштабирование обучения для больших наборов данных

Для обработки больших наборов данных (например, 10,000+ товаров) PyTorch предоставляет эффективные инструменты:

```python
import torch
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader, Dataset
import os
import time
import json

class CatalogDataset(Dataset):
    """Датасет для каталога товаров и запросов"""
    def __init__(self, training_pairs):
        self.examples = []
        for query, item, score in training_pairs:
            self.examples.append(InputExample(texts=[query, item], label=score))
    
    def __len__(self):
        return len(self.examples)
    
    def __getitem__(self, idx):
        return self.examples[idx]

def load_training_data(data_file):
    """Загрузка обучающих данных из файла"""
    with open(data_file, 'r', encoding='utf-8') as f:
        data = json.load(f)
    return data

def train_model_efficient(model, training_pairs, epochs=3, batch_size=32):
    """Эффективное обучение модели с использованием параллелизма PyTorch"""
    # Создаем датасет
    train_dataset = CatalogDataset(training_pairs)
    
    # Используем DataLoader с многопоточной загрузкой
    train_dataloader = DataLoader(
        train_dataset, 
        batch_size=batch_size, 
        shuffle=True,
        num_workers=4,  # Параллельная загрузка данных
        pin_memory=True  # Оптимизация для GPU
    )
    
    # Для нескольких GPU используем DataParallel
    if torch.cuda.device_count() > 1:
        print(f"Используем {torch.cuda.device_count()} GPU!")
        model = torch.nn.DataParallel(model)
    
    # Функция потерь и обучение
    train_loss = losses.CosineSimilarityLoss(model)
    
    # Стандартное обучение с оптимизацией PyTorch
    model.fit(
        train_objectives=[(train_dataloader, train_loss)],
        epochs=epochs,
        warmup_steps=100,
        show_progress_bar=True
    )
    
    return model

# Пример использования
def main():
    # Определяем устройство для обучения
    device = 'mps' if torch.backends.mps.is_available() else 'cuda' if torch.cuda.is_available() else 'cpu'
    print(f"Используется устройство: {device}")
    
    # Загружаем модель
    model = SentenceTransformer("sberbank-ai/sbert_large_mt_nlu_ru")
    model.to(device)
    
    # Загружаем данные
    training_pairs = load_training_data("catalog_data.json")
    
    # Обучаем модель
    start_time = time.time()
    model = train_model_efficient(model, training_pairs)
    elapsed_time = time.time() - start_time
    print(f"Обучение завершено за {elapsed_time:.2f} секунд")
    
    # Сохраняем модель
    model.save("custom_catalog_model")

if __name__ == "__main__":
    main()
```

### Распределенное обучение на нескольких машинах

Для очень больших датасетов или моделей можно использовать распределенное обучение PyTorch:

```python
# Запуск в терминале:
# python -m torch.distributed.launch --nproc_per_node=NUM_GPUS_PER_NODE train_distributed.py

import torch
import torch.distributed as dist
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader, Dataset, DistributedSampler
import os
import time
import json

def setup(rank, world_size):
    """Инициализация распределенного обучения"""
    os.environ['MASTER_ADDR'] = 'localhost'
    os.environ['MASTER_PORT'] = '12355'
    dist.init_process_group("nccl", rank=rank, world_size=world_size)

def cleanup():
    """Завершение распределенного обучения"""
    dist.destroy_process_group()

def train_distributed(rank, world_size, training_pairs):
    # Настройка распределенного обучения
    setup(rank, world_size)
    
    # Загрузка модели на каждом устройстве
    model = SentenceTransformer("sberbank-ai/sbert_large_mt_nlu_ru")
    model.to(rank)
    
    # Создание датасета и distributed sampler
    train_dataset = CatalogDataset(training_pairs)
    sampler = DistributedSampler(train_dataset, num_replicas=world_size, rank=rank)
    
    # DataLoader с sampler
    train_dataloader = DataLoader(
        train_dataset, 
        batch_size=32,
        sampler=sampler,
        num_workers=4,
        pin_memory=True
    )
    
    # Обучение
    train_loss = losses.CosineSimilarityLoss(model)
    model.fit(
        train_objectives=[(train_dataloader, train_loss)],
        epochs=3,
        warmup_steps=100,
        show_progress_bar=True
    )
    
    # Сохранение модели только на первом ранге
    if rank == 0:
        model.save("custom_catalog_model")
    
    cleanup()
```

Для больших объемов данных рекомендуется:

1. **Использовать встроенный параллелизм PyTorch** вместо собственной реализации асинхронности
2. **Оптимизировать процесс загрузки данных** с помощью `num_workers` в DataLoader
3. **Для нескольких GPU** использовать `DataParallel` или `DistributedDataParallel`
4. **Для обучения на нескольких машинах** настроить распределенное обучение через `torch.distributed`

## Заключение

Fine-tuning модели эмбеддингов позволяет существенно повысить качество векторного поиска для специфических предметных областей. Правильно настроенная модель обеспечивает более релевантные результаты, устойчивость к опечаткам и лучшее понимание терминологии.

При дообучении важно соблюдать баланс между специализацией модели и сохранением её общих языковых способностей, а также следить за рисками переобучения и дисбаланса данных.

## Дополнительные ресурсы

- [Документация Sentence Transformers](https://www.sbert.net/)
- [Руководство по fine-tuning в HuggingFace](https://huggingface.co/docs/transformers/main/en/training)
- [Документация LangChain по векторным хранилищам](https://python.langchain.com/docs/integrations/vectorstores/)
