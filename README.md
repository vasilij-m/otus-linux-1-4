Написать скрипт для крона, который раз в час присылает на заданную почту
- X IP адресов (с наибольшим кол-вом запросов) с указанием кол-ва запросов c момента последнего запуска скрипта
- Y запрашиваемых адресов (с наибольшим кол-вом запросов) с указанием кол-ва запросов c момента последнего запуска скрипта
- все ошибки c момента последнего запуска
- список всех кодов возврата с указанием их кол-ва с момента последнего запуска
в письме должно быть прописан обрабатываемый временной диапазон, должна быть реализована защита от мультизапуска.

В ходе выполнения ДЗ был написан скрипт **logparser.sh**, состоящий из двух функций: **prepare** и **parser**. В начале скрипта описываются следующие переменные:
`$logfile` - лог-файл из задания, путь к файлу:`/vagrant/access.log` - заменить на путь до реального лога (напрмиер, для nginx это `/var/log/nginx/access.log`)
`$result` - результат работы последнего запуска скрипта, путь к файлу:`/var/log/parser/parser.log`
`$runtimelog` - файл для хранения даты из последней обработанной строки аксес-лога, путь до файла:`/var/log/parser/runtime.log`
`$templogfile` - временный файл, в который копируется необработанная часть аксес-лога, путь к файлу`/var/log/parser/tempaccess.log`
`$lockfile` - временный файл, предотвращающий повторный запуск скрипта, путь к файлу:`/tmp/parserlockfile`
`$emailaddress` - адрес, на который будет отправено письмо с отчетом - `"root@localhost"`
`$ipcount` - количество IP адресов с наибольшим количеством запросов, с которых клиенты обращались к серверу -`10` - выбрано значение 10.
`$urlcount` - количество страниц с наибольшим количеством запросов, которые запрашивали клиенты -`10` - выбрано значение 10.


Сначала в скрипте выполняется функция **prepare**:
```
prepare(){
    if ! [[  -f $runtimelog ]];then
#Если файл /var/log/parser/runtime.log, создать его и записать в него дату из первой строки аксес-лога. 
        head -1 $logfile | awk '{print $4}' | sed 's/\[//' > $runtimelog;
    fi
#Считать с файла /var/log/parser/runtime.log дату для начала обработки аксеслога в $processingtime и в переменную $starttime
    processingtime=$(cat $runtimelog | sed 's!/!\\/!g')
    starttime=$(cat $runtimelog)
#Сохранить из /vagrant/access.log все строки от строки, содержащей дату из $processingtime, и до конца в файл /var/log/parser/tempaccess.log:
    sed -n "/${processingtime}/,$ p" $logfile > $templogfile
#Считать из последней строки файла /var/log/parser/tempaccess.log дату и записать её в /var/log/parser/runtime.log и в переменную $endtime
    tail -1 $templogfile | awk '{print $4}' | sed 's/\[//' > $runtimelog
    endtime=$(cat $runtimelog)
}
```
Затем выполняется функция **parser**, которая формирует отчет в файл `/var/log/parser/parser.log` и с помощью утилиты `mailx` отправляет его на почту, адрес которой указан в переменной `$emailaddress`. 

В скрипте реализована защита от мультизапуска, основанная на создании файла блокировки по пути `/tmp/parserlockfile`. После выполнения скрипта или получении сигнала SIGINT или SIGTERM для его завершения файл блокировки удаляется, и скрипт может быть запущен снова.
