While Analogue uses lazy loading and eager loading transparently, it's important to understand the concept as using them the wrong way can have a dramatic impact on your application's performances.

#Lazy Loading

When you run a query on an entity which have relationships defined in its EntityMap, Analogue set the attributes corresponding to the relation name with a **Proxy** class. This proxy class is loaded with a prepared query statement which will be executed only if you access this property in your code.

This can lead to the infamous N+1 problem when you access a lazy loaded relation within a foreach loop : 

```php

$users = $userMapper->get();

$avatars = [];

foreach($users as $user)
{
    $avatars[] = $user->avatar->url;
}

```

If you loop through 100 users, you'll end up with 101 queries ! 

#Eager Loading

To solve this issue, you can use the with() statement in your initial query :

```php

$users = $userMapper->with('avatar')->get();

$avatars = [];

foreach($users as $user)
{
    $avatars[] = $user->avatar->url;
}

```

You can also configure your UserMap to automatically eager load some choosen relations. 

```php

class UserMap extends EntityMap {

    protected $with = ['avatar'];
    
    public function avatar(User $user)
    {
        return $this->hasOne($user, Avatar::class);
    }

}
```

Keep in mind that auto eager loading cascades themselves. If AvatarMap would have an auto eager loading on an Images relation, these one would be loaded along with all User request. So, choose them with care, as overdoing it can also lead to pretty big queries. 
