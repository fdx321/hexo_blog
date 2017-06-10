title: 还在整天写if else ?
date: 2017-06-10 18:55:35
tags:
- Java
- Spring
---
### **缘起**
在业务团队写代码，经常会遇到这种情况，产品经理来个需求，好的，我们加个字段，加个if else. 谢晞鸣最近总结了一些比 if else 优雅一点的套路，能够让自己代码的逼格看起来稍微高点。很多时候分支是不可避免的，不同的业务，肯定要走不同的分支，我这里所谓的优雅套路，本质上就是用一个 Map 将 **“条件”** 和 **”业务处理逻辑“** 两者映射起来。

这里假设我们的项目是基于Spring的。文章中列的代码是不完整的，说明意思即可。

### **if else**
有这样一种业务场景，我们做保险产品项目，不同产品核保的逻辑是不一样的（对年龄、限额、职业、性别等等的要求都不一样），假设有两个字段， productCategory, productSubCategory 用于区分不同类型的保险(万能险、投连险、养老险、产险等)。我们可以这么写
```java
if (productCategory.equals("XXX")){
    processXXX();
} else if(productCategory.equals("YYY")){
    processYYY();
}
```

### **优雅的套路**
我们也可以设计这样一个结构来处理这种业务，将我们的 **”条件”（即产品类型）** 和 **“业务处理逻辑”（即核保逻辑）** 映射起来， 下面是一个很简单的 demo：<!--more-->，[源码地址](aaa)

![300](/images/还在整天写ifelse.png)

先来介绍各个package:
* resource, 项目入口层，可以理解成我们常见的controller、facade 之类的层
* service, 业务逻辑层
* registry, 我们的注册中心，负责业务逻辑的路由
* dto, 接口调用传递的数据对象
* common, 一些公共组件

后面的介绍中删减了get/set方法，先看service层
```java
public interface FdxProcessor {
    void process(FdxDto<?> fdxDto);
}
```

``` java
public class FdxDto<T> {
    private String productCategory;
    private String productSubCategory;
    private T param;
}
```

面向接口编程，每种类型的产品的处理逻辑都必须实现该接口，这里的 DTO 是一个泛型，为了支持各种类型的数据，也定义了一些必传的参数，比如这里的两个category.接下来是一个关键的抽象类，这里完成了业务逻辑的注册。

```java
public abstract class AbstractFdxProcessor implements FdxProcessor, InitializingBean, BeanNameAware {
    @Autowired
    private FdxProcessorRegistry fdxProcessorRegistry;
    protected String beanName;

    public void setBeanName(String name) {
        beanName = name;
    }

    protected abstract FdxProcessorRegistry.FdxKeyPair getKeyPair();

    @Override
    public void afterPropertiesSet() throws Exception {
        fdxProcessorRegistry.put(getKeyPair(), beanName);
    }
}
```
1. 实现业务接口FdxProcessor
2. 实现 BeanNameAware 接口，为了获得每个processor 在 Spring context 中的 beanName
3. 实现 InitializingBean 接口(这是 Spring 提供的一些生命周期接口中的一个),在 Spring 完成该Bean初始化之后，将 beanName 注册到注册中心去
4. 注意抽象方法 getKeyPair， 这是子类必须实现的，用于构造注册时用的key，即之前说的 **"条件"**。关于这个 **“条件”**，有多种实现方式，后面再细聊。

然后就是每种产品具体的业务逻辑实现：
```java
/**
 * XXX 产品的业务逻辑
 */
@Service
public class XxxFdxProcessorImpl extends AbstractFdxProcessor{
    @Override
    public void process(FdxDto<?> fdxDto) {
        //do business
    }

    @Override
    protected FdxProcessorRegistry.FdxKeyPair getKeyPair() {
        return new FdxProcessorRegistry.FdxKeyPair("XXX","XXX");
    }
}

/**
 * YYY 产品的业务逻辑
 */
@Service
public class YyyFdxProcessorImpl extends AbstractFdxProcessor {
    @Override
    public void process(FdxDto<?> fdxDto) {
        //do business
    }

    @Override
    protected FdxProcessorRegistry.FdxKeyPair getKeyPair() {
        return new FdxProcessorRegistry.FdxKeyPair("YYY", "YYY");
    }
}
```

