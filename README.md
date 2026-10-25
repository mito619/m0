# DnD Scene Studio — установка полной GPU-версии

Локальный редактор фэнтезийных сцен: создание фона и персонажей, удаление фона, изменение позы, размещение объектов и финальная сборка изображения. Эта копия содержит исправления поз и финального рендера от 26.10.2026.

В GitHub-архив включены исходники, файлы зависимостей и скрипты установки/загрузки. Модели, установленные библиотеки, кэши и пользовательские сцены скачиваются или создаются отдельно. Без установки библиотек и моделей полная GPU-версия сразу после распаковки не запустится. Исключение этих файлов из архива не удаляет их из вашей существующей установки.

## 1. Что установить заранее

- Windows 11 x64 и Python **3.12 x64** с Python Launcher (`py`). Проверка: `py -3.12 --version`.
- Node.js **22 LTS** с npm. Проверка: `node --version` и `npm --version`.
- Видеокарта NVIDIA и драйвер с поддержкой CUDA 12.8. Проверка: `nvidia-smi`.
- Для проверенной конфигурации использовались RTX 5060 с 8 ГБ видеопамяти и 32 ГБ оперативной памяти. На 8 ГБ применяется последовательная выгрузка моделей в RAM; редактирование поз может работать медленно. Для другой видеокарты потребление и скорость могут отличаться.
- Желательно 80–100 ГБ свободного места для моделей, окружений, кэшей и временных частей загрузки. Сам FLUX занимает около 16 ГБ, при сборке загрузки нужны дополнительные временные файлы.
- Интернет при установке. Для Hugging Face иногда требуется VPN или HTTP-прокси.

