---
layout: post
title:  "Особенности шаблонизатора PHP"
date:   2012-02-28 15:33:00 +0300
categories: programming php
---
PHP замечательный язык программирования. При всех его недостатках он не перестает удивлять. Недавно столкнулся со следующим — на первый взгляд загадочным — его поведением.

Как известно PHP имеет встроенный шаблонизатор. Весь текст, который интерпретатор встречает между тегами обозначающими конец и начало PHP кода, он отправляет в буфер вывода. Убедиться в этом можно на следующем примере:

{% highlight php %}
<?php
echo "Hello, ";
?>

World.

<?php

echo "All is fine\n";
{% endhighlight %}

Выводом программы будет «Hello, World. All is fine», что и следовало ожидать. Но что происходит на самом деле? Посмотрим на другой пример:

{% highlight php %}
<?php
$three = function() {
?>
        Three
<?php
};

$one = function() {
?>
        One
<?php
};

$two = function() {
?>
        Two
<?php
};

$one();
$two();
$three();
{% endhighlight %}

Если выполнить исходный код, то выводом программы будет «One Two Three», что немного странно. Ведь текст в коде встречался совсем в другой последовательности и в буфер вывода должно было попасть «Three One Two».

На самом деле PHP не отправляет текст в буфер вывода как только он его встречает. В интерпретаторе языка есть специальный опкод ZEND_ECHO (в этот опкод транслируется echo) и кусок текста между PHP кодом будут транслироваться в аргумент этого опкода. Именно поэтому у нас текст во втором примере выводиться в той последовательности в которой мы вызываем созданные анонимные функции (вывод текста стал частью анонимных функций благодаря опкоду ZEND_ECHO.

В подтверждении моих слов кусочек содержимого файла zend_language_parser.y

{% highlight c %}
|	T_INLINE_HTML			{ zend_do_echo(&$1 TSRMLS_CC); }
{% endhighlight %}

И реализация самой функции zend_do_echo из zend_compile.c:
{% highlight c %}
void zend_do_echo(const znode *arg TSRMLS_DC)
{
	zend_op *opline = get_next_op(CG(active_op_array) TSRMLS_CC);

	opline->opcode = ZEND_ECHO;
	SET_NODE(opline->op1, arg);
	SET_UNUSED(opline->op2);
}
{% endhighlight %}

Из официальной документации поведение интерпретатора не совсем очевидно. Более того в ней совсем не упоминается о преобразовании текста в аргумент опкода:

{% highlight text %}
when the PHP interpreter hits the ?> closing tags, it simply starts outputting whatever it finds (except for an immediately following newline — see instruction separation) until it hits another opening tag
{% endhighlight %}