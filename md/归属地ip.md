请求拦截器获取ip hutool的ServletUtil.*getClientIP*(request)

```
package com.abin.mallchat.custom.common.intecepter;

import cn.hutool.extra.servlet.ServletUtil;
import com.abin.mallchat.common.common.domain.dto.RequestInfo;
import com.abin.mallchat.common.common.utils.RequestHolder;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.Optional;

/**
 * 信息收集的拦截器
 */
@Order(1)
@Slf4j
@Component
public class CollectorInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        RequestInfo info = new RequestInfo();
        info.setUid(Optional.ofNullable(request.getAttribute(TokenInterceptor.ATTRIBUTE_UID)).map(Object::toString).map(Long::parseLong).orElse(null));
        info.setIp(ServletUtil.getClientIP(request));
        RequestHolder.set(info);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        RequestHolder.remove();
    }

}
```

在holder中获取

```
package com.abin.mallchat.common.common.utils;

import com.abin.mallchat.common.common.domain.dto.RequestInfo;

/**
 * Description: 请求上下文
 * Author: <a href="https://github.com/zongzibinbin">abin</a>
 * Date: 2023-04-05
 */
public class RequestHolder {

    private static final ThreadLocal<RequestInfo> threadLocal = new ThreadLocal<>();

    public static void set(RequestInfo requestInfo) {
        threadLocal.set(requestInfo);
    }

    public static RequestInfo get() {
        return threadLocal.get();
    }

    public static void remove() {
        threadLocal.remove();
    }
}

```



websocket获取ip





netty 配置一个pipline

```
                        //保存用户ip
                        pipeline.addLast(new HttpHeadersHandler());
```



建立连接只能获取一次

```
package com.abin.mallchat.custom.user.websocket;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.codec.http.FullHttpRequest;
import io.netty.handler.codec.http.HttpHeaders;
import io.netty.util.AttributeKey;
import org.apache.commons.lang3.StringUtils;

import java.net.InetSocketAddress;

public class HttpHeadersHandler extends ChannelInboundHandlerAdapter {
    private AttributeKey<String> key = AttributeKey.valueOf("Id");

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof FullHttpRequest) {
            HttpHeaders headers = ((FullHttpRequest) msg).headers();
            String ip = headers.get("X-Real-IP");
            if (StringUtils.isEmpty(ip)) {//如果没经过nginx，就直接获取远端地址
                InetSocketAddress address = (InetSocketAddress) ctx.channel().remoteAddress();
                ip = address.getAddress().getHostAddress();
            }
            NettyUtil.setAttr(ctx.channel(), NettyUtil.IP, ip);
        }
        ctx.fireChannelRead(msg);
    }
}
```



统一放到 RequestHolder

```
package com.abin.mallchat.custom.user.websocket;

import com.abin.mallchat.common.common.constant.MDCKey;
import com.abin.mallchat.common.common.domain.dto.RequestInfo;
import com.abin.mallchat.common.common.utils.RequestHolder;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.MDC;

import java.util.UUID;


@Slf4j
public class NettyCollectorHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        String tid = UUID.randomUUID().toString();
        MDC.put(MDCKey.TID, tid);
        RequestInfo info = new RequestInfo();
        info.setUid(NettyUtil.getAttr(ctx.channel(), NettyUtil.UID));
        info.setIp(NettyUtil.getAttr(ctx.channel(), NettyUtil.IP));
        RequestHolder.set(info);

        ctx.fireChannelRead(msg);
    }
}

```

淘宝ip解析



ip解析限流： 排队，重试，异步



用户刷新页面 用户认证token时进行ip获取解析

```
    /**
     * 登录成功，并更新状态
     */
    private void loginSuccess(Channel channel, User user, String token) {
        //更新上线列表
        online(channel, user.getId());
        //返回给用户登录成功
        boolean hasPower = iRoleService.hasPower(user.getId(), RoleEnum.CHAT_MANAGER);
        sendMsg(channel, WSAdapter.buildLoginSuccessResp(user, token, hasPower));
        //发送用户上线事件
        boolean online = userCache.isOnline(user.getId());
        if (!online) {
            user.setLastOptTime(new Date());
            user.refreshIp(NettyUtil.getAttr(channel, NettyUtil.IP));
            applicationEventPublisher.publishEvent(new UserOnlineEvent(this, user));
        }
    }
```

