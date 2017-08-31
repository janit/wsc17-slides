# Doctrine ORM with eZ Platform REST framework<br />(and GraphQL)

<center><small>Jani Tarvainen, September 1st 2017</small></center>

---

## http://janit.iki.fi/ez-rest-graphql

---


## Agenda

 - Doctrine ORM as a backend for eZ Platform REST
 - Using GraphQL with eZ Platform
 - Extending GraphQL with a Doctrine endpoint

---

## Doctrine ORM as a backend<br/>for eZ Platform REST

--

### Use an appropriate storage backend

 - The eZ REST can be used for alternative backends
 - Not everything should be stored in the content repository, you can use other backends for parts of your data
 - You can use shared REST infrastructure from eZ Platform
 - Makes it familiar to work with and to consume

--

### Storing WSC attendees in Doctrine ORM

 - Using Doctrine ORM entities to store attendee information
 - Shared Database connection, separate database tables
 - eZ Platform is unaware of Doctrine and the entities
 - Exposing them are identical to eZ Content Objects

--

### Prebuilt and configured Entities

- Enable dev in /etc/apache2/sites-available/ezrestapi.conf
- Entities and fixtures are in place

```
git pull
git checkout janit-1
composer update
php app/console doctrine:schema:update --force
php app/console restapi:generate-attendees
```

- Includes some configuration and code:
  - app/config/config.yml
  - src/AppBundle/Entity/Attendee.php
  - src/AppBundle/Repository/AttendeeRepository.php
- Attendees dump in: http://ezrestapi.websc/attendees

--

### Install Postman

 - Install Postman
 - https://chrome.google.com/webstore/detail/postman/fhbjgbiflinjbdggehcddcbncdddomop
 - http://ezrestapi.websc/api/ezp/v2/attendees
   - accept: application/vnd.ez.api.Attendee+json

--

### Extend controller with new route

- src/AppBundle/Controller/AttendeesController.php

```
public function listAllAction()
{

    $em = $this->container->get('doctrine.orm.default_entity_manager');

    $attendees = $em->getRepository('AppBundle:Attendee')->findAll();

    return new AttendeeList($attendees);
}
```

--

### Add route

- src/AppBundle/Resources/config/routing_rest.yml

```
list_attendees:
    path:     /attendees
    defaults: { _controller: AppBundle:Attendees:listAll }
    methods: [GET]
```

--

### Add value

- src/AppBundle/Rest/values/AttendeeList.php

```
<?php

namespace AppBundle\Rest\values;

class AttendeeList
{
    public $attendees;

    public function __construct( $attendees )
    {
        $this->attendees = $attendees;
    }
}
```

 - Also add namespace to Controller

```
use AppBundle\Rest\values\AttendeeList;
```

--

### Configure Value Object Visitor

- app/config/services.yml

```
app.rest.value_object_visitor.attendee_list:
  parent: ezpublish_rest.output.value_object_visitor.base
  class: AppBundle\Rest\valueObjectVisitor\AttendeeList
  tags:
      - { name: ezpublish_rest.output.value_object_visitor, type: AppBundle\Rest\values\AttendeeList }
```

--

### Add Value Object Visitor

- src/AppBundle/Rest/valueObjectVisitor/AttendeeList.php

```
<?php

namespace AppBundle\Rest\valueObjectVisitor;

use eZ\Publish\Core\REST\Common\Output\ValueObjectVisitor;
use eZ\Publish\Core\REST\Common\Output\Generator;
use eZ\Publish\Core\REST\Common\Output\Visitor;

class AttendeeList extends ValueObjectVisitor
{
    public function visit( Visitor $visitor, Generator $generator, $data )
    {

        $generator->startObjectElement('AttendeeList');
        $generator->startList('Attendees');

        foreach($data->attendees as $attendee){
            $generator->startObjectElement('Attendee');
            $generator->startValueElement('id', $attendee->getId());
            $generator->endValueElement('id');
            $generator->startValueElement('name', $attendee->getName());
            $generator->endValueElement('name');
            $generator->endObjectElement('Attendee');
        }

        $generator->endList('Attendees');
        $generator->endObjectElement('AttendeeList');
    }
}
```

--

### CURL to get attendees as JSON

 - Voilà! Our entities via eZ REST framework:
```
curl -X GET \
  http://ezrestapi.websc/api/ezp/v2/attendees \
  -H 'accept: application/vnd.ez.api.Attendee+json'
```
 
 - Complete code is in branch: "janit-2"

---

## Using GraphQL<br />with eZ Platform

--

