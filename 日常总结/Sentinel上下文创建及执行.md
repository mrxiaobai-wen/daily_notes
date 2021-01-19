Sentinel上下文创建及执行，入口示例代码：

~~~java
public static void fun() {
    Entry entry = null;
    try {
        entry = SphU.entry(SOURCE_KEY);
    } catch (BlockException e1) {
    } finally {
        if (entry != null) {
            entry.exit();
        }
    }
}
~~~

***



## 执行entry

在执行SphU.entry时获取Entry，Entry代表当前调用的入口，用来保存当前调用信息。

进入到SphU.entry方法可以发现，Entry的获取使用的是Sph的默认实现CtSph。Sph是资源统计和规则检查的接口定义。

~~~java
public class Env {

    public static final Sph sph = new CtSph();

    static {
        // If init fails, the process will exit.
        InitExecutor.doInit();
    }
}
~~~

进到CtSph.entry方法：

~~~java
@Override
public Entry entry(String name, EntryType type, int count, Object... args) throws BlockException {
    StringResourceWrapper resource = new StringResourceWrapper(name, type);
    return entry(resource, count, args);
}
~~~

可以看出第一步是创建一个当前资源的包装类，然后将标识当前请求资源的包装类传进entry方法获取Entry。值得一提的是StringResourceWrapper继承自ResourceWrapper。而ResourceWrapper重新了hashCode和equals方法，如下：

~~~java
@Override
public int hashCode() {
    return getName().hashCode();
}

@Override
public boolean equals(Object obj) {
    if (obj instanceof ResourceWrapper) {
        ResourceWrapper rw = (ResourceWrapper)obj;
        return rw.getName().equals(getName());
    }
    return false;
}
~~~

可以看出比较两个Warpper是否指向同一个资源，主要是比较的name，只要获取的资源名相同那么就是要求获取同一个资源，这一点在后面有用。

然后回到CtSph.entry方法，最终进入到了CtSph.entryWithPriority方法，代码如下：

~~~java
private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
    throws BlockException {
    // 1.从当前线程获取context
    Context context = ContextUtil.getContext();
    if (context instanceof NullContext) {
        // The {@link NullContext} indicates that the amount of context has exceeded the threshold,
        // so here init the entry only. No rule checking will be done.
        return new CtEntry(resourceWrapper, null, context);
    }

    if (context == null) {
        // Using default context.
        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
    }

    // Global switch is close, no rule checking will do.
    if (!Constants.ON) {
        return new CtEntry(resourceWrapper, null, context);
    }

    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

    /*
     * Means amount of resources (slot chain) exceeds {@link Constants.MAX_SLOT_CHAIN_SIZE},
     * so no rule checking will be done.
     */
    if (chain == null) {
        return new CtEntry(resourceWrapper, null, context);
    }

    Entry e = new CtEntry(resourceWrapper, chain, context);
    try {
        chain.entry(context, resourceWrapper, null, count, prioritized, args);
    } catch (BlockException e1) {
        e.exit(count, args);
        throw e1;
    } catch (Throwable e1) {
        // This should not happen, unless there are errors existing in Sentinel internal.
        RecordLog.info("Sentinel unexpected exception", e1);
    }
    return e;
}
~~~

该方法做了如下几件事：

1. 首先尝试从当前线程获取context，可以看ContextUtil.getContext方法：

   ~~~java
   public static Context getContext() {
       return contextHolder.get();
   }
   ~~~

   查看contextHolder属性是一个ThreadLocal:

   ~~~java
   /**
    * Store the context in ThreadLocal for easy access.
    */
   private static ThreadLocal<Context> contextHolder = new ThreadLocal<>();
   ~~~

2. 判断当前线程上下文是否超出了阈值，也就是下面语句：

   ~~~java
   if (context instanceof NullContext)
   ~~~

   我们可以看看NullContext的定义：

   ~~~java
   /**
    * If total {@link Context} exceed {@link Constants#MAX_CONTEXT_NAME_SIZE}, a
    * {@link NullContext} will get when invoke {@link ContextUtil}.enter(), means
    * no rules checking will do.
    *
    * @author qinan.qn
    */
   public class NullContext extends Context {
   
       public NullContext() {
       	super(null, "null_context_internal");
       }
   }
   ~~~

   当上面判断为真时，那么就不再进行规则检查。

3. 当从当前线程获取的Context为空时，创建新的Context。**（这里在下面再详细解读。） **

4. 获取当前资源对应的Slot执行链：

   ~~~java
   ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);
   ~~~

   在获取执行链的方法：CtSph.lookProcessChain：

   ~~~java
   ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
       ProcessorSlotChain chain = chainMap.get(resourceWrapper);
       if (chain == null) {
           synchronized (LOCK) {
               chain = chainMap.get(resourceWrapper);
               if (chain == null) {
                   // Entry size limit.
                   if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                       return null;
                   }
   
                   chain = SlotChainProvider.newSlotChain();
                   Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
                       chainMap.size() + 1);
                   newMap.putAll(chainMap);
                   newMap.put(resourceWrapper, chain);
                   chainMap = newMap;
               }
           }
       }
       return chain;
   }
   ~~~

   其中chainMap定义：

   ~~~java
   private static volatile Map<ResourceWrapper, ProcessorSlotChain> chainMap
           = new HashMap<ResourceWrapper, ProcessorSlotChain>();
   ~~~

   可以看出每个资源都是对应的一个执行链，在chainMap中就是用ResourceWrapper做为键类型，而我们上面已经看到了ResourceWrapper重写了hashCode和equals方法，所以唯一确定一个资源的就是资源名。

5. 执行Slot链，如果规则检查未通过那么抛出BlockException异常，否则代表符合规则进入成功。

