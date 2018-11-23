# VSCode 導入手順

- 経緯 : VSCode を使いたい！！
- 目的 : VSCode でソースを編集できるようにする
- 状況 : リモートの CoreOS で実行環境を動かしている
- 決定 : ローカルにソースを展開して、リモートにコピー


###### Table of Contents

- [構成](#user-content-構成)
- [手順](#user-content-手順)


### 構成

- mac → sftp → CoreOS
- mac に実行環境をデプロイしたくない(shell script も含む)
- コマンドは ssh 経由で実行する


### 手順

1. sftp をインストール
1. プロジェクトを git clone
1. sftp config 設置


#### sftp config

```json
{
    "protocol": "sftp",
    "host": "192.168.0.109",
    "port": 22,
    "username": "USER",
    "remotePath": "/home/USER/volumes/apps/path/to/app",
    "privateKeyPath": "/path/to/.ssh/id_rsa",
    "ignore": [
        ".vscode",
        ".git",
        ".DS_Store"
    ],
    "syncMode": "full",
    "uploadOnSave": true,
    "watcher": {
        "files": "**/*",
        "autoUpload": false,
        "autoDelete": false
    }
}
```

#### vscode setting

```json
{
  "workbench.colorTheme": "Solarized Light",
  "editor.fontFamily": "'APJapanesefontT', Menlo, Monaco, 'Courier New', monospace",
  "editor.fontSize": 16,
  "editor.lineHeight": 16,
  "explorer.confirmDelete": false,
  "editor.minimap.enabled": true,
  "elm.makeCommand": null,
  "vim.leader": "<space>",
  "vim.normalModeKeyBindingsNonRecursive": [
    {
        "before": ["<leader>","s"],
        "commands": ["workbench.action.files.save"]
    },
    {
        "before": ["<leader>","w"],
        "commands": ["workbench.action.closeActiveEditor"]
    },
    {
        "before": ["<leader>","q"],
        "commands": ["workbench.action.closeFolder"]
    },
    {
        "before": ["u"],
        "commands": ["undo"]
    },
    {
        "before": [";"],
        "commands": ["workbench.action.showCommands"]
    },
    {
        "before": ["<leader>","n"],
        "commands": ["workbench.action.nextEditor"]
    },
    {
        "before": ["<leader>","h"],
        "commands": ["workbench.action.previousEditor"]
    },
    {
        "before": ["<leader>","t"],
        "commands": ["workbench.action.terminal.new"]
    },
    {
        "before": ["<leader>","a"],
        "commands": ["git.stageAll"]
    },
    {
        "before": ["<leader>","c"],
        "commands": ["git.commitStaged"]
    },
    {
        "before": ["<leader>","b"],
        "commands": ["git.branch"]
    },
    {
        "before": ["<leader>","j"],
        "commands": ["git.checkout"]
    },
    {
        "before": ["<leader>","f"],
        "commands": ["git.pull"]
    },
    {
        "before": ["<leader>","d"],
        "commands": ["git.deleteBranch"]
    },
    {
        "before": ["<leader>","p"],
        "commands": ["git.push"]
    },
    {
        "before": ["<leader>","u"],
        "commands": [
            "git.fetch",
            "git.checkout",
            "git.pull",
            "git.deleteBranch"
        ]
    }
  ]
}
```
