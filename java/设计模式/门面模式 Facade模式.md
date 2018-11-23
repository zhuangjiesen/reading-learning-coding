# 门面模式 Facade模式


其实就是很多类都要进行先后的调用使用一个门面类来封装
我的理解 ： 每次去公司都要 开门，开灯，开电脑 这些东西
            门面模式就是将这些整合成一个 行为组合 -> 包含以上的东西


```

public class Facade {
    public void doOpen(){

        Door door = new Door();
        Light light = new Light();
        Computer computer = new Computer();

        door.open();
        light.open();
        computer.open();
    }

}



public class Door {
    public void open(){
        System.out.println("i am opening door ...");
    }

}



public class Light {
    public void open(){
        System.out.println("i am opening Light ...");
    }

}


public class Computer {
    public void open(){
        System.out.println("i am opening Computer ...");
    }

}



调用示例：

/**
 * Created by zhuangjiesen on 2017/10/25.
 */
public class DesignTest {


    public static void main(String[] args) {

        System.out.println("设计模式...");


        Facade facade = new Facade();
        // 一次解决
        Facade.doOpen();

    }


}


```


