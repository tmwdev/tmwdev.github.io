## Object Pooling

A common issue in games is dealing with objects that need to be repeatedly created and destroyed (e.g. enemies, bullets, etc). Since creating and destroying objects is expensive, I'm using object pools to store deactivated instances of these objects, and activating/deactivating them as necessary.

**Pool Manager**

I wanted the pooling system to be as simple to use as possible. For this reason, I have a singleton "manager" class that keeps track of all the pools. The manager stores the pools in a Dictionary, with the object prefab acting as the key. This way, any class that needs to use a pool only needs to know the "type" of object it will be using. To further simplify (and safeguard) the pool usage, I've also set up the manager class to automatically create new pools whenever a new object is requested.

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

**Object Pool**

Continuing with the theme of keeping the pool system as straightforward as possible, I designed the object pools themselves to automatically grow as needed.

{% highlight csharp %}
/* =================================================================================================
 * Object pools are used to manage instances of objects that need to be repeatedly recycled.  We use
 * a Queue to store each instance (deactivated) of the object.  When an instance is needed, we
 * remove the first item from the queue, activate it, place it in the world, and then return the
 * instance's reference to the class that requested it.  If the queue is empty, we create a new
 * instance first.  This way, the pool will grow to whatever size is needed automatically.
 * ===============================================================================================*/

public class ObjectPool
{
	private GameObject m_poolPrefab;
	private Queue<GameObject> m_pool = new Queue<GameObject>();
	private List<ObjectPoolHelper> m_poolHelpers = new List<ObjectPoolHelper>();
	private Transform m_poolRoot;

	//..............................................................................................
	public ObjectPool (GameObject prefab, int initialSize, Transform poolRoot)
	{
		m_poolPrefab = prefab;
		m_poolRoot = poolRoot;

		for (int i = 0; i < initialSize; i++) {
			AddObjectToPool();
		}
	}

	//..............................................................................................
	private void AddObjectToPool()
	{
		// instantiate and activate the object
		GameObject newObject = MonoBehaviour.Instantiate(m_poolPrefab) as GameObject;
		newObject.SetActive(false);

		// set the object's parent to the "pool".  This isn't important for gameplay but it does
		// help keep things organized
		newObject.transform.SetParent(m_poolRoot);

		// attach a "pool helper" to the object - this helper script allows the object to return to
		// the pool and ensures no errors occur if one or both need to be destroyed
		ObjectPoolHelper helper = newObject.AddComponent<ObjectPoolHelper>();
		helper.SetPool(this);
		m_poolHelpers.Add(helper);

		// add the object to the pool
		m_pool.Enqueue(newObject);
	}

	//..............................................................................................
	public void BreakConnectionWithPool(ObjectPoolHelper helper)
	{
		m_poolHelpers.Remove(helper);
	}

	//..............................................................................................
	public GameObject Spawn(GameObject prefab, Vector3 position, Quaternion rotation)
	{
		GameObject pooledObject = null;

		// for as long as objects exist in the pool, attempt to grab the first one as long as it
		// isn't null (ie has been destroyed)
		// if the end of the pool is reached, create a new object
		while (m_pool.Count > 0 && !(pooledObject = m_pool.Dequeue())) {}
		if (pooledObject == null && m_pool.Count == 0) {
			AddObjectToPool();
			pooledObject = m_pool.Dequeue();
		}

		// activate the object and set its position and rotation
		pooledObject.SetActive(true);
		pooledObject.transform.position = position;
		pooledObject.transform.rotation = rotation;

		// return a reference to the newly spawned object
		return pooledObject;
	}

	//..............................................................................................
	public void Despawn(GameObject gameObject, ObjectPoolHelper helper)
	{
		if (m_poolHelpers.Contains(helper)) {
			gameObject.SetActive(false);
			m_pool.Enqueue(gameObject);
		} else {
			MonoBehaviour.Destroy(gameObject);
			throw new System.Exception("Attempting to add a non-pooled item to object pool.");
		}
	}

	//..............................................................................................
	public void EmptyPool()
	{
		foreach (ObjectPoolHelper helper in m_poolHelpers) {
			MonoBehaviour.Destroy(helper.gameObject);
		}
	}
}

{% endhighlight %}


