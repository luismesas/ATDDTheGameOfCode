Let's assume you've already read some ATDD philosophy articles, you're already convinced that it is the rigth way to code but now you want to see it in action.

We'll use (Strajah)[https://github.com/strajah/strajah] as an example. The successor of cipherlayer, a security layer based on Oauth 2. Strajah is build from scratch, so pretty we have some work to do until it can replace cipherlayer, feel free to contribute using your ATDD skills adquired in this tutorial.




Start at the tag https://github.com/strajah/strajah/tree/ATDD-theGameOfCode


# Doing a real registration process

As for now Strajah has a heartbeat and a registration endpoint. Since we're using cucumber for writing the features please have a look at both features. 
The registration feature just expects a 201 response code, and so that's ALL strajah do, sends back a 201 every time it is called. We want to improve that. We want a real registration process.

Adding a second step to check if the customer can log in after registration sounds like a good idea. 

```
  Scenario: Successful registration
    When a not registered user requests to register with data
      | user name | password |
      | Ironman   | Av3ng3Rs |
    Then the response code must be 201
    And the customer is able to log in with his credentials
      | user name | password |
      | Ironman   | Av3ng3Rs |
```    

That looks good enough, problem is, we do not know what it means to be logged in. 

## Login feature

Let's add another feature to describe that. We haven't writen a single line of code yet.

```
  Scenario: Customer logs in successfully
    Given a registrated customer with data
      | user name | password |
      | Ironman   | Av3ng3Rs |
    When the customer "Ironman" logs in with password "Av3ng3Rs"
    Then the response code must be 200
    And the response body has "accessToken" property
```    

For now that should be ok. Now we have a pretty good idea of want we have to do. 
In ATDD we always start by writing the glue code, that is, the step definitions.

### First step: Given a registrated customer with data

The first step is asking us to have a registrated customer in the system. We could archieve that in two ways: adding the customer to the database or by doing a POST to the registration endpoint. 
The first option is usually bad, accessing the data base from the acceptance tests means doing a white box test, which tend to be less maintainable. And... did I mention we don't have a database yet?
For now we are going to do a POST to the registration endpoint.

    request({
        uri: testConfig.publicHost + ':' + testConfig.publicPort + '/api/registration',
        method: 'POST',
        json: true,
        body: {
            'name': registrationData['user name'],
            'password': registrationData['password']
        }
    }, _.partial(saveResponse, this, done));



	function saveResponse(world, done, error, response) {
	    should.not.exist(error);

	    world.publishValue('statusCode', response.statusCode);
	    done();
	}


The only thing that we need to check here is that there aren't any conection errors. And we propagate the response body (via the world object) to the the other verifications later.

### Second step: When the customer "Ironman" logs in with password "Av3ng3Rs"

This step is much like the previous one: get the data provided in the table and do a POST request.
And for now the request body for the login will be

    body: {
        'name': customerName,
        'password': password
    }

Why not encrypt the body? Becouse there's no feature telling us to do so, and we don't like doing extra work.

If we run now the acceptance tests we get passed though step 3 which is red right now:
AssertionError: expected 404 to deeply equal 200

That's because there's no '/api/auth/login' endpoint.


### Third step: Then the response code must be 200

This step is reused from previous features. But it is red, and we have to fix it.

Let's write the endpoint code!! Auch... not yet. Unit test first, please. 

We're only going to test the login middleware, the function which registers the endpoint won't be tested as it is considered covered by the acceptance tests.

To turn this step green we need an 200 response code for the given endpoint.
The first unit test is about calling the next function in restify, if not done the channel remains open and that's bad. 
Using a simple mock object

    it('must call the next function', function(done){
        let res = {
            send: function(){}
        };

        loginMiddleware(null, res, done);
    });

The code to make this unit test pass could be as simple as 

    function login (request, response, next){
        response.send();
        return next();
    }

The next unit test verifies the response code

    it('must send back a response', function(){
        let obtainedStatusCode;

        let res = {
            send: function(value){
                obtainedStatusCode = value;
            }
        };

        loginMiddleware(null, res, function(){});
        obtainedStatusCode.should.equal(200);
    });

To turn it green change the response.send() method adding a 200 argument to it

    response.send(200);

Even though the unit tests are passing we still need to register the endpoint in restify to make this third step green.

    function registerIn (server) {
        server.post('/api/auth/login', login);
    }


