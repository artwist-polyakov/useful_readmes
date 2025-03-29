# Руководство по yt-dlp: скачивание видео и решение проблем

yt-dlp — это мощный инструмент командной строки для скачивания видео с различных сайтов, включая YouTube, Vimeo, Twitter/X и многих других.

## Содержание
- [Установка](#установка)
- [Основное использование](#основное-использование)
- [Выбор качества видео](#выбор-качества-видео)
- [Работа с различными платформами](#работа-с-различными-платформами)
- [Полезные опции](#полезные-опции)
- [Решение проблем](#решение-проблем)
- [Примеры использования](#примеры-использования)
- [Обновление](#обновление)

## Установка

Репозиторий на гитхаб https://github.com/yt-dlp/yt-dlp/wiki/Installation

### Windows
```
winget install yt-dlp
```
или скачайте exe-файл с [официального сайта](https://github.com/yt-dlp/yt-dlp/releases).

### macOS
```
brew install yt-dlp
```

### Linux
```
sudo apt install yt-dlp
```
или
```
sudo pip install yt-dlp
```

## Основное использование

### Скачивание видео в лучшем качестве
```
yt-dlp https://www.youtube.com/watch?v=VIDEO_ID
```

### Просмотр доступных форматов
```
yt-dlp -F https://www.youtube.com/watch?v=VIDEO_ID
```

### Скачивание конкретного формата
```
yt-dlp -f FORMAT_ID https://www.youtube.com/watch?v=VIDEO_ID
```
где FORMAT_ID — номер из списка форматов (например, 22 для 720p MP4).

## Выбор качества видео

### Лучшее качество
```
yt-dlp -f "best" URL
```

### Конкретное разрешение (например, 1080p)
```
yt-dlp -f "bestvideo[height<=1080]+bestaudio/best[height<=1080]" URL
```

### Только аудио (MP3)
```
yt-dlp -x --audio-format mp3 URL
```

### Скачивание с ограничением по разрешению
```
yt-dlp -f "bestvideo[height<=720]+bestaudio/best[height<=720]" URL
```

## Работа с различными платформами

### YouTube
```
yt-dlp https://www.youtube.com/watch?v=VIDEO_ID
```

### Vimeo
```
yt-dlp https://vimeo.com/VIDEO_ID
```

### X.com (Twitter)
```
yt-dlp https://x.com/username/status/TWEET_ID
```

### Facebook
```
yt-dlp https://www.facebook.com/watch/?v=VIDEO_ID
```

### Instagram
```
yt-dlp https://www.instagram.com/p/POST_ID/
```

### TikTok
```
yt-dlp https://www.tiktok.com/@username/video/VIDEO_ID
```

## Полезные опции

### Скачивание с субтитрами
```
yt-dlp --write-sub URL
```

### Выбор имени выходного файла
```
yt-dlp -o "%(title)s.%(ext)s" URL
```

### Скачивание плейлиста
```
yt-dlp --yes-playlist URL
```

### Избирательное скачивание элементов плейлиста
```
yt-dlp --playlist-items 1,3,5-7 URL
```

### Ограничение скорости загрузки
```
yt-dlp --limit-rate 1M URL
```

### Продолжение незавершенной загрузки
```
yt-dlp -c URL
```

## Решение проблем

### Проблема: Видео не воспроизводится в определенных плеерах (QuickTime, презентации)

Если видео воспроизводится в VLC, но не в QuickTime или не имеет превью в мессенджерах, используйте следующие команды:

#### 1. Ремуксинг в MP4
```
yt-dlp -f FORMAT_ID --remux-video mp4 URL
```

#### 2. Перекодирование для максимальной совместимости
```
yt-dlp -f FORMAT_ID --recode-video mp4 URL
```

#### 3. Для встраивания в презентации
```
yt-dlp -f "bestvideo[height<=1080][ext=mp4]+bestaudio[ext=m4a]" --merge-output-format mp4 URL
```

### Проблема: "Видео недоступно в вашей стране"

```
yt-dlp --geo-bypass URL
```

### Проблема: Ошибки SSL или соединения

Если возникают ошибки SSL (например, "UNEXPECTED_EOF_WHILE_READING"):

```
yt-dlp --no-check-certificate URL
```

### Проблема: Истекшие токены (403 Forbidden)

Если получаете ошибку 403 при использовании прямых ссылок с токенами авторизации:

1. Используйте основной URL видео вместо прямой ссылки на медиафайл 
2. yt-dlp автоматически получит новые токены

### Проблема: Скачивание защищенного контента

Добавьте опцию cookies, если сайт требует авторизацию:

```
yt-dlp --cookies cookies.txt URL
```

## Примеры использования

### Пример 1: Скачивание видео с YouTube в формате MP4 1080p
```
yt-dlp -f "bestvideo[height<=1080][ext=mp4]+bestaudio[ext=m4a]" --merge-output-format mp4 https://www.youtube.com/watch?v=VIDEO_ID
```

### Пример 2: Скачивание видео с Vimeo с определенным качеством
```
yt-dlp -f "bestvideo[height=1080]+bestaudio" https://vimeo.com/VIDEO_ID
```

### Пример 3: Извлечение только аудио из видео на YouTube в MP3
```
yt-dlp -x --audio-format mp3 --audio-quality 0 https://www.youtube.com/watch?v=VIDEO_ID
```

### Пример 4: Скачивание плейлиста с YouTube
```
yt-dlp -o "%(playlist_index)s-%(title)s.%(ext)s" https://www.youtube.com/playlist?list=PLAYLIST_ID
```

### Пример 5: Скачивание видео с X.com (Twitter)
```
yt-dlp https://x.com/username/status/TWEET_ID
```

## Обновление

### Обновление yt-dlp до последней версии
```
yt-dlp -U
```

---

Это руководство охватывает основные функции yt-dlp, но инструмент имеет множество дополнительных опций. Для получения полной информации обратитесь к [официальной документации](https://github.com/yt-dlp/yt-dlp/blob/master/README.md) или используйте команду `yt-dlp --help`.
