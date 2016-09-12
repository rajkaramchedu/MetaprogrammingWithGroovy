Metaprogramming Tutorial with Groovy
=============================

---------------------------

:By: 
    Raj Karamchedu
:Date: 
    12 September 2016
|
|
.. Contents:

.. toctree::
   :maxdepth: 3

.. Indices and tables


.. * :ref:`genindex`
.. * :ref:`modindex`
.. * :ref:`search`

Back in 2001 I worked at a semiconductor company called Chameleon Systems. Our product was a reconfigurable processor that can be hardwire-reconfigurable on the fly, thereby adapting its operation while the chip was running, i.e., during runtime. Communication system designers really liked the idea of a fully reconfigurable processor because it gave them full control of what algorithms they wanted to run on the chip, and to change these algorithms without powering the chip off. 

Metaprogramming is a similar concept--but not exactly the same--in software programming. It refers to a capability of one program to make changes to another program during runtime. In this short tutorial I will introduce metaprogramming by using Groovy, a dynamic language. 

A Groovy programmer can infuse a capability into a Groovy program such that this program can change itself during runtime or compile time. Why would she want to do that? And what sort of changes are we talking about? For a programmer, the most meaningful tools give her an ability to effect runtime changes like adding methods and properties, intercepting method calls, getting and setting properties. These changes, i.e., metaprogramming, give her a measure of control during runtime to adapt the running code to changing external conditions. 

