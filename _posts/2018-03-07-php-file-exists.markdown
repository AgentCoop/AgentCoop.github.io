---
layout: post
title:  "Say yes and no to 'terse and elegant code'"
date:   2018-03-07 15:45:00 +0300
categories: programming php
---
Assume you need to check whether a particular file exists in several locations using PHP funtion [file_exists](http://php.net/manual/en/function.file-exists.php). Besides that, you have do that checking in some prioritized order. How would you do that? My first solution right off the bat was something like this:

{% highlight php %}
$phpini = null;
$loc1 = '/etc/php/php.ini';
$loc2 = '/usr/local/etc/php/php.ini';
$loc3 = '/etc/php5/apache2/php.ini ';

if (file_exists($loc1)) {
    $phpini = $loc1;
} else if (file_exists($loc2)) {
    $phpini = $loc2;
} else if (file_exists($loc3)) {
    $phpini = $loc3;
}

if ($phpini) {
    // Do something
}
{% endhighlight %}

That's a quite straightforward solution. Now let's re-write it in a more terse and elegant way:

{% highlight php %}
$phpini = null;
$loc1 = '/etc/php/php.ini';
$loc2 = '/usr/local/etc/php/php.ini';
$loc3 = '/etc/php5/apache2/php.ini ';

// Hint:  $a = [true => 1, true => 2, true => 3] will yield [true => 3] array
$fileExists = [
    file_exists($loc3) => $loc3,
    file_exists($loc2) => $loc2,
    file_exists($loc1) => $loc1
];

$phpini = ( $fileExists[true] ?? null );

if ($phpini) {
    // Do something
}
{% endhighlight %}

It does the same thing, only we got rid of three if statements. Nice, programmers like elegant and terse code. And that is actually the reason for why sometimes their code becomes hard to read, it becomes too "perlish".

Don't forget that your code is not only for you. Your code, no doubt, has to be elegant, but it has to be readable as well even for novices, which may maintain your code after you.

Finding a golden ratio between readability and terseness is what makes one a good programmer.