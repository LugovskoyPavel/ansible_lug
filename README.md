# Ansible плейбук предназначен для установки Clickhouse и Vector, а также Lighthouse  
## Описание файлов плейбука

### group_vars

Описание переменных.

| Имя переменной | Описание |
| ------------- |:------------------:|
| clickhouse_version | версия Clickhouse |
| clickhouse_packages | пакеты Clickhouse, которые необходимо скачать |
| vector_url | URL адрес для скачивания RPM пакетов Vector |
| vector_version | версия Vector |
| vector_home | каталог для скачивания RPM пакетов Vector |
| lighthouse_url |	ссылка на репозиторий Lighthouse |
| lighthouse_dir |	каталог для файлов Lighthouse |
| lighthouse_nginx_user |	пользователь, из-под которого будет работать Nginx. Для упрощения процесса на тестовом стенде используем root пользователя |

### Inventory файл
1. Группа хостов "clickhouse1" состоит из 1 хоста clickhouse1
2. Группа хостов "vector1" состоит из 1 хоста vector1
3. Группа хостов "lighthouse1" состоит из 1 хоста lighthouse1

### Playbook
Playbook состоит из 3 плей.

1. Плей "Install Clickhouse" - установка clickhouse на группу хостов "clickhouse1"

Объявляем handler для запуска clickhouse-server.

handlers:
    - name: Start clickhouse service
      become: true
      ansible.builtin.service:
        name: clickhouse-server
        state: restarted
|Имя таска	| Описание |
| ------------- |:------------------:|
| Clickhouse  Get clickhouse distrib	| Скачивание RPM пакетов. Используется цикл с перменными clickhouse_packages. Так как не у всех пакетов есть noarch версии, используем перехват ошибки rescue |
| Clickhouse Install clickhouse packages |	Установка RPM пакетов. Используем disable_gpg_check: true для отключения проверки GPG подписи пакетов. В notify указываем, что данный таск требует запуск handler Start clickhouse service |
| Clickhouse Flush handlers |	Форсируем применение handler Start clickhouse service. Это необходимо для того, чтобы handler выполнился на текущем этапе, а не по завершению тасок. Если его не запустить сейчас, то сервис не будет запущен и следующий таск завершится с ошибкой |
| Clickhouse Create database |	Создаем в Clickhouse БД с названием "logs". Также прописываем условия, при которых таск будет иметь состояние failed и changed |

2. Плей "Install Vector" применяется на группу хостов "vector1"

Объявляем handler для запуска vector.

  handlers:
    - name: Start Vector service
      become: true
      ansible.builtin.service:
        name: vector
        state: restarted
|Имя таска	| Описание |
| ------------- |:------------------:|
| Vector download packages |	Скачивание RPM пакетов в текущую директорию пользователя |
| Vector install packages |	Установка RPM пакетов. Используем disable_gpg_check: true для отключения проверки GPG подписи пакетов |
| Vector apply template |	Применяем шаблон конфига vector. Здесь мы задаем путь конфига. Владельцем назначаем текущего пользователя ansible. После применения запускаем валидацию конфига |
| Vector change systemd unit	| Изменяем модуль службы vector. После этого указываем handler для старта службы vector |

3. Плей "Install Lighthouse" применяется на группу хостов "lighthouse1"

Объявляем handler для запуска lighthouse

handlers:
    - name: Nginx reload
      become: true
      ansible.builtin.service:
        name: nginx
        state: restarted
        
| Имя претаск |	Описание |
| ------------- |:------------------:|
| Lighthouse Install git |	Установка git |
| Lighhouse Install nginx |	Установка Nginx |
| Lighthouse Apply nginx config |	Применяем конфиг Nginx |


| Имя таска	| Описание |
| ------------- |:------------------:|
| Lighthouse  Clone repository |	Клонируем репозиторий lighthouse из ветки master |
| Lighthouse  Apply config |	Применяем конфиг Nginx для lighthouse. После этого перезапускаем nginx для применения изменений |


### Template

1. Шаблон "vector.service.j2" используется для изменения модуля службы vector. В нем мы определяем строку запуска vector. Также указываем, что unit должен быть запущен под текущим пользователем ansible

2. Шаблон "vector.yml.j2" используется для настройки конфига vector. В нем мы указываем, что конфиг файл находится в переменной "vector_config" и его надо преобразовать в YAML.

3. Шаблон "nginx.conf.j2" используется для первичной настройки nginx. Мы задаем пользователя для работы nginx и удаляем настройки root директории по умолчанию.

4. Шаблон "lighthouse_nginx.conf.j2" настраивает nginx на работу с lighthouse.
