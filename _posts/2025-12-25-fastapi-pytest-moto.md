---
layout: post
show_meta: true
title: Testing FastAPI with pytest and moto
header: Testing FastAPI with pytest and moto
date: 2025-12-25 00:00:00
summary: How to test a FastAPI application using pytest and moto
categories: python fastapi cognito pytest moto testing
author: Chee Yeo
---
[moto library]: https://github.com/getmoto/moto


In the previous series on building a FastAPI application, we integrated OAuth through Cognito into the application. To test the API together with Cognito, we need to be able to stub out the external Cognito API calls using the [moto library].

Firstly, we install the test dependencies via uv using the `dev` flag, which means they will not be installed during the docker build with the `--no-dev` flag:

{% highlight python %}
uv add --dev pytest pytest-cov pytest-mock "moto[all]"
{% endhighlight %}

The first approach is to work out how to mock the AWS Cognito API calls using [moto library]. As per the documentation, I created fake AWS credentials as a fixture and pass it into a separate fixture which uses the `mock_aws` context manager to create an initial `UserPool` and registered user for our tests:

{% highlight python %}
@pytest.fixture(scope="session")
def aws_credentials():
    """Mocked AWS Credentials for moto."""
    os.environ["AWS_ACCESS_KEY_ID"] = "testing"
    os.environ["AWS_SECRET_ACCESS_KEY"] = "testing"
    os.environ["AWS_SECURITY_TOKEN"] = "testing"
    os.environ["AWS_SESSION_TOKEN"] = "testing"
    os.environ["AWS_DEFAULT_REGION"] = "us-west-1"
    os.environ["AWS_REGION"] = "us-west-1"


@pytest.fixture(scope="session")
def mock_cognito_client(aws_credentials):
    with mock_aws():
        cognito_client = boto3.client("cognito-idp", region_name="us-west-1")
        user_pool_id = cognito_client.create_user_pool(PoolName="TestUserPool")["UserPool"]["Id"]
        app_client = cognito_client.create_user_pool_client(
            UserPoolId=user_pool_id,
            ClientName="TestAppClient"
        )

        username = "test_username"
        password = "SecurePassword1234#$%"  # Password must meet security policies.
        email = "test_mail@test.com"

        cognito_client.sign_up(
            ClientId=app_client["UserPoolClient"]["ClientId"],
            Username=username,
            Password=password,
            UserAttributes=[
                {"Name": "email", "Value": email},
            ],
        )
        
        cognito_client.admin_confirm_sign_up(UserPoolId=user_pool_id, Username=username)

        yield AWSCognito(client=cognito_client, region="us-west-1", client_id=app_client["UserPoolClient"]["ClientId"], client_secret="SECRET", user_pool_id=user_pool_id)
{% endhighlight %}

The above creates a mock cognito idp client and uses it to create a user pool, user pool client and registers a test user into the user pool. We yield an instance of `AWSCognito` which is a wrapper around the cognito API. Any reference to this fixture will run in the `mock_aws` context block.

Next, we create a separate `SQLModel Session` for the test database. I'm running a separate postgresql database for the tests which will use separate credentials. The official tutorial suggests using in-memory sqlite3 database for the tests but I prefer the tests to run under the same conditions as the production database. There are 2 issues to overcome. 

Firstly, for creating the test database schema dynamically, we need to use `SQLModel.metadata.create_all` function. We need to import `SQLModel` before the database models else it will not run the migrations.

Secondly, we also need to be able to get the database credentials dynamically either through exported environment variables or through the `.env.test` environment file. As per the `dotenv` library documentation, we merge both `os.environ` and `dotenv_values` into a single dict to construct the psql connection string. This would allow the tests to pick up the environment variables when run locally or via CI/CD pipelines. Finally, we yield the `Session` object:

{% highlight python %}
@pytest.fixture(name="session", scope="session")
def session_fixture():
    config = {
        **os.environ,
        **dotenv_values(".env.test"),
    }

    postgresql_url = f"postgresql://{config.get('RDS_USERNAME')}:{config.get('RDS_PASSWORD')}@{config.get('RDS_HOSTNAME')}:{config.get('RDS_PORT')}/{config.get('RDS_DB_NAME')}"
    
    engine = create_engine(postgresql_url, connect_args={})
    
    SQLModel.metadata.create_all(engine)

    with Session(engine) as session:
        yield session

{% endhighlight %}

Next, we create a corresponding registered user in our test database, using the previous session fixture as a dependency:

{% highlight python %}
@pytest.fixture(name="user", scope="session")
def user_fixture(session):
    db_user = User(
        username="test_username",
        email="test_mail@test.com",
        password=get_password_hash("SecurePassword1234#$%")
    )
    session.add(db_user)
    session.commit()
    session.refresh(db_user)
{% endhighlight %}

FastAPI provides a `TestClient` object which is used to interact with the API in the tests. This client object is a wrapper around the FastAPI application object. We yield this client in the client fixture. Since the application uses a database session as a dependency in its routes, we need to replace it with the session fixture we created earlier. We can use `dependency_overrides` to replace it with the session fixture. We also replace the cognito client dependency with the mock client fixture created earlier. The client fixture is as follows:

