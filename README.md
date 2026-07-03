# Warden Detector

Windows x86 DLL для обнаружения модуля Warden по сигнатуре `BLL2` в памяти процесса WoW (TBC/CLASSIC).
Инструмент выполняет только детектирование Warden и сохраняет сырой снимок его
`VirtualAlloc`-аллокации; поиск и восстановление произвольных PE-модулей не выполняются.

## Сборка

Требуются Visual Studio 2019+ и CMake. Проект рассчитан на x86, поэтому генератор
нужно запускать с `-A Win32`:

```powershell
cmake -S . -B build -A Win32 -DBUILD_TESTING=ON
cmake --build build --config Release --parallel
```

После сборки в `build/Release/` появятся:

* `WardenDetector.dll` — детектор;
* `Injector.exe` — DLL-инжектор;
* `TestApp.exe` — smoke-test.

## Тесты

```powershell
ctest --test-dir build -C Release --output-on-failure
```

Smoke-test выполняет два независимых запуска. Первый создаёт Warden до загрузки DLL
и проверяет первичный полный скан. Второй создаёт Warden после установки хуков и
проверяет детект через `VirtualProtect`. Для обоих проверяются размер, сигнатура и
данные сырого дампа. Дополнительная исполняемая аллокация без `BLL2` подтверждает
отсутствие ложного дампа.

## Инъекция

```powershell
Injector.exe --pid <process_id> [path_to_WardenDetector.dll]
Injector.exe --name <process_name.exe> [path_to_WardenDetector.dll]
```

Если путь не указан, инжектор загружает `WardenDetector.dll` из папки, в которой
находится `Injector.exe`. Явно переданный абсолютный или относительный путь имеет
приоритет.

Инжектор принимает логи через канал `\\.\pipe\WardenDetectorPipe`. Перед ручным
`FreeLibrary` необходимо вызвать экспорт `WardenDetectorShutdown`, чтобы снять хуки
и дождаться завершения worker-потока вне loader lock.

Экспорт `WardenDetectorWaitForReady(timeout_ms)` позволяет дождаться установки хуков
и завершения первичного полного сканирования.

## Выходные файлы

* `warden_detector.log` — журнал DLL;
* `warden_<base_address>_<timestamp>.bin` — сырой дамп Warden-аллокации.

## Ограничения

* Поддерживается только Windows x86 и вариант Warden с сигнатурой `BLL2`.
* Аллокация ограничена размером 256 МБ; недоступные страницы сохраняются как нули.
* Для сборки MinHook `v1.3.4` CMake требуется доступ к GitHub.
* Инструмент предназначен только для анализа процессов, на которые у вас есть разрешение.
* Детектор не изменяет код или поведение Warden.