There are things that can not be tested by acceptance tests. Like the next function needed in restify. It is an implementation requirement and should be only known to the unit tests.
If we put it into the acceptance tests (like a step!) we would get coupled to the http library used in strajah. 
The idea of acceptance tests for servers (using an API Rest interface) should be to make the glue code of the tests in another language, like Ruby, and they should still be green.

With this the acceptance tests are almost done... One step left.


### Fourth step: And the response body has "accessToken" property

The glue code for this step is pretty easy

    this.getValue('body').should.include.keys(propertyName);

It will fail as for now strajah sends back only a 200 response code, with no body. We write the unit test for this

    it('has an accessToken property', function () {
        let obtainedBody;

        let res = {
            send: function(statusCode, body){
                obtainedBody = body;
            }
        };

        loginMiddleware(null, res, function(){});
        obtainedBody.should.include.keys('accessToken');
    });   

And the login function becomes

    function login (request, response, next){
        let body = {
            accessToken: '123abc'
        };
        response.send(200, body);
        return next();
    }

Instead of the restify method send we are going to use the method json, which does the same thing but also sets a header of content-type: application/json.
For that we must first change the mocks of the unit tests

    let res = {
        json: function(){}
    };

and only after this is done we can change the login function itself.


Basic login feature completed! We have a login endpoint that sends back a 200 response code and a static token. That should be enough for the registration step we started doing in the first place.

Since all tests are green - commit.




## Returning to the registration feature

We left back a step in the registration process

```
And the customer is able to log in with his credentials
  | user name | password |
  | Ironman   | Av3ng3Rs |
```

Since now we know what it means to be logged in let's implement that step. The glue code now is is made up of a POST request to the login endpoint and verification that the response status code is 200. We'll leave to the login feature to check all the rest of the request.

This step doesn't need any extra code, so no unit tests for this one.

## Registration: secondary paths

For now we have a lot a acceptance and unit tests and a simple security layer that doesn't even save the registration data... and always allows login! That doesn't seem like a good security layer at all.

Let's add some secondary paths and see if it gets any better.


### Scenario: Unsuccessful registration - already registered customer with the same name

Now we want to force strajah to store the registered customer so we write the following scenario
We'll reuse most of the steps implemented previously

```
Scenario: Unsuccessful registration - already registered customer with the same name
  Given a registered customer with data
    | user name | password |
    | Ironman   | Av3ng3Rs |
  When a not registered user requests to register with data
    | user name | password     |
    | Ironman   | I'm a clon!  |
  Then the response code must be 401
  And the customer is not able to log in with his credentials
    | user name | password |
    | Ironman   | Av3ng3Rs |
```

First failure we find: AssertionError: expected 201 to deeply equal 401

There no validation of the username at registration. We'll start by saving the customers names and passwords inside some storage as first approximation to solve the problem.

Since the first registration unit tests were writen with promises we'll continue to use them for the following unit tests. For the first one will supose that there's a 'retrieveFromStorage.js' that return all stored customers (without searches or filtering).


    it('Should return 401 when a customer exists with the same name', function () {
        const request = mockRequest(),
            response = mockResponse();

        let deferred = q.defer();
        let promise = deferred.promise;

        let retrieveFromStorageStub = sinon.stub();
        retrieveFromStorageStub.returns(promise);

        let responseSpy = sinon.spy(response, 'json');

        let registrationMiddleware = createRegistrationMiddleware(retrieveFromStorageStub);
        registrationMiddleware(request, response, function () {});

        let fulfilledPromise = promise.then(function () {
            let statusCode = responseSpy.args[0][0];
            statusCode.should.deep.equal(401);
        });

        let storageResponse = [
            {
                name: 'user1'
            }
        ];
        deferred.resolve(storageResponse);

        return fulfilledPromise;
    });


The function createRegistrationMiddleware gets a mock and uses it instead of the 'retrieveFromStorage.js'. 

Keep in mind that we still haven't created the retrieveFromStorage file, but we're mocking it with the interface we want it to have, that is, an array of maps containing a name property each one.

We can modify the registration middleware in order to fulfill the test


    retrieveCustomers().then(function (customers) {
        let filteredCustomers = _.filter(customers, function (retrievedCustomer) {
            return retrievedCustomer.name === request.body.name;
        });

        if (!_.isEmpty(filteredCustomers)) {
            response.json(401);
            return next();
        }
        
        response.json(201, 'ok');
        next();
    });


We have one last thing to add to the registration middleware: the storage of the registrated customers.




