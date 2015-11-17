# Introduction #

Unit testing your PL/SQL from inside Oracle doesn't have to be difficult or expensive.  There is no need to be tied into some third-party closed source tool and no reason to start from scratch writing your own framework.  PLUTO can help get you started on the road to unit testing bliss, or at least unit testing.

# Installing PLUTO #

Installation couldn't be much easier, just [download the latest version](http://code.google.com/p/pluto-test-framework/downloads/list) of the PLUTO source to a machine that can connect to your database, unpack the source file, and run:

```
  $> sqlplus @install.sql
```

It is recommended that you run the command in a schema called PLUTO and that the PLUTO schema have 'create type' privileges.

If you have any issues with the install, please be sure to [log them](http://code.google.com/p/pluto-test-framework/issues/list).  Be sure to mention the version of Oracle that you are running.  And better yet, if you figured out how to fix the issue, please send in the patch.

## What is installed? ##

PLUTO is intended to be a lightweight system for unit testing.  It doesn't create any tables or store a lot of data.  Instead, it creates a TYPE and a handful of objects.

| pluto\_proc\_name\_tab | A type equivalent to a varchar2(30) |
|:-----------------------|:------------------------------------|
| pluto\_output\_obj     | An object used for the default output from the testing framework.  This object can be extended to accommodate just about any desired output format. |
| pluto\_output\_tap\_obj | An example of extending the default pluto\_output\_obj.  This custom output object writes results in TAP, the Test Anything Protocol used by the [Perl](http://www.perl.org) testing system, Test::Harness. |
| pluto\_util\_obj       | Utility methods for test assertions. |
| pluto\_obj             | The core package for the testing framework.  This object is the object that custom test classes extend in order to get the benefit of the PLUTO interpretation of xUnit |



# Getting Started Unit Testing With PLUTO #

After installing PLUTO, you are ready to start writing unit tests.  You might be interested in some [test examples](http://pluto-test-framework.googlecode.com/svn/trunk/examples/).  The text below highlights what you'll see in some of that example code.

To get started using PLUTO, you just have to subclass the main PLUTO object, pluto\_obj.  To do this in Oracle, you'll need to create a package specification.

```
  create or replace type testing_obj under pluto_obj(
  )
  instantiable not final;
```

Of course, this specification doesn't really accomplish much other than to create a child class of _pluto\_obj_.  We won't realize how powerful that is until a little later in this tutorial.

Adding some meat to the object, let's include a specification for a constructor.

```
  create or replace type testing_obj under pluto_obj(
    constructor function testing_obj
      return self as result
  )
  instantiable not final;
```

The implementation of the constructor simply creates a utility object and an output object.  The utility object is stashed away in _m\_util\_object_.  _m\_util\_object_ is an attribute of _pluto\_obj_ that is intended to store the utility object.  Typically, this part of the test class construction will be very similar to what you see below, with the exception being the type of output object constructed and passed to the utility object constructor.

The type of output object chosen at this point of your testing is very important.  It determines what type of data will be emitted from the testing suite at runtime, be it the default human-readable data, TAP, inserts into a logging table, or some combination of all of the above.  In the case below, we settle on using the default human-readable form of output object.

```
  create or replace type body testing_obj is
    constructor function testing_obj
      return self as result is
    begin
      m_util_object              :=
                     pluto_util_obj( output_object => pluto_output_obj( ));
      return;
    end testing_obj;
  end;
```

Excellent, we now have the minimum amount of code necessary to run unit tests through PLUTO, but we have a problem... have you spotted it?

We don't have any tests!

Let's add a test to our new testing object.

```
  create or replace type testing_obj under pluto_obj(
    member procedure test_one,
    constructor function testing_obj
      return self as result
  )
  instantiable not final;
```

```
  create or replace type body testing_obj is
    member procedure test_one is
    begin
      m_util_object.ok(
        test_passed    => true,
        test_label     => 'running test_one'      ||
                          chr( 10 )               ||
                          'i hope that it worked'
      );
    end test_one;
    constructor function testing_obj
      return self as result is
    begin
      m_util_object              :=
                     pluto_util_obj( output_object => pluto_output_obj( ));
      return;
    end testing_obj;
  end;
```

Okay, so it's not the most exciting test, but it works.  You can see in the first part of the listing that we add a procedure specification called _test\_one_.  Later, we define that procedure in the type body.  In the definition we invoke the _ok_ method from the utility object, passing it a boolean value for the status of the test and a test label.

In order to run the test, we simply run an anonymous PL/SQL block.

```
  declare
    ut_obj  my_test_obj := my_test_obj( );
  begin
    ut_obj.run_tests;
  end;
```

This executes the _test\_one_ test and prints the results to the standard Oracle output stream.  Of course, there is a lot more to PLUTO than simply running test methods one after another.  PLUTO implements xUnit-style testing, which we'll go over shortly.  Also, there is a host of functions and procedures available in the PLUTO utility module.

# Order of Execution Within PLUTO #

The basic idea of the PLUTO framework is to extend the _pluto\_obj_ object, create procedures that implement tests, and call _run\_tests_ to execute those tests.  Of course, there is a little more to it than that.  Not every procedure is a test procedure and not all test procedures are created equally.

PLUTO introspects a given testing object and looks for procedures that are called out as tests.  Procedures are considered tests if they begin with the letters _TEST_.  PLUTO finds all procedures prefixed with _TEST_ and executes them in the order that they sort (ascending) according to the language settings of the current Oracle session.

Sometimes you need to do a little work before you get started testing though.  Often there is a fixture that needs to execute that performs an expensive operation such as creating data structures or seeding data for the tests that are about to be performed.  There is a good chance that you only really want to do most of these operations once per test run.  To accomplish this, PLUTO looks for procedures that begin with the word _STARTUP_.  Procedures prefixed with _STARTUP_ execute before any _TEST_ procedures execute.  If there are multiple _STARTUP_ procedures, they are executed in ascending order according to the language settings of the current Oracle session.

Of course, if you are creating objects, seeding data, or doing whatever wild and crazy test preparation at startup, you'll want to clean things up after you are finished testing.  In many cases, you could probably wrap your tests in a transaction an simply rollback after the tests are complete.  However, you can also take advantage of _SHUTDOWN_ methods in PLUTO.  These methods begin with the word _SHUTDOWN_ and are executed after all _TEST_ procedures are complete.  You probably guessed it by now...  if there are multiple _SHUTDOWN_ procedures, they are executed in ascending order according to the language settings of the current Oracle session.

Great!  We can now run our tests and wrap all of the tests in some startup and shutdown logic that makes sure that we are testing in a sane/standard/regular/regulated environment.  But what if we need to do some sort of setup before each tests runs?  Maybe we need to clean up some data that the test before might have changed or possibly reset some counter or buffer.  _SETUP_ procedures are there for you.

Any procedure that starts with the word _SETUP_ is executed before each _TEST_ procedure that is executed.  This execution happens after any _STARTUP_ procedures run.  If there are multiple _SETUP_ procedures, they are executed in ascending order according to the language settings of the current Oracle session.

Of course, _SETUP_ wouldn't be complete without _TEARDOWN_.  _TEARDOWN_ procedures start with the word _TEARDOWN_ and are executed after each _TEST_ procedure.  If there are multiple _TEARDOWN_ procedures, they are executed in ascending order according to the language settings of the current Oracle session.

Any procedure that doesn't start with the word _STARTUP_, _SETUP_, _TEST_, _TEARDOWN_, or _SHUTDOWN_ is ignored by PLUTO.

If you are having issues remembering the order or figuring out how often each procedure runs just remember that each _STARTUP_ procedure runs once and runs before anything else.    Similarly, each _SHUTDOWN_ procedure runs once and runs after everything else.  Each _TEST_ method runs once and all of the test methods are sandwiched in between the _STARTUP_ and _SHUTDOWN_ procedure(s).  The _SETUP_ and _TEARDOWN_ methods are the ones that get the most action typically.  Each procedure is ran one time for each _TEST_ procedure.  _SETUP_ runs before each test and _TEARDOWN_ afterward.

To illustrate, here is an example of the order of execution of a set of imaginary procedures in a PLUTO object:

  * STARTUP\_SYSTEM
  * SETUP\_FOR\_TESTING
  * TEST\_ONE
  * TEARDOWN\_AFTER\_TESTING
  * SETUP\_FOR\_TESTING
  * TEST\_TWO
  * TEARDOWN\_AFTER\_TESTING
  * SHUTDOWN\_SYSTEM