Metaprogramming in Groovy
----------------------------------
Groovy compiler and runtime engine (from now on I'll refer to this "groovy engine") follows a particular flow in handling what's written in a Groovy program. In fact, the entire Groovy system is built to support this metaprogramming capability. The Groovy language has meta classes, meta objects and the Groovy engine itself will follow a specific set of rules. 

This combination of meta classes, meta objects and the rules followed by Groovy engine is pulled together into a concept known as MOP (meta object protocol). 

In other words, the Groovy engine says, "Here are metaprogramming features the Groovy language has, and when I see one of these in your program, I will consult the MOP and this MOP will then tell me what rules I should follow." When you write a Groovy metaprogram, you are doing two things: you are writing your Groovy program, and you are also designing your own variation of MOP, i.e., you are customizing the MOP. 

This means that the first thing we have to do is to learn how to write our own MOP, remembering that it is through our customized MOP that we can dictate the behavior of Groovy engine. The way you can customize this Groovy engine behavior is to put in a few hooks into the MOP. This entire tutorial is focused on explaining how to do this during runtime. We are not going to discuss compile time metaprogramming and any other advanced metaprogramming topics in this tutorial. 

Let's start with a few basics of a Groovy class and a Groovy object. 


Groovy Object
----------------
Every Groovy object implements an interface called GroovyObject interface. This GroovyObject interface is defined as below.

:: 

    public interface GroovyObject {
    	
    	MetaClass getMetaClass();
    	Object getProperty(String propertyName)
    	Object invokeMethod(String name, Object args)
    	void setMetaClass(MetaClass metaClass)
    	void setProperty(String propertyName, Object newValue)
    }


For example, if you compile Test.groovy that has just this one line of code:
::
    class Sidonia {
    }

it will result in Sidonia.class that has this code statement.
::

    public class Sidonia implements GroovyObject {...}


Calling a method that does not exist
--------------------------------------
Now let's see how we can call a method even if that method doesn't exist on a class. 

The below code runs fine, as we expect, because we are calling the test() method that is present on the class.

::

    package com.rajkaramchedu

    /**
     * Created by RajkMacPro on 9/11/16.
     */
    class InvokeDemo {

        def test() {

            return "Method exists"
        }
    }

    def invokeDemo = new InvokeDemo()
    assert invokeDemo.test() == "Method exists"


However, if we try to invoke a method such as someMethod(), which is not defined in the class, it will fail, as we expect it should. 

::

    def invokeDemo = new InvokeDemo()
    assert invokeDemo.test() == "Method exists"
    assert invokeDemo.someMethod() == "Called invokeMethod someMethod()"

The above code will fail with the error message:
::

    Caught: groovy.lang.MissingMethodException: No signature of method: com.rajkaramchedu.InvokeDemo.someMethod() is applicable for argument types: () values: []


You remember the invokeMethod() method defined in the interface GroovyObject above? If we implement this invokeMethod() in our class, we can change the behavior of Groovy engine, because whenever the engine sees this invokeMethod() it will always invoke it first, bypassing the other methods in the class. 

Here's a code that uses metaprogramming hook invokeMethod() below:

::

    package com.rajkaramchedu

    /**
     * Created by RajkMacPro on 9/11/16.
     */
    class InvokeDemo {

        def invokeMethod (String name, Object args){
            return "Called invokeMethod $name $args"

        }

        def test() {

            return "Method exists"
        }
    }

    def invokeDemo = new InvokeDemo()
    assert invokeDemo.test() == "Method exists"
    assert invokeDemo.someMethod() == "Called invokeMethod someMethod []"

This above code runs without error, because we've implemented the invokeMethod().


Adding methods and properties during runtime
--------------------------------------------------
We can also add methods and properties to a Groovy class during runtime. Let's see how this is done. 

This below code fails for obvious reasons. 

::
    
    package com.rajkaramchedu

    /**
     * Created by RajkMacPro on 9/11/16.
     */
    class TutMetaClass {
    }
    TutMetaClass sid = new TutMetaClass()
    sid.metaClass.name = "Sid"
    sid.whoAreYou = { -> println "$name is a Knight of Sidonia"}
    sid.whoAreYou()

It fails because we are trying to assign a variable "Sid" to a property name that is not defined on sid object, and also we are trying to assign a closure to a method whoAreYou() that is not defined on sid object.

How do we assign methods and properties to our class, then? A quick digression into MetaClass concept will help us out.

MetaClass Concept
---------------------
Groovy supports what is called as a MetaClass. Every Groovy class has a MetaClass that defines the behavior of that Groovy class. Moreover, every MetaClass is an Expando class, and that means we can extend the MetaClass of our TutMetaClass by calling its MetaClass and assigning methods and properties to that MetaClass. This works on an instance basis. 

::

    class TutMetaClass {
    }

    TutMetaClass sid = new TutMetaClass()

    sid.metaClass.name = "Sid"
    sid.metaClass.whoAreYou = { -> println "$name is a Knight of Sidonia"}
    sid.whoAreYou()

This runs okay with the result as below:
::

    Sid is a Knight of Sidonia
    Process finished with exit code 0

DANGER!!
------------
With such a power of assigning methods and properties to the MasterClass of any Groovy class, we need to watchout for easy damage we can do. See below for how easily we can manipulate the String class.


::

    class TutMetaClass {
    }

    TutMetaClass sid = new TutMetaClass()

    sid.metaClass.name = "Sid"
    sid.metaClass.whoAreYou = { -> println "$name is a Knight of Sidonia"}
    sid.whoAreYou()

    //Danger!!
    String.metaClass.loud = { -> toUpperCase() }
    println "I am a Knight of Sidonia".loud()

Results in the below output:
::

    Sid is a Knight of Sidonia
    I AM A KNIGHT OF SIDONIA

    Process finished with exit code 0

Because we manipulated the String class via its MetaClass, every print call will print in upper case. We have to be careful with the power of accessing the MetaClass this way.

Accessing a property not defined on the class
------------------------------------------------

This below code runs fine, as we expect.
::

    package com.rajkaramchedu

    /**
     * Created by RajkMacPro on 9/11/16.
     */

    class PropertyDemo {

        def prop1 = "prop1"
        def prop2 = "prop2"
        def prop3 = "prop3"
    }

    def pd = new PropertyDemo()
    println pd.prop1
    println pd.prop2
    println pd.prop3

and here below is the output.

::

    prop1
    prop2
    prop3
    Process finished with exit code 0


How about accessing a property that's *not* defined on the class? We can do that too, with metaprogramming. We just have to intercept the default getter of the property with our own implementation of getProperty(), as below.

::

    /**
     * Created by RajkMacPro on 9/11/16.
     */

    class PropertyDemo {

        def prop1 = "prop1"
        def prop2 = "prop2"
        def prop3 = "prop3"

        def getProperty(String name) {

            println "Overriden with getProperty() called with argument $name"
        }
    }

    def pd = new PropertyDemo()
    println pd.prop1
    println pd.prop2
    println pd.prop3
    println pd.prop4

The above code runs with the below results.
::

    Overriden with getProperty() called with argument prop1
    null
    Overriden with getProperty() called with argument prop2
    null
    Overriden with getProperty() called with argument prop3
    null
    Overriden with getProperty() called with argument prop4
    null
    Process finished with exit code 0

Because we are routing all property access requests by explicitly implementing our own getProperty() method, all property access requests through it. The "null" output in the output is because the default getter is not returning anything because we are completely bypassing it. 

Accessing Missing Property
-----------------------------

Another way of dealing with accessing missing property is by implementing the propertyMissing() method. 
::

    package com.rajkaramchedu

    /**
     * Created by RajkMacPro on 9/11/16.
     */

    class PropertyDemo {

        def prop1 = "prop1"
        def prop2 = "prop2"
        def prop3 = "prop3"

    //    def getProperty(String name) {
    //
    //        println "Overriden with getProperty() called with argument $name"
    //    }

        def propertyMissing(String name) {

            "Caught missing property: $name"
        }
    }

    def pd = new PropertyDemo()
    println pd.prop1
    println pd.prop2
    println pd.prop3
    println pd.prop4

The above code runs with below results:
::

    prop2
    prop3
    Caught missing property: prop4
    Process finished with exit code 0

Groovy supports this interception for failing properties.

Conclusion
-------------
This concludes our short tutorial on metaprogramming with Groovy. It is a big topic, and we are barely scratching the surface in our tutorial. Please refer to the links below to advance your learning further.

 - `Groovy Language Official Documentation page: <http://groovy-lang.org/documentation.html>`_.
 - `Dan Vega's The Complete Apache Groovy Developer Course: <https://www.udemy.com/apache-groovy/learn/v4/overview>`_.









