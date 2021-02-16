## Object Pooling

A common issue in games is dealing with objects that need to be repeatedly created and destroyed (e.g. enemies, bullets, etc). Since creating and destroying objects is expensive, I'm using object pools to store deactivated instances of these objects, and activating/deactivating them as necessary.

I wanted the pooling system to be as simple to use as possible. For this reason, I have a singleton "manager" class that keeps track of all the pools. The manager stores the pools in a Dictionary, with the object prefab acting as the key. This way, any class that needs to use a pool only needs to know the "type" of object it will be using. To further simplify (and safeguard) the pool usage, I've also set up the manager class to automatically create new pools whenever a new object is requested.

**Pool Manager**
{% highlight csharp %}

/* =================================================================================================
 * Class used to manage all the object pools within a scene.
 * 
 * We store all the pools in a Dictionary, using the prefab object created by the pool as a key.
 * This way, anything that needs to use that particular prefab only needs to pass a reference to it
 * to the singleton instance of this manager class, and it will be directed to the appropriate pool.
 * ===============================================================================================*/

public class PoolManager : TWMonoBehaviour
{
	public static PoolManager m_instance;
	private Dictionary<int, ObjectPool> m_poolList = new Dictionary<int, ObjectPool>();

	// =============================================================================================
	// SINGLETON
	// =============================================================================================

	//..............................................................................................
	private void Awake()
	{
		if (m_instance == null) {
			m_instance = this;
		} else if (m_instance != this) {
			Destroy(this);
		}
	}

	//..............................................................................................
	public static PoolManager Get()
	{
		return m_instance;
	}

	// =============================================================================================
	// POOL MANAGEMENT
	// =============================================================================================

	//..............................................................................................
	// creates a pool for the given prefab and prepopulates it with items
	public void CreatePool(GameObject prefab, int startingSize)
	{
		int key = prefab.GetInstanceID();

		// if the pool already exists, we don't need to do anything
		if (m_poolList.ContainsKey(key)) {
			return;
		}

		// create an empty object that will hold everything in the pool
		GameObject poolroot = new GameObject();
		poolroot.name = prefab.name + " Pool";
		poolroot.transform.SetParent(this.transform);

		// pool doesn't exist yet, so let's create it
		m_poolList.Add(key, new ObjectPool(prefab, startingSize, poolroot.transform));
	}

	//..............................................................................................
	// removes a pool from the list of pools and destroys all objects it contains
	public void DestroyPool(GameObject prefab)
	{
		int key = prefab.GetInstanceID();

		// first we want to make sure the pool actually exists
		if (!m_poolList.ContainsKey(key)) {
			return;
		}

		// delete all objects from the pool, and then destroy it
		m_poolList[key].EmptyPool();
		m_poolList.Remove(key);
	}

	// =============================================================================================
	// OBJECT MANAGEMENT
	// =============================================================================================

	//..............................................................................................
	// finds the appropriate pool and spawns an object from it
	public GameObject Spawn(GameObject prefab, Vector3 position, Quaternion rotation)
	{
		int key = prefab.GetInstanceID();

		// if we don't have a pool for the given object, go ahead and create one
		if (!m_poolList.ContainsKey(key)) {
			CreatePool(prefab, 1);
		}

		return m_poolList[key].Spawn(prefab, position, rotation);
	}
}
{% endhighlight %}


### 1. Suggest hypotheses about the causes of observed phenomena

Sed ut perspiciatis unde omnis iste natus error sit voluptatem accusantium doloremque laudantium, totam rem aperiam, eaque ipsa quae ab illo inventore veritatis et quasi architecto beatae vitae dicta sunt explicabo. 

```javascript
if (isAwesome){
  return true
}
```

### 2. Assess assumptions on which statistical inference will be based

```javascript
if (isAwesome){
  return true
}
```

### 3. Support the selection of appropriate statistical tools and techniques

<img src="images/dummy_thumbnail.jpg?raw=true"/>

### 4. Provide a basis for further data collection through surveys or experiments

Sed ut perspiciatis unde omnis iste natus error sit voluptatem accusantium doloremque laudantium, totam rem aperiam, eaque ipsa quae ab illo inventore veritatis et quasi architecto beatae vitae dicta sunt explicabo. 

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).
