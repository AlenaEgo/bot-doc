# 6. **Примечания к алгоритму**

На данной странице собраны подробные описания неочевидных моментов работы алгоритма робота.

## 6.1. **Особенности использования торговых стаканов**

В роботе для получения цен используются таблицы/потоки торговых стаканов по инструментам. Особенность работы со стаканами такова, что стакан не приходит с биржи в готовом виде. Чтобы минимизировать объём пересылаемой информации биржа присылает сначала снимок/слепок стакана по инструменту на какой-то момент времени, а потом приходят только обновления, сообщающие об изменениях в стакане. Робот НЕ строит стакан сразу по всем бумагам, стакан строится только по используемым в портфелях бумагам. Поэтому, для добавления новой бумаги в список бумаг, по которым строится стакан, необходимо переоткрыть поток с текущим слепком, а потом применять к нему обновления из потока инкрементальных обновлений. В момент получения потока со слепком, стакан в роботе временно перестает обновляться по всем используемым бумагам. Он начнет обновляться только после того как будет закрыт поток со слепком (это может занять некоторое время, зависящее от общего количества бумаг, приходящих в данном потоке, и размера стаканов) и начнут применяться инкрементальные обновления. Соответственно, в момент переоткрытия стакана вы можете наблюдать отсутствие цен по бумагам. Переоткрытие требуется в большинстве подключений, потому что стаканы по всем бумагам или полный лог заявок приходят, как правило, в одном потоке, т.е. нельзя получать данные по конкретным бумагам, получать вы всегда будете все данные, а использовать можете только для нужных бумаг.

Стакан в роботе может переоткрываться в следующих случаях:
- создание нового портфеля;
- добавление бумаги в портфель;
- снятие флага `Disabled` с портфеля;
- пропуски в номерах сообщений инкрементального потока обновлений стакана в UDP подключениях к биржам (чем больше у вас портфелей и бумаг, тем выше вероятность возникновения этих пропусков).

Т.е. произойдёт приостановка торговли по всем портфелям, использующим инструменты данной биржи. Это не является ошибочным поведением робота.

## **6.2. Использование приказа переместить заявку**

На некоторых подключениях реализована отправка приказа переместить заявку (иногда эта команда также называется изменением заявки). Использование данного приказа позволяет сократить количество транзакций и увеличить отношение количества сделок к количеству транзакций, особенно при котировании, а так же увеличить время нахождения заявок в рынке. Так как особенности использования данной команды отличаются на разных рынках и типах подключений, то, в зависимости от подключения, данный приказ может применяться только для первой ноги портфеля или же для инструментов обеих ног. На данный момент отправка такой команды реализована для FIX-подключений фондового и валютного рынков Московской биржи, а так же TWIME-подключения срочного рынка Московской биржи.

Использование приказа переместить заявку для поддерживаемых подключений не является обязательным. Эта возможность может быть отключена при создании нового транзакционного подключения.

## **6.3. Правила перемещения Lim_Sell и Lim_Buy**

Сигнальные цены [Lim_Sell]() и [Lim_Buy]() перемещаются только при прохождении сделок по [Is first]() инструменту портфеля.

Правила перемещения сигнальных цен можно разделить на две части: произошла продажа по [Is first]() бумаге и произошла покупка по [Is first]() бумаге. Внутри каждой из этих частей алгоритм делится еще на две части: позиция портфеля до прохождения данной сделки была равна нулю и была не равна нулю.

Введем следующие обозначения: `diffpos` - знаковое количество лотов в сделке по [Is first]() бумаге, `V` - это `v_in_left × Count` или `v_out_left × Count` в зависимости от того открываем мы позицию или закрываем данной заявкой, `Count` - это `Count Is first` бумаги, `Сurpos` - текущая позиция по [Is first]() бумаге портфеля (т.е. прошедшая только что сделка еще НЕ учтена), нижний индекс 0 - предыдущее значение параметра, 1 - новое значение параметра. В данных обозначениях алгоритм перемещения сигнальных цен примет вид:

- прошла продажа(соответственно в количестве `diffpos`):

![Alt text](00-img/6-3-1.jpg)

