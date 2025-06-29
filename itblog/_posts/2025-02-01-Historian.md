---
layout: post
title: Полезные программы для работы с GE Proficy Historian
---

## Мои проекты

Я думаю многие пользователи GE Proficy Historian столкнулись с проблемами, когда базовые задачи не покрывает стандартная утилита Historian Administration. Поэтому для решения проблем и примера использования фирменного SDK я написал две небольшие программы.

[https://github.com/OlegBezverhii/AddGEHistorianArhives](https://github.com/OlegBezverhii/AddGEHistorianArhives) - программа для массового добавления архивов из файлов в GE Proficy Historian. Иногда после сбоев Historian теряет весь список файлов, приходится вручную добавлять их через Historian Administration, который имеет ограничение не более 15 файлов за один выбор. Моя программа пробежится по указанной вами папке и добавить их в хранилище.

[https://github.com/OlegBezverhii/BackuperGEArchives](https://github.com/OlegBezverhii/BackuperGEArchives) - программа для периодического бэкапа самого старого архива.

Надеюсь, это будет кому-то полезно.