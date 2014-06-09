---
layout: post
title:  "Combining Slim with Doctrine 2"
date:   2014-06-09 12:00:00
tags:   PHP, Rest, Doctrine2, ORM
published: true
---

###Introduction
[Slim](http://www.slimframework.com/) is a lightweight framework for building restful APIs. With its intuitive routing mechanism one can quickly setup
a restful app.
In case you need to work with [Doctrine](http://www.doctrine-project.org/) as the ORM of choice the following post should be an introduction on integrating the popular ORM with Slim.
Before we start adding Doctrine, let us have a look at the original basic implementation via the Slim documentation and how easy it is to create a GET on a resource:

```php
<?php
$app = new \Slim\Slim();
$app->get('/hello/:name', function ($name) {
    echo "Hello, $name";
});
$app->run();
```

###Setting up Doctrine
Adding Doctrine is really straight forward. Doctrine comes with an excellent documentation, for more details on configuring Doctrine also visit the detailed
[documentation](http://doctrine-orm.readthedocs.org/en/latest/reference/configuration.html).
Integrating the Doctrine library should be really easy when working with [Composer](https://getcomposer.org/).

```
{
	"require": {
		"slim/slim": "2.*",
		"doctrine/orm": "2.4.2"
	},

	// alternatively add path for autoloading
	// depends on how autoloading is handled in your application
	"autoload": {
		"psr-4": {
			"App\\": "src/App"
		}
	}
}
```

Otherwise add the library manually.

###Adding the EntityManager
We will create a new file inside the _**src/App/**_ folder called _**AbstractResource.php**_ :

```php
<?php

namespace App;

use Doctrine\ORM\EntityManager;
use Doctrine\ORM\Tools\Setup;

abstract class AbstractResource
{
	/**
	 * @var \Doctrine\ORM\EntityManager
	 */
	private $entityManager = null;

	/**
	 * @return \Doctrine\ORM\EntityManager
	 */
	public function getEntityManager()
	{
		if ($this->entityManager === null) {
			$this->entityManager = $this->createEntityManager();
		}

		return $this->entityManager;
	}

	/**
	 * @return EntityManager
	 */
	public function createEntityManager()
	{
		$path = array('Path/To/Entity');
		$devMode = true;

		$config = Setup::createAnnotationMetadataConfiguration($path, $devMode);

		// define credentials...
		$connectionOptions = array(
			'driver'   => '',
			'host'     => '',
			'dbname'   => '',
			'user'     => '',
			'password' => '',
		);

		return EntityManager::create($connectionOptions, $config);
	}
}

```

For more details on the configuration options read the [documentation](http://doctrine-orm.readthedocs.org/en/latest/reference/configuration.html).

###Adding an Entity

Next we will create an initial Entity, create a new directory _**src/App/Entity**_ and add a new file entitled _**User.php**_ and add the following code:

```php
<?php

namespace App\Entity;

use App\Entity;
use Doctrine\ORM\Mapping;

/**
 * @Entity
 * @Table(name="users")
 */
class User
{
	/**
	 * @var integer
	 *
	 * @Id
	 * @Column(name="id", type="integer")
	 * @GeneratedValue(strategy="AUTO")
	 */
	protected $id;

	/**
	 * @var string
	 * @Column(type="string", length=64)
	 */
	protected $name;

	/**
	 * @var string
	 * @Column(type="string", length=255)
	 */
	protected $email;

	// Define setters/getters for all properties...
}
```
For more information on Entities read the [documentation](http://doctrine-orm.readthedocs.org/en/latest/reference/working-with-objects.html).


###Creating a UserResource

Let's create a new directory _**src/App/Resource**_ and add a new file entitled _**UserResource.php**_:

```php
<?php

namespace App\Resource;

use App\AbstractResource;
use App\Entity\User;

/**
 * Class Resource
 * @package App
 */
class UserResource extends AbstractResource
{

	/**
	 * @param $id
	 *
	 * @return string
	 */
	public function get($id)
	{
		if ($id === null) {
			$users = $this->getEntityManager()->getRepository('App\Entity\User')->findAll();
			$users = array_map(function($user) {
				return $this->convertToArray($user); },
				$users);
			$data = json_encode($users);
		} else {
			$data = $this->convertToArray($this->getEntityManager()->find('App\Entity\User', $id));
		}

		// @TODO handle correct status when no data is found...

		return json_encode($data);
	}

	// POST, PUT, DELETE methods...

	private function convertToArray(User $user) {
        return array(
            'id' => $user->getId(),
            'name' => $user->getName(),
            'email' => $user->getEmail()
        );
    }
```

UserResource class should handle specific data, for example a PUT would be handled similar to this:

```php
<?php
public function put($id)
{
	$app = \Slim\Slim::getInstance();

	$name = $app->request()->params('name');
	$email = $app->request()->params('email');

	// handle if $id is missing or $name or $email are valid etc.
    // return valid status code or throw an exception
    // depends on the concrete implementation

	/** @var User $user */
	$user = $this->getEntityManager()->find('App\Entity\User', $id);
	// also check if $user has been found else handle correctly

	$user->setEmail($email);
	$user->setName($name);

	$this->getEntityManager()->persist($user);
	$this->getEntityManager()->flush();

	return json_encode($this->convertToArray($user));
}
```

The User Resource should implement all relevant methods including _POST_ and _DELETE_ and even consider implementing _OPTIONS_.

###Setting up the index.php file

First we will need to add the vendor autoload file in case you are using composer else this part depends on how autoloading is handled in your setup.

```php
<?php
use App\Entity\User;

require '../vendor/autoload.php';
```

Next step is to create the UserResource and the Slim instance.

```php
$userResource = new \App\Resource\UserResource();

$app = new \Slim\Slim();
```

The following section enables a rest call that returns all users or a single user depending on whether an id is defined or not.

```php
<?php
$app->get('/users(/(:id)(/))', function($id = null) use ($userResource) {
	echo $userResource->get($id);
});
```

We can also add the POST, PUT and DELETE to the users resource:

```php
<?php
$app->post('/users', function() use ($userResource) {
	echo $userResource->post();
});

$app->put('/users/:id', function($id = null) use ($userResource) {
	echo $userResource->put($id);
});

$app->delete('/users/:id', function($id = null) use ($userResource) {
	echo $userResource->delete($id);
});

$app->run();
```

This is all it actually needs to have a Users Resource up and running that handles creating, updating, deleting and retrieving a specific resource.

As a tip: one might consider adding another level of abstraction when dealing with multiple resources.
Instead of having to explicitly define routes for every possible resource an alternative would be to add a resource factory that returns the correct
Resource implementation. For example:

```php
<?php
$app->get('/:resourceType(/(:id)(/))', function($resourceType, $id = null) {
	$resource = ResourceFactory::get($resourceType);
	echo $resource->get($id);
});
```

###Adding the Doctrine Commandline Tool

Create a new file **doctrine_cli.php** :

```php
<?php

require 'vendor/autoload.php';

$path = array('Path/To/Entity');
$devMode = true;

$config = Setup::createAnnotationMetadataConfiguration($path, $devMode);

$connectionOptions = array(
	'driver'   => '',
	'host'     => '',
	'dbname'   => '',
	'user'     => '',
	'password' => '',
);

$em = \Doctrine\ORM\EntityManager::create($connectionOptions, $config);

$helpers = new Symfony\Component\Console\Helper\HelperSet(array(
	'db' => new \Doctrine\DBAL\Tools\Console\Helper\ConnectionHelper($em->getConnection()),
	'em' => new \Doctrine\ORM\Tools\Console\Helper\EntityManagerHelper($em)
));
```

For more details on the configuration options read the [documentation](http://doctrine-orm.readthedocs.org/en/latest/reference/configuration.html).

To create the schema simply type :

```javascript
$ /path/to/vendor/bin/doctrine orm:schema-tool:create
```

###Testing the API via CURL

	$ curl -i -X GET rest.localhost/users/1

	$ curl -i -X GET rest.localhost/users

	$ curl -i -X POST -d "email=foo2@bar&name=baz2" rest.localhost/users

	$ curl -i -X PUT -d "email=foo@bar&name=baz" rest.localhost/users/10

	$ curl -i -X DELETE rest.localhost/users/1

###Roundup
This entry was just a short introduction into setting up Doctrine with Slim.
If you need a lightweight approach for setting up a restful API you might consider looking into the [Slim](http://www.slimframework.com/) framework.

If you need to test your api, you might have a look at the behaviour driven development (BDD) testing framework [Behat](http://behat.org/).

Finally you should also take a look at [Slim REST API](https://github.com/jeroenweustink/slim-rest-api) on github, especially if you do not want to incorporate
Doctrine with Slim manually.