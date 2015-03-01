Let's assume you've already read some ATDD philosophy articles, you're already convinced that it is the rigth way to code but now you want to see it in action.

We'll use (Strajah)[https://github.com/strajah/strajah] as an example. The successor of cipherlayer, a security layer based on Oauth 2. Strajah is build from scratch, so pretty we have some work to do until it can replace cipherlayer, feel free to contribute using your ATDD skills adquired in this tutorial.




Start at the tag https://github.com/strajah/strajah/tree/ATDD-theGameOfCode


# Doing a real registration process

As for now Strajah has a heartbeat and a registration endpoint. Since we're using cucumber for writing the features please have a look at both features. 
The registration feature just wants a 201 response code, and so that's ALL strajah do, sends back a 201 every time it is called. We want to improve that. We want a real registration process.

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
    Then the response status code is 200
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






















