## yaml plugin

```sh
git clone --depth 1 https://github.com/dense-analysis/ale.git ~/.vim/pack/git-plugins/start/ale
git clone https://github.com/Yggdroot/indentLine.git ~/.vim/pack/vendor/start/indentLine
```

`.vimrc`

```text
set mouse-=a
syntax on
set tabstop=8
set expandtab
set shiftwidth=2
set softtabstop=2
au BufNewFile,BufRead *.conf setf cfg

autocmd FileType yaml setlocal ts=2 sts=2 sw=2 expandtab
autocmd FileType yml setlocal ts=2 sts=2 sw=2 expandtab
let g:ale_echo_msg_format = '[%linter%] %s [%severity%]'
let g:ale_sign_error = 'x'
let g:ale_sign_warning = '!'
let g:ale_lint_on_text_changed = 'never'
```

Установлен yamllint. Добавить в `.config/yamllint/config`
```text
extends: relaxed

rules:
  line-length: disable
```

## кнопки для редактирования

`2>>` сделать отступ из 2 пробелов (так в .vimrc) для 2 строк

`2dd` удалить 2 строки

`2Y` скопировать 2 строки

`p` вставить удаленные или скопированные 4 строки ниже курсора
