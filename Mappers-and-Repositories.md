Analogue provides two way of interacting between your objects and the database : **mappers** and **repositories**. They share some common behaviour, as the store() and delete() methods, but they also differ in many points.

## Mapper 

Mappers are accessed by invoking :

```php

// $entity can be either an instance or the name of the class.
$analogue->mapper($entity); 

```

One Mapper object is created by Entity class during a request lifecycle, subsequently calling the mapper() method will return the already created instance.

You can run complex queries directly against the mapper object :

```
$mapper->where('age', '>', '18')->orWhere('tutor_approvation', '=', true)->get();
```

Which the mapper will turn into an Entity object according to the configuration provided in the EntityMap.

As Analogue queries are provided by the Illuminate Query Builder, you can consult the [Laravel Documentation](http://laravel.com/docs/5.0/queries) for more information on how to build advanced queries.

The Mapper class cannot be extended, although you can add custom commands to it (that's how the SoftDeletingPlugin add the restore() command).

The mapper has a few important methods :

### store(*mixed* $entity)

Persist an Entity, a Collection, or an array of entity into the database.

```php
$mapper->store($user);

$mapper->store([$userA, $userB]);
```

### delete(*mixed* $entity)

Remove an Entity, a Collection, or an array of entity from the database.

```php
$mapper->delete($user);
```

### query()

Return a new query builder instance. 

### globalQuery()

Return a query builder without query scopes.



## Repository

On the opposite, Repository class can (and are meant) to be extended.

Creating your custom repository is very easy :

```php
use Analogue\ORM\Repository;

class UserRepository extends Repository {
    
    public function __construct()
    {
        parent::__construct(User::class);
    }

}

```

As the __construct() method takes no argument, the repositories are easily injected into services using dependency injection.

The Repository will embed the $mapper property corresponding to your entity.

For example, we can factorize the previous example in our custom repo :

```php

    public function getAllowedUsers()
    {
        return $this->mapper->where('age', '>', '18')
            ->orWhere('tutor_approvation', '=', true)->get();
    }

```

The base repository has a few handful methods :

### all()

Return all the Entities from the table.

### find($id)

Return a specific Entity by its primary key.

### firstMatching(array $attributes)

Return the first entity matching the given attributes.

```php

$repository->firstMatching(['name' => 'john']);

```
### allMatching(array $attributes)

Return all the entities matching the given attributes.

```php

$repository->allMatching(['age' => 30]);

```

### paginate($perPage = null)

Return a paginator instance. If $perPage is omitted, the default configured in the
EntityMap will be used.

### store(*mixed* $entity)

Persist an Entity, a Collection, or an array of entity into the database.

```php
$repository->store($user);

```

### delete(*mixed* $entity)

Remove an Entity, a Collection, or an array of entity from the database.

```php
$repository->delete($user);

```