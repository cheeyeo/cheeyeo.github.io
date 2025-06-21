---
layout: post
show_meta: true
title: Testing Flask webapp with login user
header: Testing Flask webapp with login user
date: 2025-06-05 00:00:00
summary: How to test a logged in user in functional tests in Flask
categories: flask python 
author: Chee Yeo
---
[Flask testing documentation]: https://flask.palletsprojects.com/en/stable/testing/
[Flask-Login automated testing]: https://flask-login.readthedocs.io/en/latest/#automated-testing
[FlaskLoginClient source code]: https://github.com/maxcountryman/flask-login/blob/main/src/flask_login/test_client.py


While learning how to build Flask web application with logged in users, I used the `Flask-Login` package to handle the registration and authentication process. However, I was having issues trying to create a current user in the context of functional tests for some of the routes which were protected by the `login_required` decorator.

This post assumes some working knowledge of testing Flask web applications with pytest.

Firstly, we need to create a test client from the application under test. This is defined in the `conftest.py` file for pytest. The test client can be created via `app.test_client()` method call. This is the same client which we will be using for our functional tests in order to make requests to application routes. 

There are 2 main approaches to creating a valid session user:

* Setting the current user via the session object without logging in

* Using the `FlaskLoginClient` object from `Flask-Login` package


#### Setting the current user via the session object without logging in

`Flask-Login` works by storing the current user's ID into the session object which is retrieved during authentication to perform a database lookup of the user. Using the test client's `session_transaction()` method, we can do a database lookup for the test user and add its database ID into the session directly before making any actual test requests. This approach is detailed in [Flask testing documentation]. Below is a snippet of my setup in `conftest.py`:

{% highlight python %}
@pytest.fixture(scope='session')
def client(app):
    return app.test_client()


@pytest.fixture(scope='session')
def test_user(app):
    with app.app_context():
        user =  Database.session.scalar(
            Database.select(User).where(User.email == "test@example.com")
        )

    return user


@pytest.fixture(scope='function')
def login_user_via_session(app, client, test_user):
    with client.session_transaction() as session:
        session['_user_id'] = test_user.id
    
    yield client

    client.get('/logout')
{% endhighlight %}

Firstly, we create a test client using `app.test_client()`. Next, we fetch the test user using the test application context and set it to `session` scope so its available for the entire test run. Lastly, we create a new fixture of `login_user`. This fixture takes in the app, client and test_user fixtures and sets the session object with the test user ID. Note that the fixture is set to `function` scope rather than `session` scope. This means that the session is modified for each test case that requires an authenticated user. For test cases that don't use the fixture, the session remains unchanged. This allows us to test for redirects for unauthorized access. We yield the test client after updating the session and finally, we call '/logout' to clear the session.

A typical function test that uses the above would be to access the home page which requires a current user and check that the output indicates the user is logged in:

{% highlight python %}
def test_login_user(login_user_via_session):
    resp = login_user_via_session.get("/")
    assert b'/user/chee' in resp.data
    assert b'Logout' in resp.data
    assert b'Say something' in resp.data
{% endhighlight %}

#### Using the FlaskLoginClient object from Flask-Login package

The second approach is detailed in [Flask-Login automated testing]. This requires setting the `app.test_client_class` to `FlaskLoginClient` and passing in the test user as an argument:

{% highlight python %}
@pytest.fixture(scope='function')
def login_user(app, test_user):
    app.test_client_class = FlaskLoginClient

    with app.test_client(user=test_user) as client:
        yield client
        client.get('/logout')
{% endhighlight %}

This works exactly the same way as method 1 but requires the test client to be run in a block. We set the scope to `function` in order to test unauthorized routes. The same functional test from before will work the same way:

{% highlight python %}
def test_login_user(login_user):
    resp = login_user.get("/")
    assert b'/user/chee' in resp.data
    assert b'Logout' in resp.data
    assert b'Say something' in resp.data
{% endhighlight %}

In the [FlaskLoginClient source code], `FlaskLoginClient` is a subclass of `FlaskClient` and calls the test client `session_transaction()` method to set the user id, which is identical to the first method as discussed previously.