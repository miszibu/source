---
layout: springcloud
title: SpringCloud Feign
date: 2018-12-04 09:51:49
tags: SpringCloud
---

> 尚在🚧 尽请期待
>
> 在微服务应用场景中，不同模块相互调用，不再是传统项目中的函数调用，而是跨服务器的Http请求。倘若每次请求都需要编写Http请求头，未免太过于冗杂。Feign的作用就是为了解决这个问题，自动封装了HTTP请求，做到了以声明式调用的方式，调用其他模块。

<!--more-->

## 1. 开始使用

```Java
@EnableFeignClients
@EnableDiscoveryClient
@EnableEurekaClient
@SpringBootApplication
public class OrderServiceApplication {
	public static void main(String[] args) {
		SpringApplication.run(OrderServiceApplication.class, args);
	}

}
```

**@EnableFeignClients** 启用Feign

```java
@FeignClient(value = "order-service")
public interface OrderService {
    @RequestMapping(value = "/",method = RequestMethod.GET)
    String placeOrder(@RequestParam("score") Integer score,@RequestParam("sku") String sku);
}

public class UserController {

    @Autowired
    private OrderService orderService;

    @RequestMapping("/info")
    public UserDTO getCurrentInfo(){
        return orderService.get(1L);
    }
}
```

**@FeignClient** 声明该模块的服务名称，用于创建Ribbon的负载均衡客户端，

其他模块即可直接声明式调用该模块了。 Ribbon客户端会根据服务名称，从EurekaServer获取对应的服务器IP和端口，默认使用Round Robin轮询算法负载均衡 请求服务器，Feign会自动创建请求，发送请求，解析请求，返回结果。

## 2.