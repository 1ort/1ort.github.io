---
title: "Install_hugo"
date: 2023-08-12T05:22:52+05:00
draft: true
---

### Итак, у меня есть сайт на Hugo и Github pages.
Небольшой гайд как сделать что-то похожее.
## Setup
1. Идём на https://gohugo.io/installation/, выбираем ОС и вариант установки, ставим по инструкции.
2. Создаём проект
```sh
hugo new site ort.soy
cd ort.soy
git init
```
3. Ставим тему. Я выбрал risotto

```sh
git submodule add https://github.com/joeroe/risotto.git
echo "theme = 'risotto'" >> hugo.toml
```
4. Запускаем сайт локально и проверяем http://localhost:1313/.
	Чтобы завершить  - CONTROL + C

```
hugo server
```

## Создаём контент
Теперь нам нужно создать пост. Вызываем 
```sh
hugo new hello-world.md
```

Hugo создаст md-файл со своей метадатой:

```
---
title: "Hello World"
date: 2023-08-12T05:38:18+05:00
draft: true
---


```

Уже в следующей строчке мы можем писать любой markdown-контент.
Делать этго удобно в markdown-редакторе.
Я предпочитаю obsidian. Нужно только добавить `.obsidian` в `.gitignore` 
Дока: https://gohugo.io/content-management/

`draft: true` в метадате не позволит Hugo рендерить пост, т.к. это черновик.

Используем `hugo server -D` чтобы локально поднять сервер с черновиками
Используем `hugo` чтобы отрендерить весь сайт перед пушем в репозиторий

##  Хостим
Нам нужен репозиторий на github, который будет называться так:
```
{username}.github.io
```
где username - ваш ник на github.

После этого нам нужно подключить локальный репозиторий к удалённому.
Как настроить ключики в github писать не буду.

