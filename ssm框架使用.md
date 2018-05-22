SSM框架就是Spring，Spring MVC，MyBatis三个框架的缩写。

###注解

+ @Resource。@Resource(name=”XXX”)，就相当于为该属性注入一个id为XXX的 bean。
+ @Service。@Service(name="XXX")就是类似于在spring配置文件中配置一个bean对象，id为name属性指定的名字。
+ @Controller，用于标注控制层组件（如 Struts 中的 action）
+ @Repository，用于标注数据访问组件，即 DAO 层组件。和service作用相似，只是标明是数据访问层的组件
+ @Component，泛指组件，当组件不好归类的时候，咱们就可以用这个注解进行标注。


Service,Controller,Respository,Component这四个注解都是基于类的，咱们可以定义名称，也可以不定义名称。在不定义名称的时候，Spring 就会默认以类名且首字母小写的词组为 bean 的名称
























内容参考

[详述 @Service 和 @Resource 注解的区别](https://blog.csdn.net/qq_35246620/article/details/59484024)