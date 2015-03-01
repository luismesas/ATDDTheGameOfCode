Let's assume you've already read some ATDD philosophy articles, you're already convinced that it is the rigth way to code but now you want to see it in action.

We'll use (Strajah)[https://github.com/strajah/strajah] as an example. The successor of cipherlayer, a security layer based on Oauth 2. Strajah is build from scratch, so pretty we have some work to do until it can replace cipherlayer, feel free to contribute using your ATDD skills adquired in this tutorial.




Start at the tag https://github.com/strajah/strajah/tree/ATDD-theGameOfCode


# Doing a real registration process

As for now Strajah has a heartbeat and a registration endpoint. Since we're using cucumber for writing the features please have a look at both features. 
The registration feature just expects a 201 response code, and so that's ALL strajah do, sends back a 201 every time it is called. We want to improve that. We want a real registration process.

Adding a second step to check if the customer can log in after registration sounds like a good idea. 

  Scenario: Successful registration
    When a not registered user requests to register with data
      | user name | password |
      | Ironman   | Av3ng3Rs |
    Then the response code must be 201
    And the customer is able to log in with his credentials
      | user name | password |
      | Ironman   | Av3ng3Rs |
    

That looks good enough, problem is, we do not know what it means to be logged in. 

## Login feature

Let's add another feature to describe that. We haven't writen a single line of code yet.

  Scenario: Customer logs in successfully
    Given a registrated customer with data
      | user name | password |
      | Ironman   | Av3ng3Rs |
    When the customer "Ironman" logs in with password "Av3ng3Rs"
    Then the response code must be 200
    And the response body has "accessToken" property
    

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




















