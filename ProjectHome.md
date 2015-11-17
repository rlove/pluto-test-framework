# PLUTO #

As in-database languages become increasingly powerful, programmers are beginning to create more and more code that runs within database management systems.  Included in this mix is one of the most popular database management systems in use today, Oracle, along with its proprietary programming language, PL/SQL.

Though there many powerful tools for creating PL/SQL code, there are limited tools and frameworks for properly testing that code.  The PL/SQL Unit Testing for Oracle framework, PLUTO, is the open source, object oriented, xUnit-based system for developing unit tests that attempts to fill the gap in Oracle unit testing.

The framework is written in PL/SQL and is based on the Oracle object system.  The inner-workings of the system are based largely on the JUnit design, with a few PL/SQL'isms added in.  The system is built to be easily extended and has hooks to interface with common test runners like Perl's Test::Harness.

PLUTO is lightweight, easy to install, and is free and open for everyone to use.  Please feel free to give it a spin and comment back with any feedback that you might have.

Interested in learning more, check out the project [wiki](http://code.google.com/p/pluto-test-framework/wiki/PlutoWikiMain).