刷新ip User表里面

```
    /**
     * 最后上下线时间
     */
    @TableField(value = "ip_info", typeHandler = JacksonTypeHandler.class)
    private IpInfo ipInfo;
```



```
    public void refreshIp(String ip) {
        if (ipInfo == null) {
            ipInfo = new IpInfo();
        }
        ipInfo.refreshIp(ip);
    }
```

ip info

```
    public void refreshIp(String ip) {
        if (StringUtils.isEmpty(ip)) {
            return;
        }
        updateIp = ip;
        if (createIp == null) {
            createIp = ip;
        }
    }
```



监听上线刷新ip

```
    @Async
    @EventListener(classes = UserOnlineEvent.class)
    public void saveDB(UserOnlineEvent event) {
        User user = event.getUser();
        User update = new User();
        update.setId(user.getId());
        update.setLastOptTime(user.getLastOptTime());
        update.setIpInfo(user.getIpInfo());
        userDao.updateById(update);
        //更新用户ip详情
        ipService.refreshIpDetailAsync(user.getId());
    }
```



异步更新 获取淘宝接口

在这里在进行一次详细获取ip detail

```
package com.abin.mallchat.common.user.service.impl;

import cn.hutool.core.lang.TypeReference;
import cn.hutool.core.thread.NamedThreadFactory;
import cn.hutool.core.util.StrUtil;
import cn.hutool.http.HttpUtil;
import cn.hutool.json.JSONUtil;
import com.abin.mallchat.common.common.handler.GlobalUncaughtExceptionHandler;
import com.abin.mallchat.common.user.dao.UserDao;
import com.abin.mallchat.common.user.domain.dto.IpResult;
import com.abin.mallchat.common.user.domain.entity.IpDetail;
import com.abin.mallchat.common.user.domain.entity.IpInfo;
import com.abin.mallchat.common.user.domain.entity.User;
import com.abin.mallchat.common.user.service.IpService;
import com.abin.mallchat.common.user.service.cache.UserCache;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.Objects;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * Description: ip
 * Author: <a href="https://github.com/zongzibinbin">abin</a>
 * Date: 2023-04-18
 */
@Service
@Slf4j
public class IpServiceImpl implements IpService, DisposableBean {
    private static final ExecutorService EXECUTOR = new ThreadPoolExecutor(1, 1,
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<>(500),
            new NamedThreadFactory("refresh-ipDetail", null, false,
                    new GlobalUncaughtExceptionHandler()));

    @Autowired
    private UserDao userDao;
    @Autowired
    private UserCache userCache;


    @Override
    public void refreshIpDetailAsync(Long uid) {
        EXECUTOR.execute(() -> {
            User user = userDao.getById(uid);
            IpInfo ipInfo = user.getIpInfo();
            if (Objects.isNull(ipInfo)) {
                return;
            }
            String ip = ipInfo.needRefreshIp();
            if (StrUtil.isBlank(ip)) {
                return;
            }
            IpDetail ipDetail = TryGetIpDetailOrNullTreeTimes(ip);
            if (Objects.nonNull(ipDetail)) {
                ipInfo.refreshIpDetail(ipDetail);
                User update = new User();
                update.setId(uid);
                update.setIpInfo(ipInfo);
                userDao.updateById(update);
                userCache.userInfoChange(uid);
            } else {
                log.error("get ip detail fail ip:{},uid:{}", ip, uid);
            }

        });
    }

    private static IpDetail TryGetIpDetailOrNullTreeTimes(String ip) {
        for (int i = 0; i < 3; i++) {
            IpDetail ipDetail = getIpDetailOrNull(ip);
            if (Objects.nonNull(ipDetail)) {
                return ipDetail;
            }
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        return null;
    }

    public static IpDetail getIpDetailOrNull(String ip) {
        String body = HttpUtil.get("https://ip.taobao.com/outGetIpInfo?ip=" + ip + "&accessKey=alibaba-inc");
        try {
            IpResult<IpDetail> result = JSONUtil.toBean(body, new TypeReference<IpResult<IpDetail>>() {
            }, false);
            if (result.isSuccess()) {
                return result.getData();
            }
        } catch (Exception ignored) {
        }
        return null;
    }

    //测试耗时结果 100次查询总耗时约100s，平均一次成功查询需要1s,可以接受
    //第99次成功,目前耗时：99545ms
    public static void main(String[] args) {
        Date begin = new Date();
        for (int i = 0; i < 100; i++) {
            int finalI = i;
            EXECUTOR.execute(() -> {
                IpDetail ipDetail = TryGetIpDetailOrNullTreeTimes("113.90.36.126");
                if (Objects.nonNull(ipDetail)) {
                    Date date = new Date();
                    System.out.println(String.format("第%d次成功,目前耗时：%dms", finalI, (date.getTime() - begin.getTime())));
                }
            });
        }
    }

    @Override
    public void destroy() throws InterruptedException {
        EXECUTOR.shutdown();
        if (!EXECUTOR.awaitTermination(30, TimeUnit.SECONDS)) {//最多等30秒，处理不完就拉倒
            if (log.isErrorEnabled()) {
                log.error("Timed out while waiting for executor [{}] to terminate", EXECUTOR);
            }
        }
    }

}

```

