# DevNet Marathon - Topology Visualization
Вариант реализации задания на визуализацию сетевых топологий для Cisco DevNet Марафона.

### Используемый стек:
  - Python3
  - [Nornir](https://nornir.readthedocs.io/en/latest/)
  - [NAPALM](https://napalm.readthedocs.io/en/latest/)
  - [NeXt UI](https://developer.cisco.com/site/neXt/) (JS+HTML5)

### Установка и первичная настройка:
Установите зависимости:
```sh
$ mkdir ~/devnet_marathon_endgame
$ cd ~/devnet_marathon_endgame
$ git clone https://github.com/iDebugAll/devnet_marathon_endgame.git
$ cd devnet_marathon_endgame
$ pip3 install -r requirements.txt
```
Отредактируйте конфигурационные файлы Nornir для доступа на целевую сетевую инфраструктуру.
По умолчанию используется модуль SimpleInventory.
В файле nornir_config.yml необходимо указать используемые файлы хостов и групп.
Файл хостов, используемый по умолчанию, настроен на работу по SSH с сетевой топологией из [Cisco Modeling Labs](https://devnetsandbox.cisco.com/RM/Diagram/Index/685f774a-a5d6-4df5-a324-3774217d0e6b?diagramType=Topology) в Cisco Devnet Sandbox.
Для подключения к Cisco DevNet Sandbox требуется бесплатная регистрация, резервирование и VPN-доступ.

### Использование:
Для синхронизации топологии необходимо запустить скрипт generate_topology.py.
После того, как скрипт завершит работу, необходимо открыть файл main.html.
```
$ python3.7 generate_topology.py 
Изменений в топологии не обнаружено.
Для просмотра топологии откройте файл main.html
```

![sample_topology](/samples/sample_topology.png)

Для отрисовки топологий используется 'force' процессор данных из NeXt UI.
Его внутренний алгоритм стремится расположить ноды таким образом, чтобы расстояние между соседями было примерно одинакомым.
В силу этой логики для комплексных топологий расположение уровней и результата в целом может быть повернутым в горизонтальной плоскости относительно желаемого. Для более точной расстановки реализована возможность выравнивания по уровням топологии, для этого в файле хостов для каждогого из хостов нужно определить роль (более детальное описание будет добавлено ниже).
Раскладка генерируется заново при каждом перезапуске страницы.

### Возможности и описание работы:

Решение является [кросплатформенным](https://napalm.readthedocs.io/en/latest/support/), интегрируемым и расширяемым.
Скрипт протестирован на IOS, IOS-XE и NX-OS.

Для сбора данных с сетевых устройств используется связка NAPALM+Nornir:
  - NAPALM геттер GET_LLDP_NEIGHBORS_DETAILS (сбор LLDP-соседств);
  - NAPALM геттер GET_FACTS (общие данные об устройстве для вывода в подсказке).

На вход принимается инициализированный инвентори Nornir с:
  - IP-адресами устройств;
  - Аутентификационными данными для этих устройств с правами на чтение.

На выходе генерируется:
  - Файл topology.js c JS объектом топологии для NeXt UI;
  - Файл cached_topology.json с JSON-представлением проанализированной топологии.
  - Файл diff_topology.js с визуализацией изменений в текущей топологии
    относительно последнего известного cached_topology.json.
  - Консольный вывод с результатами проверки на наличие изменений в топологии.

Файл topology.js перезаписывается при каждом запуске. Версия в репозитории для ознакомления содержит результат обработки топологии из [Cisco Modeling Labs](https://devnetsandbox.cisco.com/RM/Diagram/Index/685f774a-a5d6-4df5-a324-3774217d0e6b?diagramType=Topology) в Cisco DevNet Sandbox.

Реализована базовая обработка возможных ошибок и нормализация данных.
Попытка сбора данных по умолчанию осуществляется со всех устройст в инвентори.
В случае ошибок в получении данных, отсутствия LLDP соседств, прав и т.п. на
каком-либо из хостов, он включается в топологию с именем из инвентори без связей.

Поддерживается сценарий с отсутствующими LLDP-соседствами (internet-rtr01.virl.info на примере выше).
Поддерживается сценарий с отсутствием доступа на одиночные промежуточные устройства (core-rtr01.devnet.lab и core-rtr02.devnet.lab на примере выше) при условии запущенного на них LLDP.

Типы пиктограмм выбираются автоматически по следующей логике в порядке убывания приоритета:
  - На основании LLDP capabilities, полученных от соседей.
  - На основании модели устройства, если с этого устройства собирались данные, и модель имеется в выводе GET_FACTS.
  - Значение по умолчанию 'unknown' (пиктограмма вопросительного знака).

Для визуализации используется набор инструментов NeXt UI. Получаемая в результате страница main.html содержит интерактивную форму с визуализацией проанализированного участка сети.

Пиктограммы устройств могут перемещаться мышью в произвольном направлении с адаптивной отрисовкой линков. Поддерживается выделение и групповое перемещение.
При наведении указателя на элемент топологии автоматически затемняются все несвязанные с ним ноды и линки.

По левому клику мыши на устройство выводится меню со справочной информацией:
![node_details](/samples/sample_node_details.png)

Серийный номер и модель указываются для устройств на основании вывода GET_FACTS там, где это возможно.
Для устройств, на которые имеется доступ, может быть добавлена произвольная информация.
Аналогично для линков:

![node_details](/samples/sample_link_details.png)

При каждом запуске скрипта generate_topology.py выполняется анализ изменений топологии.
Полученные с устройств детали топологии записываются, помимо прочего, в файл cached_topology.json
Данный файл считывается при старте и сравнивается с текущим полученным от устройств состоянием.
В результате данные о добавленных и удаленных в топологии устройствах и линках выводится в консоль, а в файл diff_topology.js записывается объединенная версия двух топологий с расширенными атрибутами для визуализации изменений. Файл с визуализированными изменениями топологии можно посмотреть, открыв файл diff_page.html.

Вывод из консоли:

```
$ python3.7 generate_topology.py 
Для просмотра топологии откройте файл main.html

Обнаружены изменения в топологии:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Новые сетевые устройства:
vvvvvvvvvvvvvvvvvvvvvvvvvvvvv
Имя устройства: dist-rtr01.devnet.lab

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Удаленные сетевые устройства:
vvvvvvvvvvvvvvvvvvvvvvvvvvvvv
Имя устройства: edge-sw01.devnet.lab

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Новые соединения между устройствами:
vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
От dist-rtr01.devnet.lab(Gi3) к core-rtr02.devnet.lab(Gi0/0/0/2)
От dist-rtr01.devnet.lab(Gi4) к dist-sw01.devnet.lab(Eth1/3)
От dist-rtr01.devnet.lab(Gi6) к dist-rtr02.devnet.lab(Gi6)
От dist-rtr01.devnet.lab(Gi5) к dist-sw02.devnet.lab(Eth1/3)
От dist-rtr01.devnet.lab(Gi2) к core-rtr01.devnet.lab(Gi0/0/0/2)

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Удаленные соединения между устройствами:
vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
От edge-sw01.devnet.lab(Gi0/2) к core-rtr01.devnet.lab(Gi0/0/0/1)
От edge-sw01.devnet.lab(Gi0/3) к core-rtr02.devnet.lab(Gi0/0/0/1)

Для просмотра топологии с визуализацией изменений откройте файл diff_page.html
Либо откройте файл main.html и нажмите кнопку 'Показать визуализацию изменений
```

И отображение файла diff.html:

![sample_diff](/samples/sample_diff.png)

Отображение удаленных и новых элементов интуитивно понятно. :)


### Экспериментальные и дополнительные возможности:

В решение добавлена возможность автоматического выравнивания хостов с разбиением по уровням по горизонтали и вертикали.
По умолчанию топология формируется стандартным обработчиком. Для выравнивания в интерфейсе main.html и diff_page.html имеются соответствующие кнопки.

Хосты выравниваются на основании их роли в топологии, полученной из инвентори Nornir.
Ноды каждого уровня будут находиться на одной линии. Уровни следуют друго за другом по горизонтали или вертикали.
Иерархия уровней хранится в переменной NX_LAYER_SORT_ORDER в файле generate_topology.py:

```python
NX_LAYER_SORT_ORDER = (
    'undefined',
    'outside',
    'edge-switch',
    'edge-router',
    'core-router',
    'core-switch',
    'distribution-router',
    'distribution-switch',
    'leaf',
    'spine',
    'access-switch'
)
```

Для того, чтобы это начало работать, в файле хостов нужно указать для оборудования атрибут 'role' со значением из списка выше (либо расширить список):

```yaml
dist-rtr02:
    hostname: 10.10.20.176
    platform: ios
    groups:
        - devnet-cml-lab
    data:
        role: distribution-router
```

Если в инвентори отсутствует данный атрибут для хоста, значением по умолчанию будет 'undefined'.
Репозиторий содержит пример сгенерированной топологии и пример инвентори.

Пример автоматического выравнивания по горизонтали:
![sample_layout_horizontal](/samples/sample_layout_horizontal.png)
