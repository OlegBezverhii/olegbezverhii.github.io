---
layout: post
title:  Battlefield 3 в 2025 году
---

## Косяки и методы их решения

В общем, решили друзья поиграть в старый шедевр, но у нас произошло несколько проблем.

1. Руссификация

Решение:

Заходим в реестр (пишем regedit в поиске винды), далее идем по пути [HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\EA Games\Battlefield 3] . Если не получается найти, то по пути [HKEY_LOCAL_MACHINE \SOFTWARE\EA Games\Battlefield 3]

Меняем значение поля GDFBinary на GDFBinary_ru_RU.dll
Меняем значение поля Locale на ru_RU

Перезапускаем EA Play и играем.

2. Не дает запуститься, пишет не активирована игра на этом ПК.

[Решение](https://steamcommunity.com/sharedfiles/filedetails/?id=3008106492):

Мне помогло удалить файлы с папки лицензии C:\ProgramData\Electronic Arts\EA Services\License. Удалял не всю папку, а старые по дате, потому что в папке лежали лицензии от BF4/BF1/BF5.


3. Выкидывает с сервера по PunkBuster'у.

Ошибка вида "Error loading pbag". Если запустить инсталятор из папки с игрой, добавить игру, то на этапе скачивания файлов выходит ошибка. Благо добрые люди на [forums.ea.com](https://forums.ea.com/discussions/battlefield-franchise-discussion-ru/не-обновляется-punkbuster-bf3/12260941) выложили архивчик с последними файлами. По пути `C:\Users\имя пользователя\AppData\Local\PunkBuster` переносим файлы и все должно заработать.

