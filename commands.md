# Команды jira-cli

```
jira <command> <subcommand>
```

## Настройка и служебные

| Команда | Алиасы | Описание |
|---|---|---|
| `jira init` | `initialize`, `configure`, `config`, `setup` | Интерактивный мастер генерации конфига (`~/.config/.jira/.config.yml`) |
| `jira version` | — | Версия и сборка (Git commit, дата, Go-версия) |
| `jira completion` | — | Генерация shell-автодополнения (bash/zsh) |
| `jira man` | — | Генерация man-страниц |

## Issue (задачи)

| Команда | Алиасы | Описание |
|---|---|---|
| `jira issue list [text]` | `ls`, `search` | Список задач с фильтрами (тип, статус, приоритет, лейблы, даты, JQL) |
| `jira issue create` | — | Создание задачи (интерактивно или через флаги) |
| `jira issue edit ISSUE-KEY` | `update`, `modify` | Редактирование задачи |
| `jira issue view ISSUE-KEY` | `show` | Просмотр содержимого задачи |
| `jira issue assign ISSUE-KEY USER` | `asg` | Назначить исполнителя |
| `jira issue move ISSUE-KEY STATE` | `transition`, `mv` | Перевести задачу в другой статус |
| `jira issue clone ISSUE-KEY` | — | Клонировать задачу (с линком "Cloners") |
| `jira issue delete ISSUE-KEY` | `rm`, `del` | Удалить задачу |
| `jira issue link A B TYPE` | `ln` | Связать две задачи (link type) |
| `jira issue unlink A B` | `uln` | Разорвать связь между задачами |
| `jira issue link remote KEY URL TITLE` | `rmln` | Добавить веб-ссылку на задачу |
| `jira issue comment add KEY [BODY]` | — | Добавить комментарий |
| `jira issue watch KEY USER` | `wat` | Добавить наблюдателя |
| `jira issue worklog add KEY TIME` | — | Добавить запись трудозатрат |

## Epic (эпики)

| Команда | Алиасы | Описание |
|---|---|---|
| `jira epic list [EPIC-KEY]` | `ls` | Список эпиков (или задач внутри эпика) |
| `jira epic create` | — | Создать эпик |
| `jira epic add EPIC-KEY ISSUES...` | `assign` | Добавить задачи в эпик |
| `jira epic remove ISSUES...` | `rm`, `unassign` | Убрать задачи из эпика |

## Sprint (спринты)

| Команда | Алиасы | Описание |
|---|---|---|
| `jira sprint list [SPRINT_ID]` | `ls` | Список спринтов на доске (или задач в спринте) |
| `jira sprint add SPRINT_ID ISSUES...` | `assign` | Добавить задачи в спринт |
| `jira sprint close SPRINT_ID` | `complete` | Закрыть спринт |

## Board / Project / Release

| Команда | Алиасы | Описание |
|---|---|---|
| `jira board list` | `ls` | Список досок в проекте |
| `jira project list` | `ls` | Список проектов Jira |
| `jira release list` | `ls` | Список версий (релизов) проекта |

## Прочее

| Команда | Алиасы | Описание |
|---|---|---|
| `jira open [ISSUE-KEY]` | `browse`, `navigate` | Открыть задачу в браузере |
| `jira me` | — | Показать текущего пользователя |
| `jira serverinfo` | `systeminfo` | Информация о Jira-инстансе |

## Полезные глобальные флаги (на любой команде)

- `-p, --project` — переопределить проект из конфига
- `-c, --config` — путь к файлу конфига
- `--debug` — отладочный вывод (дамп HTTP-запрос/ответ)

## Полезные флаги списков (`issue/epic/sprint list`)

- `--plain`, `--raw`, `--csv` — режим вывода (по умолчанию интерактивная TUI)
- `-t` тип, `-s` статус, `-y` приоритет, `-r` reporter, `-a` assignee, `-l` label
- `--created today|week|month|-10d` — датные фильтры
- `-q "JQL"` — произвольный JQL-запрос
- `--paginate from:limit` — постраничная выдача
- `--columns key,summary,status` — выбор колонок в plain-режиме
