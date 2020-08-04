Необходимо было из csv файла взять текст и вписать его в дескрипш маршрутизатора.
Кодировку переделать, русский в транслит и убрать спецсимволы.

Утилитой recode изменить кодировку текста на utf8 в файле data_rus.csv
```text
recode windows-1251..utf8 data_rus.csv
```
Утилитой [translit](http://lingua-systems.com/translit/) воспользоваться для перевода текста в транслит.
```text
translit -t "GOST 7.79 RUS" -i data_rus.csv -o data_translit.csv
```
Удалить спецсимволы
```text
cat data_translit.csv | tr -d "'`"
```



