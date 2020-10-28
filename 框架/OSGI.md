# OSGI



## OSGI概念

我们做软件开发一直在追求一个境界，就是 **模块之间的真正“解耦”、“分离”**，这样我们在软件的管理和开发上面就会更加的灵活，甚至包括给客户部署项目的时候都可以做到更加的灵活可控。但是我们以前使用SSH框架等架构模式进行产品开发的时候我们是达不到这种要求的。

所以我们“架构师”或顶尖的技术高手都在为模块化开发努力的摸索和尝试，然后我们的OSGI的技术规范就应运而生。

**OSGI是一种技术，它可以让不同的模块做到彻底的分离，而不是逻辑意义上的分离，是物理上的分离，也就是说在运行部署之后可以在不停止服务器的时候，直接把某些模块拿下来，其他模块的功能也不受影响。同样也可以在不停止服务器的时候，直接把某些模块加上去。**

由此，OSGI技术将来会变得非常的重要，因为它在实现模块化解耦的路上，走得比现在大家经常所用的SSH框架走的更远。这个技术在未来大规模、高访问、高并发的Java模块化开发领域，或者是项目规范化管理中，会大大超过SSH等框架的地位。

现在主流的一些应用服务器，Oracle的weblogic服务器，IBM的WebSphere，JBoss，还有Sun公司的glassfish服务器，都对OSGI提供了强大的支持，都是在OSGI的技术基础上实现的。有那么多的大型厂商支持OSGI这门技术，我们既可以看到OSGI技术的重要性。所以将来OSGI是将来非常重要的技术。

但是OSGI仍然脱离不了框架的支持，因为OSGI本身也使用了很多spring等框架的基本控件(因为要实现AOP依赖注入等功能)，但是哪个项目又不去依赖第三方jar呢？









## 参考

[1初识OSGI](https://blog.csdn.net/acmman/article/details/50848595?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522160386637719724836736389%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=160386637719724836736389&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_blog_default-1-50848595.pc_v2_rank_blog_default&utm_term=OSGI&spm=1018.2118.3001.4187)

[2走进OSGI](https://blog.csdn.net/acmman/article/details/50904044?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522160386637719724836736389%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=160386637719724836736389&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_blog_default-2-50904044.pc_v2_rank_blog_default&utm_term=OSGI&spm=1018.2118.3001.4187)

[3实战OSGI-01](https://blog.csdn.net/acmman/article/details/50906996?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522160386637719724836736389%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=160386637719724836736389&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_blog_default-4-50906996.pc_v2_rank_blog_default&utm_term=OSGI&spm=1018.2118.3001.4187)

[4实战OSGI-02](https://blog.csdn.net/acmman/article/details/50916011?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522160386637719724836736389%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=160386637719724836736389&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_blog_default-6-50916011.pc_v2_rank_blog_default&utm_term=OSGI&spm=1018.2118.3001.4187)

[5实战OSGI-03](https://blog.csdn.net/acmman/article/details/50935373?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522160386637719724836736389%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=160386637719724836736389&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_blog_default-5-50935373.pc_v2_rank_blog_default&utm_term=OSGI&spm=1018.2118.3001.4187)

[IDEA中开发OSGI](https://blog.csdn.net/qq_34248376/article/details/82585930)

