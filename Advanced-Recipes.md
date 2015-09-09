Here are some examples on how to use Analogue's flexible architecture to solve some common data-related tasks. 

- [Creating a Metadata Value Object](#Creating-a-Metadata-Value-Object)
- [Nested Entities](#Nested-Entities)
- [Single Table inheritance](#Single-Table-Inheritance)

#Creating a Metadata Value Object

We're building a media library application, which contains Video, Song and Photo entities. We want to be able to store media metadata in each record. The problem is the number and key of metadatas will vary between each type of medias, and even between each files as different encoding format will have its own structure. 

In a traditionnal DB schema, we would have tackle this by using a separate table to store the metas using many-to-many relationships to link media to its corresponding meta fields :

- videos
- video_metadata
- photos
- photo_metadata
- songs
- song_metadata
- metadatas

With Analogue, we can build a Value Object which will hold it's own length-variable content, storing it directly in a 'metadatas' column into the photos, videos, songs tables, keeping our DB structure simpler :

- videos
- photos
- songs

To do so, we will create a custom class that implements the **Mappable** interface. Mappable interface is a set of methods used internally by analogue to map any object to a database table.

Let's see what's the content of it :

```php
interface Mappable {

    /**
     * Set the object attribute raw values (hydration)
     * 
     * @param array $attributes 
     */
    public function setEntityAttributes(array $attributes);

    /**
     * Get the raw object's values.
     * 
     * @return array
     */
    public function getEntityAttributes();

    /**
     * Set the raw entity attributes
     * @param string $key  
     * @param string $value
     */
    public function setEntityAttribute($key, $value);

    /**
     * Return the entity's attribute 
     * @param  string $key 
     * @return mixed
     */
    public function getEntityAttribute($key);

}
```

Let's implement these methods to serialize, unserialize our fields.

```php

use Analogue\ORM\Mappable;

class Meta implements Mappable {
    
    protected $content = [];

    public function setEntityAttributes(array $attributes)
    {
        $this->setEntityAttribute('metadatas',$attributes['metadatas']);
    }
    
    public function getEntityAttributes()
    {
        return array($this->getEntityAttribute('metadatas'));
    }
  
    public function setEntityAttribute($key, $value)
    {
        if($key == 'metadatas')
        {
            $this->content = unserialize($value);
        }
    }
 
    public function getEntityAttribute($key)
    {
        if($key == 'metadatas')
        {
            return $this->content = serialize($value);
        }
    }

}

```

Along this class, for Analogue to know which attribute to map in and out the ValueObject, we need to create a ValueMap class :

```php
use Analogue\ORM\ValueMap;

class MetaMap extends ValueMap {
    protected $attributes = ['metadatas'];
}

```

Okay, we told Analogue how to map the content of our value object in and from the database. 

Now we need some function to access the $content array from our domain objects :

```php

class Meta implements Mappable {
    
    protected $content;

    public function all()
    {
        return $this->content;
    }

    public function set($key, $value)
    {
        $this->content[$key] = $value;
    }

    public function get($key)
    {
        return $this->content[$key];
    }

    ...
}
```

This is very basic, in a more advanced version, we would likely implements ArrayAccess or ArrayIterator to make the class more 'user-friendly'.

Now that our Meta class is done, we can embed it into our domain objects. To do this, we need to link the 'metadatas' field to the Meta value object. Example in the VideoMap :

```php

class VideoMap {
    
    protected $embeddables = ['metas' => Meta::class];

}

```

Each time that Analogue will hydrate a Video entity, it will hydrate the 'metas' attribute with an instance of the Meta class.

But wait a minute, what happens when we create a Video object ? The metas attribute won't be set.. 

Right, we need to create an instance of it from the constructor method :

```php
use Analogue\ORM\Entity;

class Video extends Entity {
    
    public function __construct()
    {
        $this->metas = new Meta;
    }

}

```

Even better, we can create a Media abstract class and extends from it, so we don't have to repeat it in Photo and Song :

```php
use Analogue\ORM\Entity;

abstract class Media extends Entity {
    
    public function __construct()
    {
        $this->metas = new Meta;
    }

}

class Video extends Media {
    
}

class Photo extends Media {
    
}

class Song extends Media {
    
}

```

###Nested Entities
###Single Table Inheritance