接下来看下这个简单的注册中心，主要实现了
* 注册，将 key 和 beanName 映射起来
* 获取，通过 key 获得 beanName，然后通过 beanName 从 Spring Context 中获得真正的 bean
```java
@Component
public class FdxProcessorRegistry {
    private Map<FdxKeyPair, String> config = new HashMap<FdxKeyPair, String>();

    public void put(FdxKeyPair key, String beanName) {
        config.put(key, beanName);
    }

    public FdxProcessor get(FdxKeyPair key) {
        return SpringContextHolder.getBean(config.get(key));
    }

    public static class FdxKeyPair {
        private String productCategory;
        private String productSubCategory;

        public FdxKeyPair(String productCategory, String productSubCategory) {
            this.productCategory = productCategory;
            this.productSubCategory = productSubCategory;
        }

        @Override
        public int hashCode() {
            //TODO
            return 1;
        }

        @Override
        public boolean equals(Object obj) {
            //TODO
            return true;
        }
    }
}
```
入口层这样使用的就可以了.
```java
public void doFdx(FdxDto fdxDto) {
    fdxProcessorRegistry.get(new FdxKeyPair(fdxDto.getProductCategory(), fdxDto.getProductSubCategory())).process(fdxDto);
}
```

### **套路的变种**

最后，关于 **"条件"** 和 **“业务逻辑”** 之间的映射，可能会有这样一些其他的实现方式，

**1. 用于路由的条件比较简单**
比如说只有一个category， 我们可以定义一个 annotation，比如
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public  @interface FdxCategory {
    String productCategory() default {};
}
```
将它用在XxxFdxProcessorImpl这些子类上，然后稍微修改一下AbstractFdxProcessor
```java
public abstract class AbstractFdxProcessor implements FdxProcessor, BeanPostProcessor{
    @Autowired
    private FdxProcessorRegistry fdxProcessorRegistry;

    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
            FdxCategory fdxCategory = bean.getClass().getAnnotation(FdxCategory.class);
            //通过 fdxCategory 拿到 category, 作为 key 和 beanName 一起进入注册中心完成注册
        }
        return bean;
    }
}
```
**2. 用于路由的条件比较复杂**
可以自定义 Key 类，但是要注意重写好 equals 和 hashCode

**3. 每套逻辑可能适用于多个category**
这种情况可以将一些公共的逻辑抽象出来放在抽象类中，也可以在 **"条件"** 上做些文章，比如可以这么搞:
```java
@Service
public class XyzFdxProcessorImpl extends AbstractFdxProcessor {
    @Override
    protected List<FdxKeyPair> getKeyPair() {
      List<FdxKeyPair> list = new LinkedList<FdxKeyPair>();
      list.add (new FdxKeyPair("XXX", "XXX"));
      list.add (new FdxKeyPair("YYY", "YYY"));
      list.add (new FdxKeyPair("ZZZ", "ZZZ"));
      return list;
    }
}

public abstract class AbstractFdxProcessor implements FdxProcessor, InitializingBean, BeanNameAware {
    @Autowired
    private FdxProcessorRegistry fdxProcessorRegistry;
    protected String beanName;
    public void setBeanName(String name) {
        beanName = name;
    }

    protected abstract List<FdxKeyPair> getKeyPair();

    @Override
    public void afterPropertiesSet() throws Exception {
      List<FdxKeyPair> fdxKeyPairs =  getKeyPair();
      for(FdxKeyPair fdxKeyPair : fdxKeyPairs){
        fdxProcessorRegistry.put(fdxKeyPair, beanName);
      }
    }
}
```

### **最后**
这种优雅的套路，就是将显示的if else 转换成一个 通过 注册中心 路由的结构。注册中心维护 **"条件"** 和 **"业务逻辑”** 的映射，在注册的过程中充分利用了Spring提供的一些生命周期接口。
当然还有其他的一些变种，看具体情况而定，比如一些小的分支，我们可以路由到方法级别，而不是上面bean级别。实现上可以通过在方法上打annotation，然后通过反射的机制去调用。具体不细说了，有兴趣可以通过博客上留的联系方式和我交流。尼玛，说的好像有人看一下，我就是纯粹记笔记。





<style>
img[title="300"] {
  width:300px;
  width:300px;
  display: block;
}
img[title="500"] {
  width:500px;
  height:150px;
  display: block;
}
</style>
