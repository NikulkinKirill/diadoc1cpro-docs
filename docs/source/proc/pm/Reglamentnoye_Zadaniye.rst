
Регламентное задание
====================

При большом объеме обрабатываемых документов часть задач можно переложить на выполнение их в фоновом режиме без участия пользователя.

Алгоритм работы описывается в функции :doc:`ВыполнитьРегламентныеДействия <../../func/pm/Vypolnit'_Reglamentnyye_Deystviya>`.

Выполнение алгоритма запускается двумя способами:

* при нажатии кнопки "Выполнить регламентное задание" в настройках модуля
* при запуске регламентного задания программно из конфигурации

В первом случае алгоритм запускается на клиенте. Поэтому он может содержать обращения к экспортным функциям, расположенным в формах основного модуля.

Второй вариант имеет несколько ограничений:

* использовать можно только методы, работающие на сервере
* обращение к формам модуля невозможно
* требуется внесение изменений в конфигурацию (добавление регламентного задания в 1С, описание функции вызова :doc:`ВыполнитьРегламентныеДействия <../../func/pm/Vypolnit'_Reglamentnyye_Deystviya>`)

Чаще всего использование регламентного задания предполагает:

* обновление ленты событий
* проведение анализа документов и запись требуемых действий
* отправка на подпись или согласование
* выполнение сопоставления документов
* подписание и отправка сертификатом установленным на сервере 1С

Пример получения новых событий и отправка документов:

::

      // ОБРАБОТКА СОБЫТИЙ ЛЕНТЫ, В Т.Ч. ЗАГРУЗКА НОВЫХ ДОКУМЕНТОВ

      КоличествоПорцийСобытий = 10; // Кол-во вычитываемых порций событий (каждая порция содержит не более 100 событий)
      Для Каждого СтрокаОрганизации Из ОсновнойМодуль.ЭДО_Модуль_ТаблицаОрганизацийПользователя() Цикл
          ОсталосьСобытийВЛенте = ОсновнойМодуль.ЭДО_Модуль_ОбработатьНовыеСобытияДокументов(СтрокаОрганизации, КоличествоПорцийСобытий);
      КонецЦикла;

      // ВЫПОЛНЕНИЕ MessagePatchToPost

      Режим = ""; // Режим исполнения: ПередатьНаСогласование / ПередатьНаПодпись / ПередатьПоМаршруту / Согласование / ОтказВСогласовании

      ПараметрыMessagePatchToPost = Новый Структура;
      ПараметрыMessagePatchToPost.Вставить("Действие"                  , Режим); // вариант режима MessagePatchToPost
      ПараметрыMessagePatchToPost.Вставить("ИдентификаторСотрудника"   , Неопределено); // внутренний ID Диадока
      ПараметрыMessagePatchToPost.Вставить("ИдентификаторПодразделения", Неопределено); // внутренний ID Диадока
      ПараметрыMessagePatchToPost.Вставить("ИдентификаторМаршрута"     , Неопределено); // внутренний ID Диадока
      ПараметрыMessagePatchToPost.Вставить("Комментарий"               , "");           // произвольный текст

      // Произвольная коллекция документов Диадока, которые необходимо пропатчить (необходимо собрать по нужному алгоритму).
      // Элементы этой коллекции должны содержать ключ "ДокументДД".
      ТаблицаДокументов = Новый ТаблицаЗначений;

      Если ЗначениеЗаполнено(Режим) Тогда
          ОсновнойМодуль.ЭДО_ОтправитьMessagePatchToPostДляВыбранныхСтрокСпискаДокументов(ТаблицаДокументов, ПараметрыMessagePatchToPost);
      КонецЕсли;


При большом объеме документов на отправку возникает потребность убрать эту задачу с пользователя.

В этом случае тот сертификат, которым будет производиться подписание, устанавливается на сервер 1С.

Отпечаток сертификата указывается в настройках основного модуля для организации: элемент организации → закладка "Прочие настройки" → поле "Отпечаток сертификата на сервере 1С"

Пример подписания документов на сервере:

::

  // ПРИМЕР ПОДПИСАНИЯ И ОТПРАВКИ СЕРТИФИКАТОМ УСТАНОВЛЕННЫМ НА СЕРВЕРЕ 1С
  // ОГРАНИЧЕНИЯ: сервер под Windows; закрытый ключ сертификата установлен под учеткой агента сервера 1С с сохраненным пин-кодом

  // 1. Авторизация под сертификатами сервера 1С
  ОсновнойМодуль.ЭДО_АвторизоватьсяПодСертификатомНаСервере1С();
  КонтекстСеанса = ОсновнойМодуль.ЭДО_КонтекстСеансаКлиентСервер();

  // 2. Параметры для получения списка пакетов на отправку
  ТаблицаВидовПакетов	 = ОсновнойМодуль.ЭДО_СправочникМенеджер_ПолучитьСписокЭлементов("ВидыПакетов");
  МассивВидовПакетов	 = ТаблицаВидовПакетов.ВыгрузитьКолонку("Ссылка");

  ПараметрыОбновленияСписка = Новый Структура;
  ПараметрыОбновленияСписка.Вставить("Режим", "ОтправкаПакетов");
  ПараметрыОбновленияСписка.Вставить("НачалоПериода", ДобавитьМесяц(ТекущаяДата(), -1));
  ПараметрыОбновленияСписка.Вставить("КонецПериода", КонецДня(ТекущаяДата()));
  ПараметрыОбновленияСписка.Вставить("МассивВыбранныхВидов", МассивВидовПакетов);

  //3. Отправка пакетов по организациям, в которых авторизовались под сертом сервера 1С
  Для Каждого Элемент Из КонтекстСеанса Цикл

    СтрокаКонтекста = Элемент.Значение;
    Организация = СтрокаКонтекста.ОрганизацияДиадок.СвязанныйСправочник1;

    ПараметрыОбновленияСписка.Вставить("ОтборПоОрганизации", Организация);

    ОсновнойМодуль.ЭДО_Модуль_ОбновитьСписокДокументов(ПараметрыОбновленияСписка);

    //1 вариант: последовательная отправка
    Для Каждого СтрокаТЧ Из ОсновнойМодуль.СписокДокументов Цикл
        ОсновнойМодуль.ЭДО_ПодготовитьИОтправитьПакет(СтрокаТЧ);
    КонецЦикла;

    //2 вариант: фоновая отправка
    ОсновнойМодуль.ЭДО_ПодготовитьИОтправитьПакетыВФоне(ОсновнойМодуль.СписокДокументов);

  КонецЦикла;
