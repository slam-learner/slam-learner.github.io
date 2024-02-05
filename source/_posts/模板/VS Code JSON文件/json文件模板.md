# VS Code JSON 文件模板

`launch.json` 示例：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "RunDebug",
            "type": "lldb", // or cppdbg
            "request": "launch",
            "program": "${workspaceFolder}/bin/debug/bin_name", // 可执行文件路径
            "args": [
                "arg1",
                "arg2"
            ], // 命令行参数
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MiMode": "lldb", // or gdb
            "miDebuggerPath": "/usr/bin/lldb", // 调试器路径(/usr/bin/lldb or /usr/bin/gdb)
            "setupCommands": [
                { // 模板自带，好像可以更好地显示STL容器的内容
                    "description": "Enable pretty-printing for lldb", 
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": false
                }
            ],
            "preLaunchTask": "build_debug" // 预构建任务名称
        },
        {
            "name": "RunRelease",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/bin/release/bin_name", // 可执行文件路径
            "args": [
                "arg1",
                "arg2"
            ], // 命令行参数
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MiMode": "lldb", // or gdb
            "miDebuggerPath": "/usr/bin/lldb", // 调试器路径(/usr/bin/lldb or /usr/bin/gdb)
            "setupCommands": [
                { // 模板自带，好像可以更好地显示STL容器的内容
                    "description": "Enable pretty-printing for lldb", 
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": false
                }
            ],
            "preLaunchTask": "build_release" // 预构建任务名称
        }
    ]
}
```

`tasks.json` 示例：

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build_debug",
            "type": "shell",
            "command": "bash",
            "args": [
                "-c",
                "cd build && cmake -DCMAKE_BUILD_TYPE=DEBUG .. && make -j16"
            ],
            "group": "build",
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "shared"
            }
        },
        {
            "label": "build_release",
            "type": "shell",
            "command": "bash",
            "args": [
                "-c",
                "cd build_release && cmake -DCMAKE_BUILD_TYPE=RELEASE .. && make -j16"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "shared"
            }
        }
    ]
}
```

`c_cpp_properties.json` 示例：

```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "${workspaceFolder}/include/**",
                "/usr/include/**",
                "/usr/local/include/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/clang",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64",
            "compileCommands": "${workspaceFolder}/build/compile_commands.json",
            "configurationProvider": "ms-vscode.cmake-tools"
        }
    ],
    "version": 4
}
```

`settings.json`示例：

```json
{
    // Configure glob patterns for excluding files and folders. For example, the File Explorer decides which files and 
    // folders to show or hide based on this setting. Refer to the `search.exclude` setting to define search-specific excludes.
    "files.exclude": {
      "**/.git": true,
      "**/.svn": true,
      "**/.hg": true,
      "**/CVS": true,
      "**/.DS_Store": true,
      "**/Thumbs.db": true
    },
    // Configure paths or glob patterns to exclude from file watching.
    "files.watcherExclude": {
      "**/.git/objects/**": true,
      "**/.git/subtree-cache/**": true,
      "**/node_modules/*/**": true,
      "**/.hg/store/**": true
    },
    // Configure glob patterns for excluding files and folders in fulltext searches and quick open.
    // Inherits all glob patterns from the `files.exclude` setting.
    "search.exclude": {
      "**/node_modules": true,
      "**/bower_components": true,
      "**/*.code-search": true
    },

}
```

`user settings.json`：

