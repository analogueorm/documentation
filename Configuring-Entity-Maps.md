When you need to define relationships between Entities, along with other custom behaviors, you have to create an EntityMap class.

```php
use Analogue\ORM\EntityMap;

class ProductMap extends EntityMap {
    
}

```

If the resides in the same namespace, alongside with the entity, with the 'Map' suffix, it will be autodetected by Analogue :

Category
CategoryMap
Product
ProductMap
...

Alternatively, you can explicitely register the class using the Analogue::register() method. 

```php

// using class namespaces :
$analogue->register(Category::class, CategoryMap::class);

// or using object instances :
$analogue->register($entity, $entityMap)

```

Note : the register method has to be called when bootstrapping the application, before the first call to Analogue::mapper() on the given entity is made. 

In Laravel, place them in the boot() method of your service provider.

##Options :

###Database Table

Use the $table property to setup a custom table name :

```php
use Analogue\ORM\EntityMap;

class ProductMap extends EntityMap {
    
    protected $table='product';

}

```

###Primary Key

Use the $primaryKey property to setup a custom primary key :

```php
    
    protected $primaryKey='product_id';

```

###Entity Class

Use the $class property to indicate the class name of the Entity the Map is attached to.

```php
    
    protected $class='App\Products\Product';

```

##Timestamps and SoftDeletes

Timestamps and SoftDeletes are supported via plugins. If you use Analogue with the included service provider, these plugins are automatically loaded for you.

Otherwise, you need to register them at boot time :

```php
$analogue->registerPlugin('Analogue\ORM\Plugins\Timestamps\TimestampsPlugin');
$analogue->registerPlugin('Analogue\ORM\Plugins\SoftDeletes\SoftDeletesPlugin');
```

###Timestamps 

Timestamps are two columns, *created_at* and *updated_at*, that will automatically update upon creation or update of your entity. They are *disabled* by default. To enable timestamps support :

```php
    
    public $timestamps = true;

```

Overriding timestamps column names

```php
    
    protected $createdAtColumn = 'creation_date';

    protected $updatedAtColumn = 'update_date';

```

###Soft Deletes

When soft deleting an entity, it is not actually removed from your database. Instead, a deleted_at timestamp is set on the record.

To enable soft delete support :

```php
    
    public $softDeletes = true;

```

Overriding Soft Delete column name

```php
    
    protected $deletedAtColumn = 'delete_date';

```

####Querying deleted entities 

Softdeletes uses Scopes on the query object meaning that every call to *$mapper->query()* will only apply on the non-deleted entities.

To retrieve deleted entities :

```php

$trashed = $mapper->globalQuery()->whereNotNull('deleted_at')->get();

```

####Restoring entities

The Soft Deletes plugin adds a *restore()* method to the mapper, simply pass the Entity or a Collection of entities to restore them :

```php

$mapper->restore($entity);

```

#### Emptying Trash

To definitely erase a soft deleted Entity from the database, simply pass it to the delete method :

```php

$trashed = $mapper->globalQuery()->whereNotNull('deleted_at')->get();
$mapper->delete($trashed);

```

##Relationships 

