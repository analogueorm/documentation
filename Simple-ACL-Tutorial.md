In Analogue, the content of your database is represented in your application as **Entity** & **Collection** objects. 

In this tutorial, we'll build simple ACL domain, where we have users, roles & permissions. A user can have only one role which grants him a set of permissions attached to that role. For more flexibility, a user can be granted additionnal permissions that are added to the one from his role.

Our database contains 5 tables :

- users (id, email, role_id)
- roles (id, name)
- permissions (id, label)
- role_permission (id, role_id, permission_id)
- user_permission (id, user_id, permission_id)

##Roles

First, let's create a Role class for our roles table.

```php

use Analogue\ORM\Entity;

class Role extends Entity
{
    public function __construct($name)
    {
        $this->name = $name;
    }
}
```

Having a __construct method on the Entity is not mandatory, we could have created a bare entity and just set the attributes on it, but it's a *good pratice* to enforce some attributes on our Entity, to ensure it meets our minimum requirements for saving.

Now let's create two roles, **admin** and **user**, and store them in the database.

```php

$roleMapper = $analogue->mapper(Role::class);

$adminRole = new Role('admin');
$userRole = new Role('user');

$roleMapper->store( [$adminRole, $userRole] );

```

The mapper's store() method accepts either **object**, **array** or **collection**. When passing it multiple records at a time, it will automatically wrap the several INSERT statements within a database transaction for better performance.

Next, let's create a User Entity.

```php

use Analogue\ORM\Entity;

class User extends Entity
{
    public function __construct($email, Role $role)
    {
        $this->email = $email;
        $this->role = $role;
    }
}
```

The constructor type hinting comes handy to ensure we create a user object with a valid Role object.

The user has no password field. We'll assume for this tutorial that the world is a beautiful place where everyone can be trusted (don't do it in production).

Ok. We created our User entity but before creating users, we need to inform Analogue of the relationship between the User and the Role entity. We need to create a map for it.

```php

use Analogue\ORM\EntityMap;

class UserMap extends EntityMap {
    
    public function role(User $user)
    {
        return $this->belongsTo($user, Role::class);
    }

}

```

A relationship method has always one argument, which is the entity originating the relationship. It's also have be type hinted to the correct class name, as Analogue uses Reflection to detect the relationships on your map. 

You'll almost never use an entityMap class directly in our code, only in some special cases, its main purpose is to tell Analogue how to map objects to the database.

But now that we have mapped the User entity, we can create our first user.

```php

$userMapper = $analogue->mapper(User::class);

$alice = new User('alice@example.com', $adminRole);

$userMapper->store($alice);

```

Behind the scenes, the mapper recognizes the **role** attributes as corresponding to the **role()** relationship method, and has set the foreign key (role_id) for us.

But what if the admin role did not exists ? 

```php

$bob = new User('bob@example.com', new Role('guest'));

$userMapper->store($bob);
```

When storing Entities, Analogue parse the relationships and will create the non-existing records when needed.


##Permissions

As we said earlier, we want to define permissions both for the roles and gives ourself the ability to add permissions to a user on a per case basis.

The permission itself will consist in a single string :

```php

use Analogue\ORM\Entity;

class Permission extends Entity
{
    public function __construct($label)
    {
        $this->label = $label;
    }

}
```

Then we need to define an EntityMap for the Role Entity. 

```php

use Analogue\ORM\EntityMap;

class RoleMap extends EntityMap {
    
    public function permissions(Role $role)
    {
        return $this->belongsToMany($role, Permission::class);
    }

}

```

And also add a method on the UserMap to link to user specific permissions :

```php

use Analogue\ORM\EntityMap;

class UserMap extends EntityMap {
    
    public function permissions(User $user)
    {
        return $this->belongsToMany($user, Permission::class);
    }

}

```

Now, let's go back on our User entity and create a new method to add these permissions.

```php
use Analogue\ORM\Entity;
use Analogue\ORM\EntityCollection;

class User extends Entity
{
    public function __construct($email, Role $role)
    {
        $this->email = $email;
        $this->role = $role;
        $this->permissions = new EntityCollection;
    }

    public function addPermission(Permission $permission)
    {
        $this->permissions->add($permission);
    }
}

```

Notice that we introduced added new EntityCollection class in our class constructor. This class extends Illuminate\Support\Collection with some methods to deal with entities, as the add() one we use there.

We can safely copy and paste this code into our Role entity :

```php

use Analogue\ORM\Entity;
use Analogue\ORM\EntityCollection;

class Role extends Entity
{
    public function __construct($name)
    {
        $this->name = $name;
        $this->permissions = new EntityCollection;
    }

    public function addPermission(Permission $permission)
    {
        $this->permissions->push($permission);
    }
}

```

With everything in place, we're go wild and create permissions for our role & users.

```php

$adminRole->addPermission(new Permission('access_admin'));
$adminRole->addPermission(new Permission('create_users'));

$roleMapper->store($adminRole);

$alice->addPermission(new Permission('reset_server'));

$userMapper->store($alice);

```

Finally, it would be awesome to be able to retrieve all the permissions for a user, regardless he got it from his role or from a custom grant. Let's implement it.

```php
use Analogue\ORM\Entity;
use Analogue\ORM\EntityCollection;

class User extends Entity
{
    public function __construct($email, Role $role)
    {
        $this->email = $email;
        $this->role = $role;
        $this->permissions = new EntityCollection;
    }

    public function addPermission(Permission $permission)
    {
        $this->permissions->push($permission);
    }

    /**
     * Return all User's permissions
     *
     * @return Collection 
     */
    public function getPermissions()
    {
        $rolePermissions = $this->role->permissions;

        return $this->permissions->merge($rolePermissions);
    }
}

```

As a result, we have a single method on our User class which our *PermissionService* can use regardless how we do organize ourselves internally. We *encapsulated our Logic inside our domain entities*. As the application's features evolve over time, for example if we want to implement a system where groups of users can share access to resources, we can do it without breaking client's code checking for permissions.

====

A few months have passed, and during that time Alice spent too much time reseting the server, causing a sensible business loss (that was at a time [Envoyer](https://envoyer.io/) didn't yet exist), so we've been asked to revoke her permission to do it.

How can we do that ?

The EntityCollection class has a convenient remove() methods that do it for us. Let's wrap this into a method on the User class.

```php

    public function removePermission(Permission $permission)
    {
        $this->permissions->remove($permission);
    }

```

Then :

```php

$permissionMapper = $analogue->mapper(Permission::class);

$resetServer = $permissionMapper->query()->whereLabel('reset_server')->first();

$alice->removePermission($resetServer);

// We still need to store() the entity for the changes to persist in the DB
$userMapper->store($alice);

```

By this action we deleted the record from the user_permission pivot table, that linked Alice to the reset_server permission. The actual record still exists in the 'permissions' table. If we want to get rid of it, we have to explicitely delete it.

```php

$permissionMapper->delete($resetServer);

```

Poor Alice, she's got fired cause the losses were too big... 

We have to remove her admin role. 

Well, nothing simpler.

```php

$alice->role = null; 

$userMapper->store($alice);

```

Setting any Relation attribute to null will have for effect to remove all the corresponding relationships, weither it's a one or many relationship. (again not deleting the related records).

===

So, we had a taste on how Analogue gives us the freedom to build our domain classes. Still, we only scratched the surface. For more examples, follow Alice in the rabbit hole, check the [Advanced recipes](https://github.com/analogueorm/analogue/wiki/Advanced-Recipes).  