```json
{
    "files.autoSave": "afterDelay",

    "editor.inlineSuggest.enabled": true,
    "editor.fontSize": 16,
    "editor.fontFamily": "JetBrains Mono, Source Han Serif",
    "editor.guides.bracketPairs": true,
    "editor.guides.indentation": false,
    "editor.guides.highlightActiveIndentation": false,
    "editor.renderLineHighlight": "none",
    "editor.renderWhitespace": "all",
    "editor.rulers": [
        120
    ],
    "editor.wordWrap": "on",
    "editor.wordWrapColumn": 120,
    "editor.wordBasedSuggestions": "off",
    "editor.selectionHighlight": true,
    "editor.minimap.showSlider": "always",
    "editor.autoIndent": "full",
    "editor.accessibilitySupport": "off",
    "editor.autoClosingBrackets": "always",
    "editor.autoClosingDelete": "always",
    "editor.autoClosingQuotes": "always",
    "editor.autoClosingOvertype": "always",
    "editor.minimap.enabled": false,
    
    "diffEditor.ignoreTrimWhitespace": true,
    "scm.diffDecorationsIgnoreTrimWhitespace": "true",
    
    "workbench.activityBar.location": "top",
    "workbench.tree.enableStickyScroll": true,
    "workbench.colorTheme": "Dracula",
    "workbench.iconTheme": "material-icon-theme",
    "workbench.startupEditor": "none",
    
    "security.workspace.trust.untrustedFiles": "open",
    
    "terminal.integrated.fontFamily": "Consolas",
    "terminal.integrated.fontSize": 16,
    "terminal.integrated.fontWeight": "bold",
    
    "github.copilot.enable": {
        "*": true,
        "plaintext": false,
        "markdown": true,
        "scminput": false
    },
    
    "cmake.configureOnOpen": true,
    "cmake.showOptionsMovedNotification": false,

    "C_Cpp.autocompleteAddParentheses": true,
    "C_Cpp.formatting": "clangFormat",
    "C_Cpp.vcFormat.newLine.beforeElse": false,
    "C_Cpp.vcFormat.newLine.beforeOpenBrace.function": "ignore",
    "C_Cpp.clang_format_sortIncludes": true,
    "C_Cpp.intelliSenseEngine": "disabled",
    
    // "[markdown]": {
    //     "editor.defaultFormatter": "DavidAnson.vscode-markdownlint"
    // },
    
    "remote.SSH.remotePlatform": {
        "server.natappfree.cc": "linux"
    },
    
    "clang-format.executable": "/usr/bin/clang-format-12",
    "clang-format.style": "file",

    // code-runnerq
    "code-runner.runInTerminal": true,
    "code-runner.saveFileBeforeRun": true, // run code前保存
    "code-runner.clearPreviousOutput": true, // 每次run code前清空属于code runner的终端消息，默认false
    "clangd.arguments": [
        "--compile-commands-dir=${workspaceFolder}/build",//指定配置文件compelie_commands.json所在目录，这里有三种方法生成
        // 在后台自动分析文件（基于complie_commands)
        "--background-index",
        // 同时开启的任务数量
        "-j=12",
        // "--folding-ranges"
        // 告诉clangd用那个clang进行编译，路径参考which clang++的路径
        "--query-driver=/usr/bin/clang++",
        // clang-tidy功能
        "--clang-tidy",
        "--clang-tidy-checks=performance-*,bugprone-*",
        // 全局补全（会自动补充头文件）
        "--all-scopes-completion",
        // 更详细的补全内容
        "--completion-style=detailed",
        "--function-arg-placeholders",
        // 补充头文件的形式
        "--header-insertion=iwyu",
        // pch优化的位置
        "--pch-storage=memory",
    ],

    "latex-workshop.latex.autoBuild.run": "never",
    "latex-workshop.showContextMenu": true,
    "latex-workshop.intellisense.package.enabled": true,
    "latex-workshop.message.error.show": false,
    "latex-workshop.message.warning.show": false,
    "latex-workshop.latex.tools": [
        {
            "name": "xelatex",
            "command": "xelatex",
            "args": [
                "-synctex=1",
                "-interaction=nonstopmode",
                "-file-line-error",
                "-pdf",
                "%DOC%"
            ]
        },
        {
            "name": "pdflatex",
            "command": "pdflatex",
            "args": [
                "-synctex=1",
                "-interaction=nonstopmode",
                "-file-line-error",
                "%DOC%"
            ]
        },
        {
            "name": "latexmk",
            "command": "latexmk",
            "args": [
                "-synctex=1",
                "-interaction=nonstopmode",
                "-file-line-error",
                "-pdf",
                "-outdir=%OUTDIR%",
                "%DOC%"
            ]
        },
        {
            "name": "bibtex",
            "command": "bibtex",
            "args": [
                "%DOCFILE%"
            ]
        }
    ],
    "latex-workshop.latex.recipes": [
        {
            "name": "XeLaTeX",
            "tools": [
                "xelatex"
            ]
        },
        {
            "name": "PDFLaTeX",
            "tools": [
                "pdflatex"
            ]
        },
        {
            "name": "pdf->pdf",
            "tools": [
                "pdflatex",
                "pdflatex"
            ]
        },
        {
            "name": "xe->xe",
            "tools": [
                "xelatex",
                "xelatex"
            ]
        },
        {
            "name": "BibTeX",
            "tools": [
                "bibtex"
            ]
        },
        {
            "name": "LaTeXmk",
            "tools": [
                "latexmk"
            ]
        },
        {
            "name": "xe->bib->xe->xe",
            "tools": [
                "xelatex",
                "bibtex",
                "xelatex",
                "xelatex"
            ]
        },
        {
            "name": "pdf->bib->pdf->pdf",
            "tools": [
                "pdflatex",
                "bibtex",
                "pdflatex",
                "pdflatex"
            ]
        }
    ],
    
    "latex-workshop.latex.clean.fileTypes": [
        "*.aux",
        "*.bbl",
        "*.blg",
        "*.idx",
        "*.ind",
        "*.lof",
        "*.lot",
        "*.out",
        "*.toc",
        "*.acn",
        "*.acr",
        "*.alg",
        "*.glg",
        "*.glo",
        "*.gls",
        "*.ist",
        "*.fls",
        "*.log",
        "*.fdb_latexmk"
    ],
    "latex-workshop.latex.autoClean.run": "onSucceeded",
    "latex-workshop.latex.recipe.default": "lastUsed",
    "latex-workshop.view.pdf.viewer": "tab",
    "latex-workshop.view.pdf.internal.synctex.keybinding": "double-click",
    
    "[cpp]": {
        "editor.defaultFormatter": "xaver.clang-format"
    },
}
```
