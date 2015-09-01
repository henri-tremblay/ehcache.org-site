---
---
# Code Samples


As this example shows, running Ehcache with Terracotta clustering is no different from normal programmatic use.


    import net.sf.ehcache.Cache;
    import net.sf.ehcache.CacheManager;
    import net.sf.ehcache.Element;
    public class TerracottaExample {
    CacheManager cacheManager = new CacheManager();
      public TerracottaExample() {
        Cache cache = cacheManager.getCache("sampleTerracottaCache");
        int cacheSize = cache.getKeys().size();
        cache.put(new Element("" + cacheSize, cacheSize));
        for (Object key : cache.getKeys()) {
          System.out.println("Key:" + key);
        "/>
      "/>
      public static void main(String[] args) throws Exception {
        new TerracottaExample();
      "/>
    "/>


The above example looks for sampleTerracottaCache.
In ehcache.xml, we need to uncomment or add the following line:

<pre>
&lt;terracottaConfig url="localhost:9510"/&gt;
</pre>

This tells Ehcache to load the Terracotta server config from localhost port 9510. Note: You must have a
Terracotta 3.1.1 or higher server running locally for this example.

Next we want to enable Terracotta clustering for the cache named `sampleTerracottaCache`. Uncomment or add the
following in ehcache.xml.

<pre>
  &lt;cache name="sampleTerracottaCache"
     maxElementsInMemory="1000"
     eternal="false"
     timeToIdleSeconds="3600"
     timeToLiveSeconds="1800"
     overflowToDisk="false"&gt;
    &lt;terracotta/&gt;
  &lt/cache&gt;
</pre>

That's it!