Концепция потоков NiFi
=======================

Ключевые концепции **NiFi** тесно связаны с основными идеями потокового программирования (`Flow Based Programming -- FBP <https://en.wikipedia.org/wiki/Flow-based_programming#Concepts>`_). Далее в таблице приведены некоторые из основных концепций **NiFi** и их сопоставление с **FBP**.

.. csv-table:: Сопоставление концепций NiFi и FBP
   :header: "NiFi Term", "FBP Term", "Описание"
   :widths: 25, 25, 50

   "FlowFile", "Information Packet", "FlowFile представляет каждый объект, перемещающийся через систему, и для каждого из них NiFi отслеживает список атрибутов (пары ключ/значение) и связанного с ним содержимого с количеством байт от нуля и больше"
   "FlowFile Processor", "Black Box", "Процессоры, выполняющие задание. В терминах `eip <https://www.enterpriseintegrationpatterns.com/>`_ процессор выполняет некоторую комбинацию маршрутизации, преобразования и обмена информацией между системами. Процессоры имеют доступ к атрибутам предоставленного FlowFile и его контенту. Процессоры могут работать с ноль и более FlowFiles в данном блоке задания, либо выполнять работу (commit), либо отменять результат ее выполнения откатом (rollback)"
   "Connection", "Bounded Buffer", "Соединения выступают в роли фактической связи между процессорами. Они действуют как очереди и позволяют разным процессам взаимодействовать с разной скоростью. Очереди могут быть распределены по приоритетам и иметь верхние границы нагрузки, инициализирующие процесс *back pressure*"
   "Flow Controller", "Scheduler", "Контроллер потока хранит знание о том, как соединены процессы, управляет и распределяет потоки, которые используются процессами. Flow Controller выступает в качестве брокера, облегчающего обмен FlowFiles между процессорами"
   "Process Group", "subnet", "Группа процессов представляет собой определенный набор процессоров и их соединений, которые могут принимать данные через порты для входящих соединений и отправлять их через выходные порты. Таким образом, группы процессов позволяют создавать совершенно новые компоненты просто посредством композиции других компонентов"

Такая модель проектирования (схожая с `seda <https://www.mdw.la/papers/seda-sosp01.pdf>`_) предоставляет множество преимуществ по созданию мощных и масштабируемых потоков данных, например:

+ Подходит для визуального создания и управления направленными графами процессоров;
+ По своей сути является асинхронной, что обеспечивает очень высокую пропускную способность и естественную буферизацию даже при колебаниях скорости обработки и потока;
+ Обеспечивает высококонкурентную модель без необходимости участия разработчика в вопросах потокобезопасности;
+ Способствует развитию связанных и слабосвязанных компонентов, которые в последствии могут быть повторно использованы в других контекстах и способствовать тестированию;
+ Соединения с верхними и нижними границами по ресурсам делают такие критические функции, как *back pressure* и *pressure release*, закономерными и интуитивно понятными;
+ Обработка ошибок становится ожидаемым результатом успешного выполения конкретного сценария процесса (happy-path), а не catch-all;
+ Точки входа и выхода данных, а также перемещение потоков в системе, понятны и легко отслеживаются.