然后接下来看一下Context的创建，也就是下面这段代码：

~~~java
if (context == null) {
    // Using default context.
    context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
}
~~~

跟踪代码，最终进入到了ContextUtil.trueEnter方法。在阅读ContextUtil.trueEnte方法时，有必要先看一张图，来理清一下线程thread和Context、Context和Node之间的关系：

图片来源（该文章可以一看）：https://www.jianshu.com/p/e39ac47cd893

![img](https://upload-images.jianshu.io/upload_images/3397380-a53df77b1587fad1.png?imageMogr2/auto-orient/strip|imageView2/2/w/663/format/webp)

前面代表的是3个线程，可以看成他们都是获取helloWorld资源，可以看出每一个线程在执行的时候都是独立的创建了一个Context，每一个线程里面的Context都是对应到了一个点EntranceNode上，而该EntranceNode则是用于存储一个资源的信息。梳理了这三者之间的关系，那么接下来看ContextUtil.trueEnt方法，代码如下：

~~~java
protected static Context trueEnter(String name, String origin) {
    Context context = contextHolder.get();
    if (context == null) {
        Map<String, DefaultNode> localCacheNameMap = contextNameNodeMap;
        DefaultNode node = localCacheNameMap.get(name);
        if (node == null) {
            if (localCacheNameMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
                setNullContext();
                return NULL_CONTEXT;
            } else {
                LOCK.lock();
                try {
                    node = contextNameNodeMap.get(name);
                    if (node == null) {
                        if (contextNameNodeMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
                            setNullContext();
                            return NULL_CONTEXT;
                        } else {
                            node = new EntranceNode(new StringResourceWrapper(name, EntryType.IN), null);
                            // Add entrance node.
                            Constants.ROOT.addChild(node);

                            Map<String, DefaultNode> newMap = new HashMap<>(contextNameNodeMap.size() + 1);
                            newMap.putAll(contextNameNodeMap);
                            newMap.put(name, node);
                            contextNameNodeMap = newMap;
                        }
                    }
                } finally {
                    LOCK.unlock();
                }
            }
        }
        context = new Context(node, name);
        context.setOrigin(origin);
        contextHolder.set(context);
    }

    return context;
}
~~~

这个方法做的事情如下：

1. 从contextHolder中获取Context，如果有了那么就直接返回。

2. 没有Context信息，那么准备开始创建该上下文信息，准备工作：从contextNameNodeMap中获取对应节点，contextNameNodeMap定义如下：

   ~~~java
   private static volatile Map<String, DefaultNode> contextNameNodeMap = new HashMap<>();
   ~~~

   这个map中是上下文名和节点的对应关系，而上下文名即是资源名。

3. 如果获取到了这个节点，那么直接创建一个Context并设置到contextHolder中，然后直接返回。

4. 当上面节点不存在，那么先创建该节点，逻辑如下：

   1. 先检查当前上下文数是否超过指定阈值，如果超过了那么返回NullContext，本次请求不做规则检查。

   2. 没有超过指定阈值，那么加锁，进行双重检查。

   3. 使用当前资源创建节点，将创建的节点关联到根节点下，然后存入contextNameNodeMap中。然后创建Context并返回。可以看看在新增节点的时候，它的做法是在原map的基础上新建一个size+1的新map，然后将原map的所有节点信息加入新map中，同时保存新节点信息：

      ~~~java
      Map<String, DefaultNode> newMap = new HashMap<>(contextNameNodeMap.size() + 1);
      newMap.putAll(contextNameNodeMap);
      newMap.put(name, node);
      contextNameNodeMap = newMap;
      ~~~

      而我们的contextNameNodeMap属性是用volatile进行修饰的，当contextNameNodeMap引用的值发生变更时，能够立即对其它线程可见。

      那么为什么不在原来的contextNameNodeMap中直接加入新节点，而要新建map然后进行一次复制呢？

做完上面这些事情后，我们就得到了需要的Context。



## 执行exit

不管是上面示例代码中的finally里面的entry.exit()调用，还是CtSph.entryWithPriority方法中调用的e.exit(count, args)方法，最终都是在CtEntry.exitForContext方法中执行，代码如下：

~~~java
protected void exitForContext(Context context, int count, Object... args) throws ErrorEntryFreeException {
    if (context != null) {
        // Null context should exit without clean-up.
        if (context instanceof NullContext) {
            return;
        }

        if (context.getCurEntry() != this) {
            String curEntryNameInContext = context.getCurEntry() == null ? null
                : context.getCurEntry().getResourceWrapper().getName();
            // Clean previous call stack.
            CtEntry e = (CtEntry) context.getCurEntry();
            while (e != null) {
                e.exit(count, args);
                e = (CtEntry) e.parent;
            }
            String errorMessage = String.format("The order of entry exit can't be paired with the order of entry"
                    + ", current entry in context: <%s>, but expected: <%s>", curEntryNameInContext,
                resourceWrapper.getName());
            throw new ErrorEntryFreeException(errorMessage);
        } else {
            // Go through the onExit hook of all slots.
            if (chain != null) {
                chain.exit(context, resourceWrapper, count, args);
            }
            // Go through the existing terminate handlers (associated to this invocation).
            callExitHandlersAndCleanUp(context);

            // Restore the call stack.
            context.setCurEntry(parent);
            if (parent != null) {
                ((CtEntry) parent).child = null;
            }
            if (parent == null) {
                // Default context (auto entered) will be exited automatically.
                if (ContextUtil.isDefaultContext(context)) {
                    ContextUtil.exit();
                }
            }
            // Clean the reference of context in current entry to avoid duplicate exit.
            clearEntryContext();
        }
    }
}
~~~

逻辑比较简单，执行chain.exit，清空context等。











