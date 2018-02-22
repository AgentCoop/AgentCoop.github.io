---
layout: post
title:  "Сказ о шаблоне проектирования Singleton"
date:   2011-11-11 07:19:00 +0300
categories: fun
---
Сидя на заваленке в окружении внуков, греясь в теплых майский лучах солнца, старый Хэккер курил трубку, пуская клубы густого сизого дыма и изредка поглаживая седую бороду. Его уставшие глаза безропотно и неподвижно смотрели в голубое небо, с носящимися туда-сюда в погоне за насекомыми ласточками.
<img src="/assets/images/2011/design_pattern_illu.jpg" class="align-right" />

– Деда, а деда. А что такое шаблон проектирования? – спросил один из внуков.

– Шаблон проектирования... – задумчиво произнес дед. – Шаблон проектирования это то о чем много пишут в умных книгах, но, как правило, никогда не применяют на практике.

– Как это? – удивленно, почти хором, спросили внуки.

– А просто – слегка приподнявший, крякнул дед. – Была, значится, у меня одна такая история...

Давным-давно работал я на одну компанию и нужно было нам срочно сделать рефакторинг кода, ибо система из-за архитектурных перекосов и тонн говнокода при 100k уникальных посетителей в день  не справлялась с нагрузкой на достаточно мощном железе.

Мои коллеги по цеху слишком ретиво взялись за это дело и, не послушавшись вашего деда, влепив туда страшный костыль: при разносе базы данных на master и slave со словами "в будущем все исправим" был создана копия класса DB, отвечающего за работу с базой данных по имени DB2. Причем методы и свойства двух классов были абсолютно - может за небольшим исключением - идентичны. Один класс соединялся с БД master, а другой - с БД slave.

– А что же ты предложил, деда? – с интересом спросил один из внуков.

– А я предложил использовать старый надежный и проверенный временем, как автомат АК-47, класс Singleton, но не простой, а с небольшой заковыркой. – продолжал дед. – Вот этот класс и есть то, что называют шаблон проектирования.

– То есть шаблон проектирования - это некое проверенное временем программное решение?

– Именно.

С этими словами, достав из-за пазухи планшет, дед стал на нем что-то набрасывать.

– Вот. – спустя некоторые время произнес дед. – Смотрите.


{% highlight php %}
abstract class Singleton
{
    static private $object_pool = array();

    private function __construct() {/* ... */}
    private function __clone() {/* ... */}

    protected function init() {/* ... */}

    static final function getInstance()
    {
        $args = func_get_args();
        $class = get_called_class();
        if (static::useArgSignature() && count($args)) { // Args have been passed.
            $str = '';
            foreach ($args as $item) {
                if (is_array($item) || is_object($item)) {
                    $str .= serialize($item);
                }
                else {
                    $str .= $item;
                }
            }
            $key = md5($class . $str);
        }
        else { // Use class name as key
            $key = $class;
        }
        if (!isset(self::$object_pool[$key])) {
            $instance = new $class();
            self::$object_pool[$key] = $instance;
            call_user_func_array(array($instance, 'init'), $args);
            return $instance;
        }
        else {
            return self::$object_pool[$key];
        }
    }

    /**
    * Must return `true` if argument value singnature to be used for
    * the class instantiation.

    * @return boolean
    */
    static protected function useArgSignature()
    {
        return false;
    }
}
{% endhighlight %}

– Теперь наследуем наш класс DB от класса Singleton. – старый Хэккер продолжал строчить код.

{% highlight php %}
<?php
/**
 * DB class
 */
class DB extends Singleton
{
    protected function init($dsn)
    {
        // Establish connection to a database.
    }
    /**
     * Enable signature of argument values.
     */
    static protected function useArgSignature()
    {
        return true;
    }
}
{% endhighlight %}

– Теперь пример использования. – не унимался дед и все быстрее строчил что-то гусиным пером по планшету.

{% highlight php %}
// Example of use
$masterDSN = 'mysql://john:pass@master.dbnode/my_db';
$slaveDSN = 'mysql://john:pass@slave.dbnode/my_db';

// Establish connection with master database.
$masterDb = DB::getInstance($masterDSN);
// Slave database.
$slaveDb = DB::getInstance($slaveDSN);

// Returns reference to the same object.
$masterDb = DB::getInstance($masterDSN);
{% endhighlight %}

 – Мы определили дочерний класс DB и "сказали" ему использовать сигнатуру вызова медота ::getInstance. В зависимости от сигнатура нам возвращается тот или иной указатель на соответствующий объект класса DB, представляяющий соединение к master- или slave-БД.

Окончив говорить, дед выбил трубку и положил ее вместе с планшетом на землю. Посмотрев на внуков, дед закрыл глаза,  подставив своей старое морщинистое лицо весеннему солнцу. Внуки переглянулись между собой, но не стали  его тревожить - пусть отдыхает, решили они.  А сердце Хэккера билось, тук...тук........тук.