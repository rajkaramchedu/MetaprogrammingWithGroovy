=====
Metaprogramming Tutorial Draft
=====

Contents:

.. toctree::
   :maxdepth: 2

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`


Groovy (is a dynamic language that) supports metaprogramming capabilities to make changes to the program itself during runtime. What are these changes? Changes like adding methods and properties, intercepting method calls, getting and setting properties. The principal difference between Groovy and a language that does not support metaprogramming is that with Groovy it is possible to adapt the program during runtime.

When a Groovy programmer wants to infuse a capability into a Groovy program such that this program can change itself during runtime or compile time, then this programmer would make explicit use of metaprogramming features supported by Groovy. 

Groovy compiler and runtime engine ("engine") follows a particular flow in handling what's written in a Groovy program. In particular, if the engine sees a specific kind of a method call  <> The entire Groovy system is built to support this metaprogramming capability: the Groovy language has meta classes, meta objects and the Groovy engine itself will follow a specific set of rules. At the same time, this engine imposes certain obligations on the programmer to use these language features in a certain way. <expectations>  This combination of meta classes, meta objects and the rules followed by Groovy engine is pulled together into a concept known as MOP (meta object protocol). In other words, the Groovy engine says, "Here are metaprogramming features the Groovy language has, and when I see one of these in your program, I will consult the MOP and this MOP will then tell me what rules I should follow."

When you write a Groovy metaprogram, you are doing two things: you are writing your Groovy program, and you are also designing your own variation of MOP, i.e., you are customizing the MOP. 

Groovy engine can call a method even if this method is not in a Groovy object, or it can get and set a property, etc. Each such 

First thing you have to do is to learn how to write your own MOP.  Remember MOP is a protocol that dictates the behavior of Groovy engine. The way you can customize this behavior is to put in a few hooks into the MOP. But to do this, we need to understand a few basics of a Groovy class and a Groovy object. 

A) RUN TIME METAPROGRAMMING

(1) In Groovy, there's an interface called  GroovyObject. It is defined as follows.

:: public interface GroovyObject{
	
	MetaClass getMetaClass();
	Object getProperty(String propertyName)
	Object invokeMethod(String name, Object args)
	void setMetaClass(MetaClass metaClass)
	void setProperty(String propertyName, Object newValue)
}

Every Groovy object implements this GroovyObject interface. 
For example, if you compile Test.groovy that has this code:
::
class Sidonia {
}

it will result in Sidonia.class that has
::public class Sidonia implements GroovyObject {...}


2) Now let's see how we can call a method even if that method doesn't exist
2.1) ---method exists, will run okay--
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
-----
2.2) method does not exist, call to method fails
However, if we try to invoke a method such as someMethod(), which is not defined in the class, it will fail, as we expect it should. 

def invokeDemo = new InvokeDemo()
assert invokeDemo.test() == "Method exists"
assert invokeDemo.someMethod() == "Called invokeMethod someMethod()"

The above code will fail with the error message:

Caught: groovy.lang.MissingMethodException: No signature of method: com.rajkaramchedu.InvokeDemo.someMethod() is applicable for argument types: () values: []

2.3) method does not exist, call to method does not fail with metaprogramming

-----------
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
-----------
This above code runs okay because we've implemented the invokeMethod() <insert the bottom flow chart section only>

Note: Did not cover methodMissing() method. 

2.3.1) We can add methods and properties to a Groovy class during runtime. Let's see how this is done. This below code fails for obvious reasons because we are not using any metaprogramming protocol. 

-----
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
---
It fails because we are trying to assign a variable "Sid" to a property name that is not defined on sid object, and also we are trying to assign a closure to a method whoAreYou() that is not defined on sid object.

2.3.2) Groovy supports what is called as a MetaClass. Every Groovy class has a MetaClass that defines the behavior of that Groovy class. Moreover, every MetaClass is an Expando class, and that means we can extend the MetaClass of our TutMetaClass by calling its MetaClass and assigning methods and properties to that MetaClass. This works on an instance basis. 

---
class TutMetaClass {
}

TutMetaClass sid = new TutMetaClass()

sid.metaClass.name = "Sid"
sid.metaClass.whoAreYou = { -> println "$name is a Knight of Sidonia"}
sid.whoAreYou()
--- 
This runs okay with the result as below:

Sid is a Knight of Sidonia
Process finished with exit code 0

2.3.3) DANGER!!

class TutMetaClass {
}

TutMetaClass sid = new TutMetaClass()

sid.metaClass.name = "Sid"
sid.metaClass.whoAreYou = { -> println "$name is a Knight of Sidonia"}
sid.whoAreYou()

//Danger!!
String.metaClass.loud = { -> toUpperCase() }
println "I am a Knight of Sidonia".loud()
-----
Results in the below output:

Sid is a Knight of Sidonia
I AM A KNIGHT OF SIDONIA

Process finished with exit code 0
------
Because we manipulated the String class via its MetaClass. We have to be careful with the power of accessing the MetaClass. 


2.4) accessing the property defined in the class. works fine.

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
---
the above code runs fine with the below output
prop1
prop2
prop3
Process finished with exit code 0

2.5) How about accessing a property that's not defined on the class? We can do that with metaprogramming. We just have to intercept the default getter of the property with our own implementation of getProperty(), as below.
ackage com.rajkaramchedu

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
-----
The above code runs with the below results.
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

2.6) Another way of dealing with accessing missing property is by implementing the propertyMissing() method. 
----
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
--------
The above code runs with below results:
prop2
prop3
Caught missing property: prop4
Process finished with exit code 0

Groovy supports this interception for failing properties.

B) COMPILE TIME METAPROGRAMMING










