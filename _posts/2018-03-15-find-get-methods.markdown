---
layout: post
title:  "The difference between getSomething and findSomething methods in OOP"
date:   2018-03-15 19:02:00 +0300
categories: programming oop
---
Sometimes semantics behind method (function) names is very important. Well, probably, naming things in a correct way is one of the most important aspects of programming. But, before talking about the subject of this essay, let's take a look at a real code taken from production:

{% highlight php %}
    /**
     * Notify consultants
     *
     * @return array
     */
    private function sendConsultantNotification()
    {
        $results = [];
        $consultant = null;

        try {
            $consultant = User::getByLegacyId($this->order->getConsultantLegacyId());

            dispatch(new NotifyConsultant($consultant->getFullName(), $consultant->getEmail(), $this->order->getId()))
                ->onQueue('mailer');

            $results[] = $consultant->getEmail();
        } catch (\Exception $e) {
            // Nothing to do
        }

        try {
            $freeConsultant = User::getByLegacyId($this->order->getFreeConsultantLegacyId());
        } catch (\Exception $e) {
            return;
        }

        if ($consultant && $freeConsultant->getId() == $consultant->getId()) {
            return $results;
        }

        dispatch(new NotifyConsultant($freeConsultant->getFullName(), $freeConsultant->getEmail(), $this->order->getId()))
            ->onQueue('mailer');

        $results[] = $freeConsultant->getEmail();

        return $results;
    }
{% endhighlight %}

As you can see, some parts of the code are wrapped in try..catch statements. So, you might guess that *getByLegacyId* method throws an exception if user is not found, and that's exactly what happens.

And why throw an exception instead of returning null? The main reason for that is some objects are critical for business logic  - if you don't have one, you can't proceed. That's it.

Instead of having lots of ifs scattered around your code, checking what your method returned, you can just throw an exception and handle it in the corresponding catch block, if one of those critical objects was not retrieved.

In other words, when you see a getSomething method, you should expect an object returned from that method or an exception should be thrown otherwise. To the contrary, findSomething might return null, so that you have to check returned values every time.

To not to write the same code twice, you can implement both methods as follows:

{% highlight php %}
class User
{
    public function getByLegacyId($userId)
    {
        $user = null;

        // Fetch user object

        if ( ! $user) {
            throw new \ModelNotFound();
        }

        return $user;
    }

    public function findByLegacyId($userId)
    {
        try {
            $user = self::getByLegacyId($userId);
        } catch (\ModelNotFound $e) {
            return null;
        }

        return $user;
    }
}
{% endhighlight %}

Now, having that, let's re-write our code:

{% highlight php %}
/**
 * Notify consultants
 *
 * @return array
 */
private function sendConsultantNotification()
{
    $results = [];
    $consultant = null;

    $consultant = User::findByLegacyId($this->order->getConsultantLegacyId());

    if ($consultant) {
        dispatch(new NotifyConsultant($consultant->getFullName(), $consultant->getEmail(), $this->order->getId()))
            ->onQueue('mailer');

        $results[] = $consultant->getEmail();
    }

    $freeConsultant = User::findByLegacyId($this->order->getFreeConsultantLegacyId());

    if ( ! $freeConsultant) {
        return;
    }

    if ($consultant && $freeConsultant->getId() == $consultant->getId()) {
        return $results;
    }

    dispatch(new NotifyConsultant($freeConsultant->getFullName(), $freeConsultant->getEmail(), $this->order->getId()))
        ->onQueue('mailer');

    $results[] = $freeConsultant->getEmail();

    return $results;
}
{% endhighlight %}

Much better, isn't it? We got rid of try...catch statements, which were a bit artificial, and made the code clearer.
