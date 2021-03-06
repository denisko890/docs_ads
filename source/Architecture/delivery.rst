Механизм доставки сообщений
============================

После введения в принципы работы поставщиков и потребителей данных следует изучить семантический анализ гарантий между ними, которые обеспечивает **ADS**. Существует несколько возможных гарантий по доставке сообщений:

+ *At most once* -- сообщения могут быть потеряны, но никогда не будут повторно отправлены.
+ *At least once* -- сообщения никогда не теряются, но могут быть повторно отправлены.
+ *Exactly once* -- наиболее востребованное -- каждое сообщение доставляется один и только один раз.

Все сводится к двум проблемам: гарантии долговечности при публикации данных и гарантии при их считывании.

Многие системы утверждают, что обеспечивают гарантию "Exactly once", но при этом большинство подобных утверждений ошибочно (т.к. ими не предусмотрены случаи, когда потребители или поставщики могут потерпеть неудачу; при многопотребительских процессах; когда данные, записанные на диск, могут быть потеряны).

Семантика **ADS** прямолинейна. При публикации сообщения, оно фиксируется в журнале. Как только опубликованное сообщение зафиксировано, оно не может быть потеряно, пока брокер, реплицирующий партицию, на которую было написано это сообщение, остается "живым". Определения зафиксированного сообщения, живой партиции, а также описание типов сбоев приведено подробно в следующем разделе. Сейчас рассмотрим идеальный брокер без потерь и попытаемся понять гарантии поставщика и потребителя. Если поставщик пытается опубликовать сообщение и возникает сетевая ошибка, он не может быть уверен в том, когда произошла ошибка -- до или после фиксации сообщения. Аналогично семантике операции вставки в таблицу базы данных с автогенерацией ключа.

Поставщик **ADS** поддерживает опцию идемпотентной доставки, гарантирующую, что повторная отправка не приводит к дублированию записей в журнале. Для достижения этого брокер назначает каждому поставщику идентификатор и дедуплицирует данные, используя порядковый номер, который отправляется поставщиком вместе с каждым сообщением. Так же поставщику предоставляется возможность отправки сообщений в несколько партиций топика, используя подобную транзакционной семантику: то есть либо успешно записаны все сообщения, либо не записано ни одно из них. Основным вариантом использования этого метода является однократная обработка между топиками **ADS** (описание приведено ниже).

Не все варианты использования требуют таких строгих гарантий. Для чувствительных к задержкам случаев у поставщика есть возможность определить желаемый уровень устойчивости (время ожидания фиксации сообщения может занять порядка *10 мс*). Однако поставщик может также указать, что он хочет выполнить запись данных полностью асинхронно, или что он хочет ждать только до тех пор, пока лидер (но не обязательно последователи) получит сообщение.

Теперь рассмотрим семантику с точки зрения потребителя. Все реплики имеют один и тот же журнал с одинаковыми смещениями. Потребитель контролирует свое положение в этом журнале. Если у потребителя не было сбоев, то он просто хранит эту позицию в памяти. Но в случае если потребитель терпит неудачу и необходимо, чтобы партиция этого топика была перехвачена другим процессом, новому процессу нужно выбрать соответствующую позицию, с которой следует продолжить обработку. К примеру, потребитель считывает данные -- он имеет два варианта обработки данных и обновления своей позиции:

1. Потребитель может считывать сообщения, затем сохранять свою позицию в журнале и потом выполнять обработку. В этом случае существует вероятность того, что процесс выйдет из строя после сохранения позиции потребителя и до сохранения результатов обработки данных. Тогда другой процесс, перехвативший на себя обработку, начинается с сохраненной позиции, даже если какие-то сообщения до этой позиции не были обработаны. Это соответствует семантике "At most once", так как в случае сбоя потребителя данные могут быть не обработаны.

2. Потребитель может считывать сообщения, затем обрабатывать их и потом сохранять свою позицию. В этом случае существует вероятность того, что процесс выйдет из строя после обработки данных, но до сохранения позиции потребителя. Тогда другой процесс, перехвативший на себя обработку, обрабатывает уже обработанные данные. В случае сбоя потребителя это соответствует семантике "At least once". При этом во многих случаях сообщения имеют первичный ключ, поэтому обновления являются идемпотентными (повторное получение одного и того же сообщения просто перезаписывает его другой копией самого себя).

Что по поводу семантики "Exactly once" (то, что действительно желаемо)? При потреблении данных из топика **ADS** и их записи в другой топик, можно использовать транзакционную семантику поставщика, которая была упомянута выше. Позиция потребителя сохраняется как сообщение в топике, поэтому можно записать смещение в **ADS** в той же транзакции, что и топики смещения, получающие обработанные данные. Если транзакция прерывается, позиция потребителя возвращается к своему старому значению, и записанные в топик смещения данные доступны другим потребителям в зависимости от их "уровня изоляции". На установленном по умолчанию уровне изоляции "read_uncommitted" все сообщения видны для потребителей, даже если они были частью прерванной транзакции, а на уровне "read_committed" потребитель возвращает только зафиксированные транзакцией сообщения (и любые данные, которые не являлись частью транзакции).

При записи данных во внешнюю систему существует оговорка, которая заключается в необходимости координировать позицию потребителя с тем, что фактически хранится в качестве выходных данных. Классический способ достижения этого -- ведение двухфазной фиксации между хранением потребительской позиции и хранением потребительских данных. Но также этого можно добиться и проще -- позволяя потребителю хранить смещение в том же месте, что и выходные данные. Например, коннектор **ADS Connect** записывает данные в **HDFS** вместе со смещениями считанных данных для гарантии их обновлений. Аналогичный шаблон используется для многих других систем данных, требующих более сильную семантику, и у которых сообщения не имеют первичного ключа для дедупликации. Такой метод лучше, потому что многие выходные системы, в которые потребитель может записывать данные, не поддерживают двухфазную фиксацию. 

Так что **ADS** эффективно поддерживает гарантию "Exactly once", и транзакции поставщик/потребитель могут использоваться для ее обеспечения при передаче и обработке данных между топиками. "Exactly once" гарантия в других системах обычно требует взаимодействия с такими же системами, а в **ADS** обеспечивается смещение, которое это реализует. В иных случаях **ADS** по умолчанию гарантирует как минимум доставку "At least once" и дает возможность пользователю реализовывать доставку по принципу "At most once" путем отключения повторных попыток записи у поставщика и фиксации смещений у потребителя перед обработкой пакета данных.