{% highlight python %}
@pytest.fixture(name="client", scope="session")
def client_fixture(session, user, mock_cognito_client):
    # Overrides the database dependency to use the test session
    def get_session_override():
        return session
    
    # Overrides the cognito client dependency to use the test client
    def get_aws_cognito_override():
        return mock_cognito_client

    app = create_app()
    app.dependency_overrides[get_session] = get_session_override
    app.dependency_overrides[get_aws_cognito] = get_aws_cognito_override

    client = TestClient(app)
    yield client
    app.dependency_overrides.clear()

    # clear data from test database
    for model in [RandomItem, User]:
        stmt = delete(model)
        session.exec(stmt)
        session.commit()
{% endhighlight %}

We yield the client from the fixture. Once the tests using the fixture is completed, it performs teardown by calling `app.dependency_overrides.clear()`. We also clear up the test database by deleting any existing records for all the models.

Since the API routes are protected with an OAuth dependency, we need to be able to generate a token to pass it in our tests. We create a separate fixture to generate a test token which we can use in our tests that require authentication:

{% highlight python %}
@pytest.fixture()
def token(user, mock_cognito_client, mock_token, client, monkeypatch):
    monkeypatch.setenv("AWS_REGION", mock_cognito_client.region)
    monkeypatch.setenv("AWS_USER_POOL_ID", mock_cognito_client.user_pool_id)
    monkeypatch.setenv("AWS_COGNITO_APP_CLIENT_ID", mock_cognito_client.client_id)
    monkeypatch.setenv("AWS_COGNITO_APP_CLIENT_SECRET", mock_cognito_client.client_secret)

    monkeypatch.setattr(AWSCognito, "decode_token", mock_token)

    resp = client.post("/users/login", data={"username": "test_username", "password": "SecurePassword1234#$%"})
    assert resp.status_code == 200
    token = resp.json()['access_token']
    assert token is not None

    return token
{% endhighlight %}

The token fixture uses some of the fixtures we create previously. It's not possible to export an environment variable using `os.environ` in the fixtures as each test runs in a separate execution thread. We use pytest `monkeypatch` to set the environment. In this example, we are setting some of the environment variables which are required by the AWS Cognito class from the values extracted from the mock cognito client created in the earlier fixture. 

One of the dependencies `get_current_user_cognito` receives the access token after the Cognito authentication and decodes it by fetching the JWKS from Cognito. In our previous post, we created a Lambda trigger to the UserPool which adds the custom user scopes during the Pre Token Generation. We stub the `decode_token` function in the cognito client to return a mock token response which contains those custom scopes. 

Finally, we use the test client fixture to call the login route to obtain a token from the moto service, which will return the mock token when `decode_token` is called.

Using the above fixtures and setup, I was able to test some of the API routes:

{% highlight python %}
def test_get_randoms_not_authenticated(client):
    response = client.get("/randoms")
    assert response.status_code == 401
    assert response.json() == {'detail': 'Not authenticated'}


def test_get_randoms(client, token):
    response = client.get("/randoms", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 200
    assert response.json() == []
{% endhighlight %}

The functional tests for the user routes use a similar approach. For example, to test user signup:

{% highlight python %}
def test_users_signup(client):
    resp = client.post("/users/signup", json={"username": "test", "email": "test@example.com", "password": "SecurePassword1234#$%"})
    assert resp.status_code == 201
    assert resp.json()["message"] == "User created successfully"
    assert "sub" in resp.json().keys()
    sub = resp.json()["sub"]

    resp = client.get(f"/users/{sub}")
    assert resp.status_code == 200
    user = resp.json()
    assert user["sub"] == sub
    assert user["username"] == "test"
    assert user["email"] == "test@example.com"
{% endhighlight %}

In order to test for exceptions thrown by the Cognito client, we use the `mocker` library to return specific exceptions before the actual API call. For example, to test an API response from an exception thrown by Cognito for a registered user that already exists, we could use `mocker` to patch the `user_signup` method to return an exception: 

{% highlight python %}
def test_users_signup_user_exists(mocker, client, mock_cognito_client):
    mocker.patch.object(mock_cognito_client, "user_signup", side_effect=ClientError(error_response={"Error": {"Code": "UsernameExistsException"}}, operation_name="user signup"))
    
    resp = client.post("/users/signup", json={"username": "test", "email": "test@example.com", "password": "SecurePassword1234#$%"})
    
    assert resp.status_code == 409
    assert resp.json() == {"detail": "Account with email exists"}
{% endhighlight %}

Using this approach, I was able to test both the API and Cognito service.

Below is the complete `conftest.py` file as detailed above.
<script src="https://gist.github.com/cheeyeo/0d19dd45ad59b09bf420002d32ecc28e.js"></script>

The full source code can be found at [https://github.com/cheeyeo/FastAPI-application-example/tree/testing](https://github.com/cheeyeo/FastAPI-application-example/tree/testing)