Relationships are defined as methods on the EntityMap class. These methods are used internaly by the mapper to assign related entities & collection to your entity object. To learn how to work with related enties, check the [tutorial](#). 

Here are the available relationships types : 

- [One To One](#one-to-one)
- [One To Many](#one-to-many)
- [Many To Many](#many-to-many)
- [Has Many Through](#has-many-through)
- [Polymorphic One To Many](#polymorphic-one-to-many)
- [Polymorphic Many To Many](#polymorphic-many-to-many)


The relationship methods only take one argument, which is the entity Object initiating the relation. This argument has to be *type hinted to the correct entity class*, as analogue uses Reflection to autodetect EntityMap's relationships.

```php
use App\User;
use App\Role;

class UserMap {
    
    public function role(User $user)
    {
        return $this->hasOne($user, Role::class);
    }
}

```

###One to One

- (optional) **foreign_key**: Name of the key column on the related entity.
- (optional) **local_key**  : Name of the key on the initiating entity.

```php

    public function role(User $user)
    {
        return $this->hasOne($user, Role::class, 'foreign_key', 'local_key');
    }

```

The inverse, of this relation, on the RoleMap class :

```php
    
    public function user(Role $role)
    {
        return $this->belongsTo($role, User::class, 'local_key', 'parent_key');
    }

```

###One to Many

The One to Many relation method works the same as the one to one, but results in a Collection instead of a single Entity.


```php
    
    public function permissions(Role $role)
    {
        return $this->hasMany($role, Permission::class, 'foreign_key', 'local_key');
    }

```

The inverse, of this relation, on the PermissionMap class :

```php
    
    public function role(Permission $permission)
    {
        return $this->belongsTo($permission, Role::class, 'local_key', 'parent_key');
    }
```

###Many to Many

Many to many relationships link two Entity class through a pivot table. Example for users that can have multiple resources they share : 

```php
public function resources(User $user)
{
    return $this->belongsToMany($user, 'App\Resource', 'user_resource', 'user_id', 'resource_id');
}
```

- (optional) *user_resource : name of the pivot table
- (optional) *user_id       : key for User on the pivot table
- (optional) *resource_id   : key for Resource on the pivot table

###Has Many Through

The "has many through" relation provides a convenient short-cut for accessing distant relations via an intermediate relation. For example, a Country might have many Post through a User entity. 

```php
public function posts(Country $country)
{
    return $this->hasManyThrough($country, 'App\Post', 'App\User', 'country_id', 'user_id');
}
```
- (optional) *country_id: key on the User table
- (optional) *user_id   : key on the Post table

###Polymorphic One To Many

Polymorphic relations allow a entities to belong to more than one other Entity class, on a single association. For example an Article or a blog Post may have a featured Image.

```php

class ImageMap extends EntityMap {

    public function imageable(Image $image)
    {
        return $this->morphTo($image);
    }

}

class ArticleMap extends Model {

    public function images(Article $article)
    {
        return $this->morphMany($article, 'App\Image', 'imageable');
    }

}

class PostMap extends Model {

    public function images(Post $post)
    {
        return $this->morphMany($post, 'App\Image', 'imageable');
    }

}
```

###Polymorphic Many To Many

In addition to traditional polymorphic relations, you may also specify many-to-many polymorphic relations. If we take our previous examples, both Article and Post could share the same Image entities.

What we need in that case is to create a 'imageables' pivot table containing :

image_id (integer)
imageable_type (string)
imageable_id (integer)

```
class ArticleMap extends Model {

    public function images(Article $article)
    {
        return $this->morphToMany($article, 'App\Image', 'imageable');
    }

}

class PostMap extends Model {

    public function images(Post $post)
    {
        return $this->morphToMany($post, 'App\Image', 'imageable');
    }

}
```

The ImageMap needs to define a methods for both relationships :

```php

class ImageMap extends EntityMap {

    public function articles(Image $image)
    {
        return $this->morphedByMany($image, 'App\Article', 'taggable');
    }

    public function posts(Image $image)
    {
        return $this->morphedByMany($image, 'App\Post', 'taggable');
    }

}

```

###Dynamic Relationships

Dynamic relationships are callbacks methods that can be added at runtime on a EntityMap class. They can be really useful when you're building a plugin-driven architecture, to allow plugin creators to link additionnal data to your core domain entities, without the need for a complex single table inheritance and/or extension mechanism.

For example, we can add an avatar relationship to a core User class :

```php

$userMapper->getEntityMap()->addRelationshipMethod('avatar', function($user, $map) {
    return $map->hasMany($user, Avatar::class);
});

```

##Injecting services into Entities

If you need to inject some service into an entity, you can provide an activator() method to Analogue, which it will uses when running Query to instantiate the Entity class before hydrating.

It's not generally considered good pratice, but it can prove useful in some situations where your entity logic is tightly bound to a service. Again, for maintenability, always favor tying your Entity to an Interface over a concrete implementation.

```php

class User extends Entity {
    
    protected $hasher;

    public function __construct(HashingContract $hasher)
    {
        $this->hasher = $hasher;
    }

}

class UserMap {
    
    public function activator()
    {
        return new User(new HashingService);
    }

}

```

**TIP** : You can get a new instance of your entities by invoking the **newInstance()** method on the mapper object.