### Introduction to GraphQL

 - An alternative approach to REST
   - A complete specification, http://graphql.org
 - GraphQL is a query language (+ client/server impl)
   - The syntax looks like JSON without data, just the keys
 - Often demoed with a <a href="https://github.com/graphql/graphiql">GraphiQL</a> client app
 - Strongly typed - provides some novel features
 - All requests sent as HTTP POSTs
   - Reads are "Queries"
   - Writes are "Mutations"
   - Caching tricky, requires new techniques

--

### Quick GraphQL hands on

 - http://www.graphqlhub.com/playground

```
{
  giphy {
    search(query: "cat") {
      id
      url
    }
  }
}
```

--

### eZ Platform and GraphQL

 - There are PHP libraries and Symfony integrations
 - There is an experimental GraphQL Bundle for eZ Platform
   - Builds on existing code (Onyx library & Overblog bundle)
   - Written by Bertrand Dunogier from eZ Systems
 - Bundle exposes the Public PHP API via GraphQL
    - It's "just" a mapping, so it's already very stable
 - https://github.com/bdunogier/ezplatform-graphql-bundle

--

### Installing GraphQL Bundle

 - https://github.com/bdunogier/ezplatform-graphql-bundle
   1. Require with Composer
   2. Enable Bundle in AppKernel
   3. Add configuration*
   4. Add routing

<p>*Old GraphiQL version, remember to specify</p>

```
overblog_graphql:
  versions:
    graphiql: 0.11
```

--

### Hands on with eZ Platform GraphQL

 - Checkout step 3 and run composer update

```
git checkout janit-3
composer update
```
 
 - After this you can hit GraphiQL client at:<br >http://ezrestapi.websc/graphiql

--

### A quick look at GraphiQL & Docs

 - Before writing anything a query look at Docs button
 - Select "Query" and you'll see all available queries
   - Terms here are eZ Platform specific
   - This is the API documentation
 - Select locationChildren
   - It takes "id" as an argument
   - Output is a <b>Location</b> object
 - Click location
    - All attributes available
    - Includes embedded <b>Content</b> object

--

### Write an eZ Platform query in GraphQL

 - "We want the days, and the sessions' title & description"
 - Start with a simple query:

```
{locationChildren}
```
 
 - Execute. Response is null, because we don't have an id.
 - To add parent id, add it:

```
{locationChildren(id:2) {
  id
}}
```

 - Now response is the IDs of child locations

--

### Write an eZ Platform query in GraphQL

 - We've now got the IDs, but we want the name too
 - To get the name we'll want to get the <b>Content</b> object
 - From the object we'll want to select <b>Name</b>, but not the ID
 
```
{locationChildren(id:2) {
  id,
  content {
    name
  }
}}
```

 - The response now contains the names, nested

--

### Write an eZ Platform query in GraphQL

 - Finally we'll want content for the child locations
 - Queries can be nested. Let's get the children of the days:
 
```
{locationChildren(id:2) {
  id,
  content {
    name
  },
  children {
    id,
    content {
      name
    }
  } 
}}
```

 - The response now contains the session names

--

### Write an eZ Platform query in GraphQL

 - Finally pull the description field from the child objects:
 
```
{locationChildren(id:2) {
  id,
  content {
    name
  },
  children {
    id,
    content {
      name,
      fields(identifier:["description"]) {
        value {
          text
        }
      }
    }
  } 
}}
```

 - There's more features, but you can see the APIdoc

---

## Extend the GraphQL backend<br />with a custom endpoint

--

### Expose Doctrine Entities via GraphQL

 - We can extend the GraphQL backend with new queries
 - You can add a separate endpoint or extend the eZ one
   - For the sake of argument, we'll add a new one
   - http://example.com/graphiql/attendee
 - Extend the Overblog bundle with configuration+code
 - Overblog GraphQLBundle: https://github.com/overblog/GraphQLBundle

--

### Update the GraphQL bundle

 - I want new features from the latest GraphQL Bundle
 - Update the bundle to use my fork via Composer.json:

```
    "repositories": [
        {
            "type": "vcs",
            "url": "https://github.com/janit/ezplatform-graphql-bundle"
        }
```

 - After that, update packages:

```
composer update
```

--

### Define a type for attendee

 - src/AppBundle/Resources/config/graphql/Attendee.types.yml:

```
Attendee:
    type: object
    config:
        description: "A query to return Attendee"
        fields:
            id:
                type: "Int!"
                description: "The unique ID of the attendee."
            name:
                type: "String"
                description: "Name of the attendee"
            country:
                type: "String"
                description: "Country of the Attendee"
            bio:
                type: "String"
                description: "Bio of the Attendee"
```

 - This defines our structure and doubles as documentation

--