解释：

- ```
  创建了一个线程池对象，其中核心线程数和最大线程数都为1，任务队列容量为500，线程名称为"refresh-ipDetail"，未捕获异常处理器为`GlobalUncaughtExceptionHandler`。这个线程池可以用于执行异步任务。
  
  - `1`：核心线程数，即线程池中保持活动状态的线程数量。
  - `1`：最大线程数，即线程池中允许创建的最大线程数量。
  - `0L`：超时时间，即任务队列中等待执行的任务在多长时间后会被拒绝执行。这里设置为0，表示不设置超时时间。
  - `TimeUnit.MILLISECONDS`：时间单位，表示超时时间以毫秒为单位。
  - `new LinkedBlockingQueue<>(500)`：任务队列，即用于存储等待执行的任务的队列。这里使用了一个容量为500的`LinkedBlockingQueue`，它是一个基于链表实现的阻塞队列。
  - `new NamedThreadFactory("refresh-ipDetail", null, false, new GlobalUncaughtExceptionHandler())`：线程工厂，即用于创建新线程的工厂类。其中`"refresh-ipDetail"`是线程的名称前缀，`null`表示不设置线程的优先级，`false`表示不设置线程的守护属性，`new GlobalUncaughtExceptionHandler()`表示设置一个全局的未捕获异常处理器。
  ```

- 淘宝接口

- ```
      public static IpDetail getIpDetailOrNull(String ip) {
          String body = HttpUtil.get("https://ip.taobao.com/outGetIpInfo?ip=" + ip + "&accessKey=alibaba-inc");
          try {
              IpResult<IpDetail> result = JSONUtil.toBean(body, new TypeReference<IpResult<IpDetail>>() {
              }, false);
              if (result.isSuccess()) {
                  return result.getData();
              }
          } catch (Exception ignored) {
          }
          return null;
      }
  ```

  重试

  ```
      private static IpDetail TryGetIpDetailOrNullTreeTimes(String ip) {
          for (int i = 0; i < 3; i++) {
              IpDetail ipDetail = getIpDetailOrNull(ip);
              if (Objects.nonNull(ipDetail)) {
                  return ipDetail;
              }
              try {
                  Thread.sleep(2000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
          }
          return null;
      }
  ```

  

刷新ip

```
    public void refreshIpDetail(IpDetail ipDetail) {
        if (Objects.equals(createIp, ipDetail.getIp())) {
            createIpDetail = ipDetail;
        }
        if (Objects.equals(updateIp, ipDetail.getIp())) {
            updateIpDetail = ipDetail;
        }
    }
```

