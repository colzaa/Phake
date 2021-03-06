*******************
Method Verification
*******************

The ``Phake::verify()`` method is used to assert that method calls have been
made on a mock object that you can create with ``Phake::mock()``.
``Phake::verify()`` accepts the mock object you want to verify calls against.
Mock objects in Phake can almost be viewed as a tape recorder. Any time the code you are testing
calls a method on an object you create with ``Phake::mock()`` it is going to
record the method that you called along with all of the parameters used to call that method. Then
``Phake::verify()`` will look at that recording and allow you to assert whether
or not a certain call was made.

.. code-block:: php

    class PhakeTest1 extends PHPUnit_Framework_TestCase
    {
        public function testBasicVerify()
        {
            $mock = Phake::mock('MyClass');

            $mock->foo();

            Phake::verify($mock)->foo();
        }
    }

The ``Phake::verify()`` call here, verifies that the method ``foo()`` has been called once (and only once) with no
parameters on the object ``$mock``. A very important thing to note here that is a departure from most (if not all)
other PHP mocking frameworks is that you want to verify the method call AFTER the method call takes place. Other
mocking frameworks such as the one built into PHPUnit depend on you setting the expectations of what will get called
prior to running the system under test.

Phake strives to allow you to follow the four phases of a unit test as laid out in xUnit Test Patterns: setup,
exercise, verify, and teardown. The setup phase of a test using Phake for mocking will now include calls to
``Phake::mock()`` for each class you want to mock. The exercise portion of your code will remain the same. The verify
section of your code will include calls to ``Phake::verify()``. The exercise and teardown phases will remain unchanged.

Verifying Method Parameters
===========================

Verifying method parameters using Phake is very simple yet can be very flexible. There are a wealth of options for
matching parameters that are discussed later on in :ref:`method-parameter-matchers-section`.

Verifying Multiple Invocations
==============================

A common need for mock objects is the ability to have variable multiple invocations on that object. Phake allows you to use
``Phake::verify()`` multiple times on the same object. A notable difference between Phake and PHPUnit’s mocking
framework is the ability to mock multiple invocations of the same method with no regard for call sequences. The PHPUnit
mocking test below would fail for this reason.

.. code-block:: php

    class MyTest extends PHPUnit_Framework_TestCase
    {
        public function testPHPUnitMock()
        {
            $mock = $this->getMock('PhakeTest_MockedClass');

            $mock->expects($this->once())->method('fooWithArgument')
                    ->with('foo');

            $mock->expects($this->once())->method('fooWithArgument')
                    ->with('bar');

            $mock->fooWithArgument('foo');
            $mock->fooWithArgument('bar');
        }
    }

The reason this test fails is because by default PHPUnit only allows a single expectation per method. The way you can
fix this is by using the `at()` matcher. This allows you to specify the index of the invocation you want to match
again. So to make the test above work you would have to change it.

.. code-block:: php

    class MyTest extends PHPUnit_Framework_TestCase
    {
        public function testPHPUnitMock()
        {
            $mock = $this->getMock('PhakeTest_MockedClass');

            //NOTICE this is now at() instead of once()
            $mock->expects($this->at(0))->method('fooWithArgument')
                    ->with('foo');

            //NOTICE this is now at() instead of once()
            $mock->expects($this->at(1))->method('fooWithArgument')
                    ->with('bar');

            $mock->fooWithArgument('foo');
            $mock->fooWithArgument('bar');
        }
    }

This test will now run as expected. There is still one small problem however and that is that you are now testing not
just the invocations but also the order of invocations. Many times the order in which two calls are made really do not
matter. If swapping the order of two method calls will not break your application then there is no reason to enforce
that code structure through a unit test. Unfortunately, you cannot have multiple invocations of a method in PHPUnit
without enforcing call order. In Phake these two notions of call order and multiple invocations are kept completely
distinct. Here is the same test written using Phake.

.. code-block:: php

    class MyTest extends PHPUnit_Framework_TestCase
    {
        public function testPHPUnitMock()
        {
            $mock = Phake::mock('PhakeTest_MockedClass');

            $mock->fooWithArgument('foo');
            $mock->fooWithArgument('bar');

            Phake::verify($mock)->fooWithArgument('foo');
            Phake::verify($mock)->fooWithArgument('bar');
        }
    }

You can switch the calls around in this example as much as you like and the test will still pass. You can mock as many
different invocations of the same method as you need.

If you would like to verify the exact same parameters are used on a method multiple times (or they all match the same
constraints multiple times) then you can use the verification mode parameter of ``Phake::verify()``. The second
parameter to ``Phake::verify()`` allows you to specify how many times you expect that method to be called with matching
parameters. If no value is specified then the default of one is used. The other options are:

* ``Phake::times($n)`` – Where ``$n`` equals the exact number of times you expect the method to be called.
* ``Phake::atLeast($n)`` – Where ``$n`` is the minimum number of times you expect the method to be called.
* ``Phake::atMost($n)`` – Where ``$n`` is the most number of times you would expect the method to be called.
* ``Phake::never()`` - Same as calling ``Phake::times(0)``.

Here is an example of this in action.

.. code-block:: php

    class MyTest extends PHPUnit_Framework_TestCase
    {
        public function testPHPUnitMock()
        {
            $mock = Phake::mock('PhakeTest_MockedClass');

            $mock->fooWithArgument('foo');
            $mock->fooWithArgument('foo');

            Phake::verify($mock, Phake::times(2))->fooWithArgument('foo');
        }
    }

Verifying Calls Happen in a Particular Order
============================================

Sometimes the desired behavior is that you verify calls happen in a particular order. Say there is a functional reason
for the two variants of ``fooWithArgument()`` to be called in the order of the original test. You can utilize
``Phake::inOrder()`` to ensure the order of your call invocations. ``Phake::inOrder()`` takes one or more arguments and
errors out in the event that one of the verified calls was invoked out of order. The calls don’t have to be in exact
sequential order, there can be other calls in between, it just ensures the specified calls themselves are called in
order relative to each other. Below is an example Phake test that behaves similarly to the PHPUnit test that utilized
``at()``.

.. code-block:: php

    class MyTest extends PHPUnit_Framework_TestCase
    {
        public function testPHPUnitMock()
        {
            $mock = Phake::mock('PhakeTest_MockedClass');

            $mock->fooWithArgument('foo');
            $mock->fooWithArgument('bar');

            Phake::inOrder(
                Phake::verify($mock)->fooWithArgument('foo'),
                Phake::verify($mock)->fooWithArgument('bar')
            );
        }
    }

Verifying No Interaction with a Mock so Far
===========================================

Occasionally you may want to ensure that no interactions have occurred with a mock object. This can be done
by passing your mock object to ``Phake::verifyNoInteraction($mock)``. This will not prevent further interaction
with your mock, it will simply tell you whether or not any interaction up to that point has happened. You
can pass multiple arguments to this method to verify no interaction with multiple mock objects.

Verifying No Further Interaction with a Mock
============================================

There is a similar method to prevent any future interaction with a mock. This can be done by passing a mock
object to ``Phake::verifyNoFurtherInteraction($mock)``. You can pass multiple arguments to this method to
verify no further interaction occurs with multiple mock objects.

Verifying Magic Methods
=======================

Magic methods are commonly used in PHP and the need to be able to seamlessly utilize them is always necessary. Most
magic methods can be verified using the method name just like you would any other method. The one exception to this is
the ``__call()`` method. This method is overwritten on each mock already to allow for the fluent api that Phake
utilizes. If you want to verify a particular invocation of ``__call()`` you can verify the actual method call by
mocking the method passed in as the first parameter.

Consider the following class.

.. code-block:: php

    class MagicClass
    {
        public function __call($method, $args)
        {
            return '__call';
        }
    }

You could mock an invocation of the `__call()` method through a userspace call to magicCall() with the following code.

.. code-block:: php

    class MagicClassTest extends PHPUnit_Framework_TestCase
    {
        public function testMagicCall()
        {
            $mock = Phake::mock('MagicClass');

            $mock->magicCall();

            Phake::verify($mock)->magicCall();
        }
    }

If for any reason you need to explicitly verify calls to ``__call()`` then you can use ``Phake::verifyCallMethodWith()``.

Verifying Static Methods
========================
Using Phake you can verify polymorphic calls to static methods using ``Phake::verifyStatic()``. It is important to note
that you cannot verify ALL calls. In order for Phake to understand that a method has been called, it needs to be able to
intercept the call so that it can record it. Consider the following class

.. code-block:: php

    class StaticCaller
    {
        public function callStaticMethod()
        {
            Foo::staticMethod();
        }
    }

You will not be able to verify the call to Foo::staticMethod() because the call was made directly on the class. This
prevents Phake from seeing that the call was made. However, say you have an abstract class that has an abstract static
method.

.. code-block:: php

    abstract class StaticFactory
    {
        abstract protected static function factory();

        public static function getInstance()
        {
            return static::factory();
        }
    }

In this case, because the ``static::`` keyword will cause the called class to be determined at runtime, you will be able
to test that the factory call was made.

.. code-block:: php

    class StaticFactoryTest extend PHPUnit_Framework_TestCase
    {
        public function testGetInstance()
        {
            $factory = Phake::mock('StaticFactory');

            $factory::getInstance();

            Phake::verifyStatic($factory)->factory();
        }
    }

It is important to note that if self::factory() was called the above test would not work, because again the class is
determined at compile time with the self:: keyword. The key thing to remember with testing statics using Phake is that
you can only test statics that leverage Late Static Binding:
http://www.php.net/manual/en/language.oop5.late-static-bindings.php
