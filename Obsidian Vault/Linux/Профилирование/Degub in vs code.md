Создай файл в корне проекта `.vscode/launch.json` со следующим содержанием:

```json
{
	"version": "0.2.0",
	"configurations": [
		{
			"name": "C++ Debug (External Build)",
			"type": "cppdbg",
			"request": "launch",
			"program": "${workspaceFolder}/build/<yor-app>", // Путь к бинарнику
			"args": [],
			"stopAtEntry": true,
			"cwd": "${workspaceFolder}",
			"environment": [],
			"externalConsole": false,
			"MIMode": "gdb",
			"setupCommands": [
				{
					"description": "Enable pretty-printing for gdb",
					"text": "-enable-pretty-printing",
					"ignoreFailures": true
				}
			],
			"miDebuggerPath": "/usr/bin/gdb"
		}
	]
}
```

Открой `code . ` и можно запускать в Debug режиме

### Сборка и дебаг через vs code

Добавляем `tasks.json` для сборки через CMake

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Build with CMake",
      "type": "shell",
      "command": "mkdir -p build && cd build && cmake .. && make",
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": ["$gcc"]
    }
  ]
}
```

Собери проект:

- Нажми `Ctrl+Shift+B` → выбери `Build with CMake`
- Либо открой терминал и выполни: `mkdir -p build && cd build && cmake .. && make`

- Запусти отладку:
    - Нажми `F5`
    - Установи точки останова в `main.cpp`



### Дебаг приложений запущеных в докер

```json
{
	"version": "0.2.0",
	"configurations": [
		{
			"name": "C++ Debug in Docker via GDB Server",
			"type": "cppdbg",
			"request": "launch",
			"program": "${workspaceFolder}/build/main", // Путь к локальной версии бинарника
			"args": [],
			"stopAtEntry": true,
			"cwd": "${workspaceFolder}",
			"environment": [],
			"externalConsole": false,
			"MIMode": "gdb",
			"miDebuggerServerAddress": "localhost:1234", // IP и порт gdbserver в контейнере
			"miDebuggerPath": "/usr/bin/gdb",
			"setupCommands": [
				{
					"description": "Enable pretty-printing for gdb",
					"text": "-enable-pretty-printing",
					"ignoreFailures": true
				}
			]
		}
	]
}
```


1. запускаем докер образ с такими параметрами для дебага -`docker run --cap-add=SYS_PTRACE -p 1234:1234 -it <image-name> bash`
2. После того как мы оказались в bash внутри docker  образа, нужно запустить -  `gdbserver :1234 main` - номер порта и название программы должны совпадать с теми что указаны в launch.json файле