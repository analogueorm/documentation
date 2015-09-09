##Entities

An entity is an object representing an individual database record (row), identified by a primary key, and containing several attributes, which can be either a *database field*, a *Value Object* or a *Relationship*.

Analogue ships with a base Entity class you can extend and which provides a little sugar, as the magical setters and getters, hidden attributes, and handling the Array/Json conversions. 

Or you can use your own Class, as long as it implements the *Mappable* contract which analogue relies to hydrate and retrieve the entity's attribute (you can use the MappableTrait for convenience). Choice is yours.

Analogue will use by default the plural of the class name to select the proper database table (Product Entity => 'products' table), but you can customize it in the corresponding *EntityMap* class. 

```php

// Using the base Entity class.

use Analogue\ORM\Entity;

class Product extends Entity 
{
    public function __construct($name, $price)
    {
        $this->attributes['name'] = $name; 
        $this->attributes['price'] = $price;  
    }
}


// Using the Mappable trait

use Analogue\ORM\Mappable;
use Analogue\ORM\MappableTrait;

class Product implements Mappable {
    use MappableTrait;

   ...
}

```

>**Note** : The Mappable interface methods are only meant to be used by the Mapper. As a good pratice, you shouldn't use these methods in your own code.


##Entity Maps

Entity Maps are configuration classes that sit next to the Entity, typically in the same namespace where it is autodected by Analogue, but you can of course set it up manually (see [Configuring Entity Maps](https://github.com/analogueorm/analogue/wiki/Configuring-Entity-Maps) ).

They're used to define relationships, set a custom table name, and many other custom mapping behaviours.

App\Product
App\ProductMap

```php

use Analogue\ORM\EntityMap;

class ProductMap extends EntityMap {
    
    protected $table = 'my_products';

}
```


##Getting the Mapper for an Entity class

Interaction with the database are handled by a Mapper object.

To request a Mapper instance just call the mapper method on Analogue with the name of the class or an instance of the Entity. 

```php
    $productMapper = $analogue->mapper(Product::class); 
```

If you want, you can also call it statically for the same result:

```php
   $productMapper =  Analogue::mapper(Product::class); 
```


##Value Objects

Value Objects in Analogue are similar to entities, but they're not tied to a database table and cannot have relationships. They are used to abstract common data types/behaviour from the sole Entity.

By convention, Value Object's attributes are stored in the Entity's table by prefixing them with the name of the Value Object ('valueobject_attribute').

As a very simple example, we can create an Identity value object, composed of 'first_name' and 'last_name' attributes. These attributes will be stored respectively in the 'identity_first_name' & 'identity_last_name' columns on the user's table.

```php
use Analogue\ORM\ValueObject;

class Identity extends ValueObject {

    public function __construct($firstName, $lastName)
    {
        $this->first_name = $firstName;
        $this->last_name = $lastName;
    }

}
```

Then, in the same fashion as the EntityMap, we need to create a ValueMap class, to reference mapped attributes.

```php
use Analogue\ORM\ValueObject;

class IdentityMap extends ValueMap
{
    protected $attributes = ['first_name', 'last_name'];
}

```

We can now use this Identity value object inside, let's say, a Person entity : 

```php
use Analogue\ORM\Entity;

class Person extends Entity {
    
    public function __construct(Identity $identity)
    {
        $this->identity = $identity;
    }

}
```

We also need to declare the ValueObject in the entity map :

```php
use Analogue\ORM\Entity;

class PersonMap extends EntityMap {
    protected $embeddables = ['identity' => Identity::class];

}
```