### Define a schema for our query

 - src/AppBundle/Resources/config/graphql/Query.types.yml:

```
AttendeesQuery:
    type: object
    config:
        description: "Attendee ORM repository"
        fields:
            attendee:
                type: "Attendee"
                args:
                    id:
                        description: "Resolves using the attendee id."
                        type: "Int"
                resolve: "@=resolver('Attendee', [args])"
```

 - This is picked up by convention, and registers <b>AttendeesQuery</b>. It also configures a <b>resolver</b> for our custom Attendee type. We will write the resolver in PHP.

--

### Configure an endpoint for attendees

 - The bundle allows <a href="https://github.com/overblog/GraphQLBundle/blob/master/Resources/doc/definitions/schema.md#multiple-schema-endpoint">configuring multiple endpoints</a>
 - app/config/config.yml:

```
overblog_graphql:
    definitions:
        internal_error_message: "An error occurred, please retry later or contact us!"
        config_validation: %kernel.debug%
        schema:
            ez:
                query: Query
                mutation: PlatformMutation
            attendees:
                query: AttendeesQuery
    versions:
        graphiql: 0.11
```

 - GraphiQL: http://ezrestapi.websc/graphiql/attendees

--

### Add a resover for Attendee

- src/AppBundle/GraphQL/Resolver/AttendeeResolver.php

```
<?php

namespace AppBundle\GraphQL\Resolver;

use Doctrine\ORM\EntityManager;
use Overblog\GraphQLBundle\Resolver\ResolverInterface;

class AttendeeResolver implements ResolverInterface {

    private $em;

    public function __construct(EntityManager $em)
    {
        $this->em = $em;
    }

    public function resolve($input)
    {

        $args = $input->getRawArguments();

        $attendee = $this->em->getRepository('AppBundle:Attendee')->find($args['id']);

        return [
            'id' => $attendee->getId(),
            'name' => $attendee->getName(),
            'country' => $attendee->getCountry(),
            'bio' => $attendee->getBio()
        ];

    }
}
```

- Like a ValueObjectVisitor in eZ REST, you can use backend. A prepared eZ Query is fine, but we'll use Doctrine ORM.

--

### Fetching Individuals

 - Fetching individual attendees

```
{attendee(id:1) {
  id,
  name,
  country,
  bio
}}
```

 - You can check out the complete code:

```
git checkout janit-4
```

--

### Defining AttendeeList type

 - To get a list of attendees, we will add a query that will embed <b>Attendee</b> objects. Let's start with the type definition.
 - src/AppBundle/Resources/config/graphql/AttendeeList.types.yml:

```
AttendeeList:
    type: object
    config:
        description: "A query to return list of Attendees"
        fields:
            id:
                type: "String!"
                description: "The unique hash of the attendee query params."
            attendees:
                type: "[Attendee]"
                description: "Attendee info"
```

 - The <b>attendees field</b> is an array of <b>Attendee objects</b>

--

### Defining attendee query

 - We need to expose a new query, so it is available.
 - src/AppBundle/Resources/config/graphql/Query.types.yml:

```
attendee_list:
    type: "AttendeeList"
    args:
        limit:
            description: "limit"
            type: "Int"
    resolve: "@=resolver('AttendeeList', [args])"
```

 - The resolver is configured and needs a resolver in PHP

--

### Add AttendeeList Resolver

 - The resolver lists all attendees and returns a nested array
 - src/AppBundle/GraphQL/Resolver/AttendeeListResolver.php:

```
<?php

namespace AppBundle\GraphQL\Resolver;

use Doctrine\ORM\EntityManager;
use Overblog\GraphQLBundle\Resolver\ResolverInterface;

class AttendeeListResolver implements ResolverInterface {

    private $em;

    public function __construct(EntityManager $em)
    {
        $this->em = $em;
    }

    public function resolve($input)
    {

        $args = $input->getRawArguments();

        $attendees = $this->em->getRepository('AppBundle:Attendee')->findAll();

        $attendeeList = [];

        foreach($attendees as $attendee){

            $attendeeList[] = [
                'id' => $attendee->getId(),
                'name' => $attendee->getName(),
                'country' => $attendee->getCountry(),
                'bio' => $attendee->getBio()
            ];

        }

        return [
            'id' => md5(serialize($args)),
            'attendees' => $attendeeList
        ];

    }

}
```

 - The limit argument is not used, but it is available in <b>$args</b>

--

### Query AttendeeList of Attendee objs

 - In GraphiQL run a query to get all the names of attendees:

```
{attendee_list {
  attendees{
    name
  }
}}
```

 - To get the full working code, check out branch janit-5:

```
git checkout janit-5
```

---

## The end

- Thank you