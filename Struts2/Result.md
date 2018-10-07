# Result
很显然 Struts2 的默认结果是 jsp 但是它也支持自定义的结果来实现 Ajax

# 自定义的Result
它有个接口 Result 接口,想要实现自定义的返回类型就必须实现这个接口
```
public interface Result extends Serializable {
    public void execute(ActionInvocation invocation) throws Exception;
}
```
大致流程
```
public void execute(ActionInvocation invocation) throws Exception {
    // 获得 PrintWriter 对象
    PrintWriter out = ServletActionContext.getResponse();
    // 然后就是和servlet 一样了
    // 当然特色功能就是可以 通过 Struts2 的API 从不同地方拿对象比如说:值栈
    // 从值栈中获取对象然后根据对象生成 json
    out.println(json);
}
```
