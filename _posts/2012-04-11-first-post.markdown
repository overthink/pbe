---
layout: post
title: First Post
---

Here is some text.

And some more. And some code:

{% highlight clojure %}
(defn munge [s]
  (let [ss (str s)
        ms (if (.contains ss "]")
             (let [idx (inc (.lastIndexOf ss "]"))]
               (str (subs ss 0 idx)
                    (clojure.lang.Compiler/munge (subs ss idx))))
             (clojure.lang.Compiler/munge ss))
        ms (if (js-reserved ms) (str ms "$") ms)]
    (if (symbol? s)
      (symbol ms)
      ms)))
{% endhighlight %}

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Curabitur aliquam, nunc at lobortis consectetur, elit sem elementum mauris, ac cursus turpis mauris sed sem. Vestibulum nisl felis, volutpat at convallis non, gravida vitae ipsum. Nam quis quam at elit tempor egestas. Aenean turpis eros, tristique porta iaculis eu, pellentesque sed nibh. Aliquam erat volutpat. Aenean vitae eros nec diam tincidunt facilisis. Maecenas egestas dignissim enim, id accumsan eros consequat nec. Donec iaculis velit ut enim gravida lobortis. Praesent ante sem, faucibus eget luctus sed, sagittis nec urna. Nulla imperdiet convallis aliquet. Praesent tellus dui, luctus iaculis sagittis et, rhoncus in metus. Nam sit amet purus magna, vel cursus nulla. Fusce tempor lectus a mauris rhoncus vitae vestibulum libero sollicitudin. Maecenas fringilla nisl a augue viverra quis congue felis pulvinar.

Aliquam vel condimentum tortor. Duis ut quam nec est pretium vulputate. Suspendisse potenti. Suspendisse eu purus turpis, ac luctus massa. Sed varius tellus sed ipsum sagittis id pulvinar dui commodo. Vestibulum congue neque interdum dolor dignissim molestie. Curabitur aliquet magna non diam tempor a commodo tellus scelerisque. Aenean dignissim fermentum tellus eget vestibulum. Curabitur consectetur accumsan orci ac porttitor. Sed quis aliquam nunc. Fusce dui massa, mollis et viverra vitae, ultricies at justo. Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed non consequat turpis. Ut dolor mauris, mollis et varius vel, mollis sit amet nulla. Fusce ullamcorper porttitor libero, ac euismod augue consectetur vel. Quisque pretium, odio id dictum tempus, urna lacus hendrerit erat, eget lobortis nunc nisi sit amet enim.

Nam sapien ligula, sagittis ut pellentesque sit amet, interdum id nunc. Sed consequat massa sed quam tincidunt eu cursus est commodo. Donec eleifend pulvinar risus sed aliquet. Aenean a lobortis magna. Cras interdum ipsum nec felis pellentesque gravida vel nec diam. Aliquam a pellentesque nibh. Proin ac leo arcu, et viverra tortor. Suspendisse tincidunt feugiat bibendum. Donec venenatis semper sagittis.

Nullam tempor ultrices augue id dapibus. Curabitur vitae magna fermentum ipsum vehicula eleifend. Vestibulum luctus bibendum ipsum sit amet dapibus. Suspendisse potenti. Phasellus porttitor enim et nibh sodales porttitor. Ut pharetra, enim sed aliquam fermentum, augue odio sodales dui, euismod fringilla erat quam at nunc. Nam sodales aliquet gravida. Etiam ac nisl neque, a hendrerit lectus. Maecenas quis lorem a metus ornare egestas. Nullam commodo tincidunt quam, quis pretium nisi dignissim sit amet. Praesent auctor, leo sit amet accumsan sodales, dui lorem malesuada diam, vel condimentum odio elit a lectus. Praesent sed nulla diam, id fermentum justo. Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Duis vel felis ante, id venenatis nibh. Mauris quis nisl eleifend massa cursus viverra quis nec mi. Morbi nec sapien id lacus sagittis auctor sed molestie nulla.

Integer vel nibh nunc. Phasellus vehicula dapibus neque tempor placerat. Nulla pharetra tempus augue vitae adipiscing. Mauris eu magna felis, vitae auctor risus. Vestibulum et nulla vitae neque aliquam ullamcorper vitae ut elit. Nullam nec elit eget nulla fermentum molestie vel sit amet diam. Mauris imperdiet tempus orci, non eleifend erat malesuada eget. In hac habitasse platea dictumst.

