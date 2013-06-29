---
layout: post
title: "Faster Django tests"
date: 2013-06-29 11:41
comments: true
categories: django performance testing
---
Tests in django can quickly get annoyingly slow as you build up more and more tests. Two thing I found that significantly improves performance are:

1. Switching the test database to something like sqlite.
2. Run test concurrently.

Both of these can be archived fairly easily.

##Using different database for testing

Using sqlite during testing is fairly easy. This is specified in [answer 1](http://stackoverflow.com/questions/3799061/speeding-up-django-testing) and [answer 2](http://stackoverflow.com/questions/3096148/how-to-run-djangos-test-database-only-in-memory). Since `sqlite` is now part of the python standard library you don't need to install any thing new. In your `settings.py` file add
```python
#Django 1.3 and 1.4:

if 'test' in sys.argv:
    DATABASES['default'] = {'ENGINE': 'django.db.backends.sqlite3'}
```

This significantly increases the performance of tests. According to `answer 2` django automatically creates the databases in memory if the database used is in `sqlite`.

##Running tests concurrently

I had been looking for a way to do this for quiet a while now. Then recently an [article](http://coreygoldberg.blogspot.ca/2013/06/python-concurrencytest-running.html) on [Pycoder Weekly](http://pycoders.com/) about running tests concurrently set me on the right track.

First you will have to install [concurrencytest](https://github.com/cgoldberg/concurrencytest):

```sh
$ pip install concurrencytest
```
Now to customize it for django add this class and related imports in a path that is accessible to django.

```python
from django.test.simple import DjangoTestSuiteRunner
from django.utils import unittest

from concurrencytest import ConcurrentTestSuite, fork_for_tests

class MyTestSuiteRunner(DjangoTestSuiteRunner):
    def run_suite(self, suite, **kwargs):
        return unittest.TextTestRunner(
            verbosity=self.verbosity, failfast=self.failfast).run(
                ConcurrentTestSuite(suite, fork_for_tests(4)))
```

Also update your `settings.py` file to set this as the `test runner` like so.
```python
TEST_RUNNER = 'path.to.MyTestSuiteRunner'
```

This has some covets though, each test case should be completely independent of each other. If you have some external file or resource that is shared read and written on to, this could case some of the tests to fail.