- прошла покупка (соответственно в количестве diffpos):

![Alt text](00-img/6-3-2.jpg)

Также перемещение сигнальных цен происходит когда заявка не может быть выставлена из-за ограничений по [v_min](), [v_max](), [To0](). Если робот не может купить из-за ограничений по [v_max](), то в соответствии с параметрами портфеля [Limits timer]() и [Percent]() цены [Lim_Sell]() и [Lim_Buy]() уменьшаются на величину параметра портфеля [K](), если же робот не может продать из-за ограничений по [v_min](), то в соответствии с параметрами портфеля [Limits timer]() и [Percent]() цены [Lim_Sell]() и [Lim_Buy]() увеличиваются на величину параметра портфеля [K]().

## **6.4. Особенности поведения заявок, переставленных по SL или Timer**

Если после переставления заявки по событиям [SL]() или [Timer]() новая заявка не проходит в течение 1 секунды, то она будет автоматически переставлена по цене `bid - k_sl` на продажу или `offer + k_sl` на покупку, в не зависимости от значений [SL]() и [Timer]() и в не зависимости от того включен ли [Timer]() вообще. Заявки, выставляемые при закрытии или выравнивании позиции (это либо настройка в расписании, либо "кликер" `To market`) всегда выставляются с включенным таймером и значение параметра [Timer]() для таких заявок всегда равно 1.

## **6.5. Подсчёт финансового результата**

Финансовый результат (в смысле просто число) считается по сделкам и никаких "экзотических" случаев, связанных с его подсчетом нет. НО есть случаи когда не будет раздвижки в таблице финансовых результатов. Нужно запомнить главное правило, чтобы была раздвижка, должна быть сделка по главной бумаге, если ее нет, то и раздвижки нет. То есть, если у вас по какой-то причине "кривая" позиция и вы ее "подравняли", нажав на кнопку `To market`, то вы не получите нормальной раздвижки в данной таблице. У вас будет только одна "кривая" раздвижка, в которой фигурирует только главная бумага (как пример, флуд контроль, проходят сделки по первой ноге, а по второй не дают выставиться, у вас будут одноногие "кривые" раздвижки с первой ногой, а после нажатия To market, когда пройдут сделки, т.к. они были только по второй ноге, то раздвижки в таблице не будет, но с финансовым результатом все будет в порядке).

И еще один вариант, это когда [Count]() первой ноги больше [Count]()-а второй. Т.е. пусть вы торгуете долларом валюты против доллара на срочном рынке, но валюта первая нога. Т.е. у вас стоит [Count]() у валюты 100, а у срочки 1. Т.е. вы перекрываете на срочке каждые 100 контрактов валютки. Вот вы выставили 100 контрактов на валютке, у вас взяли 60, раздвижки в таблице не будет, т.к. она получится заведомо "кривой", вторую ногу же не кидали, потом прошло еще 50, и вы снова не увидите раздвижки. Да, вы кините одну бумагу на срочку, она пройдет (в финансовом результате все нормально учтется). Но опять же не совсем ясно к каким сделкам ее привязывать, если привязать как обычно к последней (т.е. к 50), то будет заведомо кривая раздвижка, а искать какие-то предыдущие сделки уже не вариант, т.к. все могло быть не так просто, как в описанном примере.

## **6.6. О ценах выставления заявок второй ноги**

В момент выставлении заявки по [Is first]() инструменту запоминаются текущие цены по остальным бумагам. Таким образом в момент совершении сделки по [Is first]() инструменту все остальные бумаги в портфеле выставятся по ценам, которые робот запомнил во время выставления заявки по [Is first]() инструменту.

## **6.7. Об объёмах выставления заявок**

Для некоторых бирж (к примеру Deribit) если объём в заявке не кратен лоту, то объем выставляемой заявки принудительно округляется до целого количества лотов(например лот равен 10, объём выставляемой заявки 25, то в действительности на бирже выставится заявка объемом 20 (округление всегда происходит в меньшую сторону)). Если при подобном округлении объём выставляемой заявки станет равным нулём, то вы получите ошибку выставления `REASON_ZERO_AMOUNT_TO_MULTIPLE`.