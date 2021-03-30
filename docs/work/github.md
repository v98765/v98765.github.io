Один раз [создать токен](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token)

Создать репу, будут подсказки
```sh
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/v98765/v98765_exporter_blackbox.git
git push -u origin main
```
Ввести имя пользователя и токен/пароль

Для сохранения учетной записи перед push выполнить:
```sh
git config credential.helper store
```
В домашнем каталоге будет файл `.git-credentials`
