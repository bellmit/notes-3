```
"编码
:set encoding=utf-8
:set fileencoding=utf-8
:set termencoding=utf-8

" 语法高亮
if has("syntax")
    syntax on
endif

:set ruler "在状态栏显示光标的当前位置（位于哪一行哪一列）。
:set laststatus=2 "是否显示状态栏。0 表示不显示，1 表示只在多窗口时显示，2 表示显示。
:set cursorline "光标所在的当前行高亮
:set t_Co=256 "启用256色
:set showmode "在底部显示，当前处于命令模式还是插入模式
:set autoindent "按下回车键后，下一行的缩进会自动跟上一行的缩进保持一致
:set smartindent "智能对齐方式
:set tabstop=4 "一个tab是4个字符
:set softtabstop=4 "Tab转为多少个空格
:set expandtab "用空格代替tab
:set mouse=a "使用鼠标
:set number "显示行号
:set showmatch "光标遇到圆括号、方括号、大括号时，自动高亮对应的另一个圆括号、方括号和大括号。
:set hlsearch "搜索时，高亮显示匹配结果。
:set nobackup "不创建备份文件。默认情况下，文件保存时，会额外创建一个备份文件，它的文件名是在原文件名的末尾，再添加一个波浪号（〜）。
:set noswapfile "不创建交换文件。交换文件主要用于系统崩溃时恢复文件，文件名的开头是.、结尾是.swp。
:set undofile "保留撤销历史。
:set backupdir=~/.vim/.backup// "备份文件，需要手动创建文件夹
:set directory=~/.vim/.swp// "交换文件，需要手动创建文件夹
:set undodir=~/.vim/.undo// "操作历史文件，需要手动创建文件夹
:set history=1000 "Vim 需要记住多少次历史操作。
:set autoread "打开文件监视。如果在编辑过程中文件发生外部改变（比如被别的编辑器编辑了），就会发出提示。
```