Отдельный CUDA Toolkit для этих команд не требуется: используется готовая сборка PyTorch с CUDA. Команда установки приведена по [официальной таблице PyTorch](https://pytorch.org/get-started/previous-versions/).

Откройте PowerShell **в папке проекта**, где лежит этот README. Все следующие относительные пути отсчитываются от неё. Не используйте системный Python для установки библиотек проекта.

## 2. Установить библиотеки

### Сервер приложения

```powershell
py -3.12 -m venv backend/.venv
./backend/.venv/Scripts/python.exe -m pip install --upgrade pip
./backend/.venv/Scripts/python.exe -m pip install -r backend/requirements.txt
```

`backend/requirements.txt` содержит FastAPI, Uvicorn, Pillow, NumPy, SciPy, OpenCV Headless, HTTPX и pytest. Этот сервер хранит сцены, рассчитывает геометрию и собирает результат. Его окружение отделено от GPU.

### GPU-сервер

```powershell
py -3.12 -m venv .venv-gpu
./.venv-gpu/Scripts/python.exe -m pip install --upgrade pip
./.venv-gpu/Scripts/python.exe -m pip install torch==2.9.0 torchvision==0.24.0 --index-url https://download.pytorch.org/whl/cu128
./.venv-gpu/Scripts/python.exe -m pip install -r worker/requirements-gpu-locked.txt
```

Основные версии взяты из работавшего окружения:

| Библиотека | Версия | Назначение |
|---|---|---|
| torch / torchvision | 2.9.0+cu128 / 0.24.0+cu128 | NVIDIA GPU и распознавание суставов |
| diffusers | 0.35.2 | DreamShaper XL и старый режим SDXL |
| transformers | 4.57.6 | Текстовые энкодеры, переводчик, IP-Adapter |
| accelerate | 1.15.0 | Выгрузка частей модели в RAM |
| huggingface-hub | 0.36.2 | Загрузка и кэш моделей |
| safetensors / sentencepiece | 0.8.0 / 0.2.2 | Веса и токенизация |
| rembg / onnxruntime | 2.0.86 / 1.31.0 | Вырезание фона на CPU |
| ultralytics | 8.4.187 | Сегментация SAM2.1 |
| numpy / scipy | 2.5.4 / 1.18.1 | Массивы и обработка изображений |
| opencv-python / pillow | 5.0.0.93 / 13.0.0 | Изображения и маски |
| fastapi / uvicorn | 0.162.0 / 0.54.0 | GPU API |
| python-multipart / httpx | 0.0.32 / 0.28.1 | Передача изображений |
| matplotlib | 3.11.2 | Зависимости обработки/визуализации |

Полный список прямых зависимостей находится в `worker/requirements-gpu-locked.txt`. Дополнительные зависимости pip установит автоматически; это список проверенных основных версий, а не полный lock всех транзитивных пакетов.

### Новый редактор больших изменений позы — отдельная папка библиотек

```powershell
./.venv-gpu/Scripts/python.exe -m pip install --no-deps --target pose-edit-packages -r worker/requirements-pose-edit.txt
```

Устанавливаются **diffusers 0.37.1, mediapipe 1.1.1, absl-py 2.5.1, sounddevice 0.5.6 и flatbuffers 25.12.19**. Остальные библиотеки редактор использует из GPU-окружения. `cffi` и `matplotlib` предусмотрены в основном списке.

**Не устанавливайте этот файл обычной командой pip без `--target` и `--no-deps`.** Базовый сервер использует Diffusers 0.35.2, новый редактор — 0.37.1 в отдельном процессе. Глобальная замена версии может сломать один из режимов. Не добавляйте `pose-edit-packages` в общий `PYTHONPATH` запуска GPU-сервера.

### Интерфейс

```powershell
Push-Location frontend
npm ci
Pop-Location
```

React 18, React Konva, Konva, TypeScript и Vite устанавливаются по `frontend/package-lock.json`. Папка `node_modules` в GitHub не нужна.

## 3. Куда сохранять модели

По умолчанию используются `.local/models`, `.local/huggingface`, `.local/rembg` и `.local/torch-cache` внутри проекта. Для другого диска перед загрузкой задайте, например:

```powershell
$env:DND_MODEL_ROOT = 'D:\DnDModels'
. ./scripts/model-paths.ps1
```

Если модели уже находятся в `D:\CodexModels\dnd-scene-studio`, можно указать этот существующий каталог. Скрипт загрузки FLUX проверяет уже имеющиеся файлы и их контрольные суммы.

**Выбранный `DND_MODEL_ROOT` нужно задавать и при загрузке, и перед каждым запуском GPU-сервера в новом окне PowerShell.** Автоматического чтения `.env` в этих скриптах нет. Без переменной используется `.local`.

## 4. Какие модели скачать

| Модель и источник | Для чего нужна | Куда попадёт | Группа загрузки |
|---|---|---|---|
| [Lykon/dreamshaper-xl-1-0](https://huggingface.co/Lykon/dreamshaper-xl-1-0), FP16 Diffusers | Генерация сцен/персонажей и SDXL-обработка | `models/dreamshaper-xl` | `core` |
| [BiRefNet-general через rembg](https://github.com/danielgatis/rembg) | Основное удаление фона, около 973 МБ | Кэш `rembg` с подпапками библиотеки | `core` |
| ISNet general use через rembg | Запасная маска, около 179 МБ | Кэш `rembg` | `core` |
| [SAM2.1 base, sam2.1_b.pt](https://docs.ultralytics.com/models/sam-2/) | Уточнение сегментации | `models/sam2.1_b.pt` | `core` |
| [big-lama.pt](https://github.com/enesmsahin/simple-lama-inpainting/releases/tag/v0.1.0) | Заполнение удалённых участков | `models/big-lama.pt` | `core` |
| [Helsinki-NLP/opus-mt-ru-en](https://huggingface.co/Helsinki-NLP/opus-mt-ru-en) | Перевод русских описаний | Кэш `huggingface` | `core` |
| [Torchvision Keypoint R-CNN ResNet50 FPN](https://docs.pytorch.org/vision/stable/models/generated/torchvision.models.detection.keypointrcnn_resnet50_fpn.html), COCO | Автоматические точки скелета, около 237 МБ | `torch-cache/hub/checkpoints` | `core` |
| [black-forest-labs/FLUX.2-klein-4B](https://huggingface.co/black-forest-labs/FLUX.2-klein-4B), около 16 ГБ | Новый режим больших изменений позы | `models/flux2-klein-4b` | `pose-edit` |
| [Google MediaPipe HandLandmarker](https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task), 7,8 МБ | Проверка лишних рук в новом режиме | `models/hand_landmarker.task` | `pose-edit` |
| [xinsir/controlnet-openpose-sdxl-1.0](https://huggingface.co/xinsir/controlnet-openpose-sdxl-1.0) | Старый режим SDXL/OpenPose | Кэш `huggingface` | `legacy`, необязательно |
| [h94/IP-Adapter](https://huggingface.co/h94/IP-Adapter) | Сохранение образа в старом SDXL-режиме | Кэш `huggingface` | `legacy`, необязательно |

Все пути в таблице относительны `DND_MODEL_ROOT`. Для основных функций и нового редактора достаточно **core + pose-edit**. `legacy` нужен для доступного в интерфейсе старого режима сравнения. Бережное геометрическое движение не требует FLUX, но имеет ограничения величины движения.

### Загрузка командами

После установки библиотек, из корня проекта:

```powershell
. ./scripts/model-paths.ps1
./.venv-gpu/Scripts/python.exe scripts/download-models.py --list
./.venv-gpu/Scripts/python.exe scripts/download-models.py --group core
./.venv-gpu/Scripts/python.exe scripts/download-models.py --group pose-edit
```

Для старого режима дополнительно:

```powershell
./.venv-gpu/Scripts/python.exe scripts/download-models.py --group legacy
```

Или всё одной командой: `./.venv-gpu/Scripts/python.exe scripts/download-models.py --group all`.

Дождитесь завершения каждой команды без ошибки. Повторный запуск позволяет продолжить загрузку: Hugging Face использует свой кэш, FLUX проверяет готовые файлы и использует скачанные части. Прямые загрузки небольших моделей сохраняются сначала в `.partial`; незавершённые файлы не переименовываются в готовые. При подозрении на повреждение такой модели удалите только её файл и повторите загрузку.

### Что должно находиться в папке DreamShaper

Нужна **папка формата Diffusers**, а не один файл checkpoint:

```text
models/dreamshaper-xl/
  model_index.json
  scheduler/scheduler_config.json
  text_encoder/config.json
  text_encoder/model.fp16.safetensors
  text_encoder_2/config.json
  text_encoder_2/model.fp16.safetensors
  tokenizer/       # json/txt файлы токенизатора
  tokenizer_2/     # json/txt файлы токенизатора
  unet/config.json
  unet/diffusion_pytorch_model.fp16.safetensors
  vae/config.json
  vae/diffusion_pytorch_model.fp16.safetensors
```

### Что должно находиться в папке FLUX

Загрузчик закреплён на ревизии `e7b7dc27f91deacad38e78976d1f2b499d76a294`:

```text
models/flux2-klein-4b/
  model_index.json
  scheduler/scheduler_config.json
  text_encoder/config.json
  text_encoder/generation_config.json
  text_encoder/model-00001-of-00002.safetensors
  text_encoder/model-00002-of-00002.safetensors
  text_encoder/model.safetensors.index.json
  tokenizer/       # включая tokenizer.json и chat_template.jinja
  transformer/config.json
  transformer/diffusion_pytorch_model.safetensors
  vae/config.json
  vae/diffusion_pytorch_model.safetensors
  download-manifest.json
models/hand_landmarker.task
```

`download-manifest.json` создаётся скриптом после проверки размеров и SHA-256 весов. Без него новый редактор считается неустановленным. Не создавайте пустой файл вручную. Временные части остаются в `.download-parts` для возобновления; после успешной загрузки всех файлов их можно удалить для освобождения места.

Для IP-Adapter скрипт скачивает именно `models/image_encoder/config.json`, `models/image_encoder/model.safetensors` и `sdxl_models/ip-adapter-plus_sdxl_vit-h.safetensors`. Tencent T2I Adapter текущему режиму не требуется.

## 5. Запуск — три окна PowerShell

Каждое окно сначала откройте в корне проекта. Старый `start.bat` запускает только интерфейс и backend; для полной GPU-версии используйте порядок ниже.

**Окно 1 — GPU:**

```powershell
# Если выбрали другой диск, повторите здесь тот же путь:
# $env:DND_MODEL_ROOT = 'D:\DnDModels'
./scripts/start-gpu.ps1
```

**Окно 2 — сервер приложения:**

```powershell
./scripts/start-backend.ps1
```

**Окно 3 — интерфейс:**

```powershell
Set-Location frontend
npm run dev -- --host 127.0.0.1
```

Откройте **http://127.0.0.1:5173/**. Если Vite выбрал другой порт, используйте адрес из его окна. Порты 8000 и 8001 должны быть свободны. Модели загружаются лениво: первый запрос после запуска заметно дольше последующих. Для остановки нажмите Ctrl+C в каждом из трёх окон.

Если Windows блокирует запуск `.ps1`, можно разрешить выполнение только для текущего окна: `Set-ExecutionPolicy -Scope Process Bypass`, затем повторить запуск. Системную политику менять не требуется.

## 6. Проверка установки

```powershell
./.venv-gpu/Scripts/python.exe -c "import torch; print(torch.__version__); print('CUDA:', torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'GPU unavailable')"
Invoke-RestMethod http://127.0.0.1:8000/health | ConvertTo-Json -Depth 8
Invoke-RestMethod http://127.0.0.1:8001/api/health | ConvertTo-Json -Depth 8
```

Нужно `CUDA: True`. У GPU API параметр `pose_edit` должен подтверждать доступность нового редактора. Наличие файлов ещё не подтверждает успешную генерацию: создайте сцену, персонажа и один вариант позы через интерфейс. Надпись «GPU подключена» сама по себе не означает, что все необязательные модели уже установлены.

Для проверок кода:

```powershell
Push-Location backend
./.venv/Scripts/python.exe -m pytest -q
Pop-Location
Push-Location frontend
npm run build
Pop-Location
```

Для исходной рабочей конфигурации проходили 32 backend-теста и сборка интерфейса. Проверялись генерация вариантов позы и сохранение пикселей персонажа на последнем нейронном этапе финального рендера. Полная чистая установка всех зависимостей на другом ПК пока не проверялась. Новые скрипты не скачивают модели при обычном запуске приложения заранее.

## 7. Частые проблемы

**Failed to fetch.** Проверьте оба адреса `/health`, открытые окна серверов и ошибки в них. Backend должен иметь `WORKER_URL=http://127.0.0.1:8000`; наш скрипт выставляет его сам. VPN не должен перехватывать localhost.

**Hugging Face сбрасывает соединение.** Включите рабочий VPN. Если он даёт локальный HTTP-прокси, задайте фактический адрес перед загрузкой, например:

```powershell
$env:HTTPS_PROXY = 'http://127.0.0.1:10809'
$env:HTTP_PROXY = $env:HTTPS_PROXY
$env:NO_PROXY = '127.0.0.1,localhost'
```

Порт 10809 — только пример: уточните его в вашем VPN. ExitLag не обязательно проксирует Python. Если прокси выключен, уберите устаревшие переменные: `Remove-Item Env:HTTPS_PROXY,Env:HTTP_PROXY -ErrorAction SilentlyContinue`. При HTTP 401/403 проверьте доступ к странице модели и при необходимости выполните `./.venv-gpu/Scripts/huggingface-cli.exe login` своим токеном. Токен не включайте в репозиторий.

**pose_edit=false.** Проверьте одинаковый корень моделей при загрузке и запуске, файл `download-manifest.json`, соседний `hand_landmarker.task`, папки `pose-edit-packages/diffusers` и `pose-edit-packages/mediapipe`. После установки перезапустите GPU-сервер.

**CUDA недоступна.** Проверьте драйвер и что используете `.venv-gpu`, а не backend-окружение. Переустановите CUDA-сборку PyTorch указанной выше командой. macOS, AMD и работа полного пайплайна на CPU здесь не проверялись.

**Out of memory / Tensor on device meta.** Закройте другие программы, занимающие GPU, перезапустите GPU-сервер нашим скриптом. Не отключайте последовательную выгрузку и не объединяйте две версии Diffusers. Нужна свободная оперативная память и доступный файл подкачки; расход зависит от разрешения и режима.

**Модель/библиотека не найдена.** Устанавливайте зависимости из нужного файла requirements и нужным Python. Если старый SDXL-режим просит ControlNet/IP-Adapter, скачайте группу `legacy`. Если pip не находит закреплённую версию, сначала проверьте Python 3.12 x64 и доступность PyPI; не заменяйте все версии произвольно.

## 8. Как менять позу и чего ожидать

Для небольшого движения используйте бережный режим: он деформирует исходное изображение, защищая лицо и предметы. Слишком большие изменения он отклоняет. Для «сидит», «бежит» и других больших изменений используйте новый режим редактирования изображения на FLUX и выберите подходящий вариант.

Новый режим получает исходного персонажа и подсказку скелета. Автоматическая проверка рук отсеивает часть неудачных результатов, но не гарантирует правильные пальцы, инструмент или полное совпадение одежды. Проверяйте вариант перед сохранением. Финальный рендер сохраняет персонажа на последнем нейронном этапе, однако его видимый размер и освещение в сцене влияют на воспринимаемую детализацию.

## 9. Что загружать на GitHub

Загрузите содержимое папки с этим README, включая `.gitignore`, `backend`, `frontend`, `worker`, `scripts` и файлы зависимостей. Не загружайте модели, окружения, `node_modules`, кэши, токены и личные сцены. `.gitignore` уже исключает их.

После создания пустого репозитория на GitHub, в корне проекта:

```powershell
git init
git add .
git commit -m "Initial DnD Scene Studio source"
git branch -M main
# Замените URL на адрес СВОЕГО созданного репозитория:
git remote add origin https://github.com/YOUR_USER/YOUR_REPOSITORY.git
git push -u origin main
```

Для переноса личных сцен сделайте отдельную копию `backend/data` из старой установки при остановленном backend. В этот исходный архив сцены не включены. Лицензия на сам проект в этой поставке не назначена; условия использования моделей и библиотек смотрите в их исходных репозиториях.

Старый генератор `worker/make_notebook.py`, если используется отдельно, не является проверенным установщиком нового редактора. Эта инструкция описывает локальный Windows-запуск.
