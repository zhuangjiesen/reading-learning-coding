# 责任链模式(Java)

### tomcat中的Filter实现


```

// 责任链的调度器
public interface FilterChain {

	public void initFilters(List<Filter> filters);

	public void doFilter (Object params);
	

}


// 过滤器 责任链上的一个零件
public interface Filter {
	
	public void doFilter (Object params , FilterChain filterChain);

}


public class DefaultFilterChain implements {

	private List<Filter> filters;
	private Iterator<Filter> iterator;

	public void initFilters(List<Filter> filters) {

		// 这里可以对 filter 进行排序， 通过Ordered 接口排序
		this.filters = filters ;
		iterator = filters.iterator;
	}

	public void doFilter (Object params) {
		// 这里可以进行判空操作
		if (iterator.hasNext()) {
			iterator.next().doFilter(params , this);
		}

	}
	



}



具体过滤器 1. 2. 3. 测试用

/**
 * Created by zhuangjiesen on 2017/10/26.
 */
public class CommonFilter1 implements Filter {
    @Override
    public void doFilter(Object params, FilterChain filterChain) {

        System.out.println("i am CommonFilter1 doFilter ....");

        filterChain.doFilter(params);

    }
}



/**
 * Created by zhuangjiesen on 2017/10/26.
 */
public class CommonFilter2 implements Filter {
    @Override
    public void doFilter(Object params, FilterChain filterChain) {
        System.out.println("i am CommonFilter2 doFilter ....");

    }
}



/**
 * Created by zhuangjiesen on 2017/10/26.
 */
public class CommonFilter3 implements Filter {
    @Override
    public void doFilter(Object params, FilterChain filterChain) {
        System.out.println("i am CommonFilter3 doFilter ....");

    }
}


调用 filterChain.doFilter(params); 责任链才能继续工作

调用示例：

/**
 * Created by zhuangjiesen on 2017/10/25.
 */
public class DesignTest {


    public static void main(String[] args) {

        System.out.println("设计模式...");


        FilterChain filterChain = new DefaultFilterChain();

        List<Filter> filters = new ArrayList<>();
        filters.add(new CommonFilter1());
        filters.add(new CommonFilter2());
        filters.add(new CommonFilter3());
		// 初始化
        filterChain.initFilters(filters);
        // 调用过滤器
        filterChain.doFilter("zhuangjiesen");

    }


}


// 责任链模式就是你只要实现 Filter 接口加入到队列中
如果需要下滤，可以调用 filterChain.doFilter(params); 
如果不需要下滤， 不调用就可以，就停在当前过滤器中



```




