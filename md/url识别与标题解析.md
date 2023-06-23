jsoup解析url获取标题 责任链





定义接口

```
package com.abin.mallchat.common.common.utils.discover;

import cn.hutool.core.date.StopWatch;
import org.jsoup.nodes.Document;

import javax.annotation.Nullable;
import java.util.Map;

public interface UrlTitleDiscover {


    @Nullable
    Map<String, String> getContentTitleMap(String content);


    @Nullable
    String getUrlTitle(String url);

    @Nullable
    String getDocTitle(Document document);

    public static void main(String[] args) {//用异步多任务查询并合并 974 //串行访问的速度1349  1291  1283 1559
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        String longStr = "这是一个很长的字符串再来 www.github.com，其中包含一个URL www.baidu.com,, 一个带有端口号的URL http://www.jd.com:80, 一个带有路径的URL http://mallchat.cn, 还有美团技术文章https://mp.weixin.qq.com/s/hwTf4bDck9_tlFpgVDeIKg ";
        PrioritizedUrlTitleDiscover discover = new PrioritizedUrlTitleDiscover();
        Map<String, String> contentTitleMap = discover.getContentTitleMap(longStr);
        System.out.println(contentTitleMap);
//
//        Jsoup.connect("http:// www.github.com");
        stopWatch.stop();
        long cost = stopWatch.getTotalTimeMillis();
        System.out.println(cost);
    }//{http://mallchat.cn=MallChat, www.baidu.com=百度一下，你就知道, https://mp.weixin.qq.com/s/hwTf4bDck9_tlFpgVDeIKg=超大规模数据库集群保稳系列之二：数据库攻防演练建设实践, http://www.jd.com:80=京东(JD.COM)-正品低价、品质保障、配送及时、轻松购物！}
}

```

抽象类继承

```
package com.abin.mallchat.common.common.utils.discover;

import cn.hutool.core.util.ReUtil;
import cn.hutool.core.util.StrUtil;
import com.abin.mallchat.common.common.utils.FutureUtils;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.jsoup.Connection;
import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.springframework.data.util.Pair;

import javax.annotation.Nullable;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.CompletableFuture;
import java.util.regex.Pattern;
import java.util.stream.Collectors;

/**
 * Description: urlTitle查询抽象类
 * Author: <a href="https://github.com/zongzibinbin">abin</a>
 * Date: 2023-05-27
 */
@Slf4j
public abstract class AbstractUrlTitleDiscover implements UrlTitleDiscover {
    //链接识别的正则
    private static final Pattern PATTERN = Pattern.compile("((http|https)://)?(www.)?([\\w_-]+(?:(?:\\.[\\w_-]+)+))([\\w.,@?^=%&:/~+#-]*[\\w@?^=%&/~+#-])?");

    @Nullable
    @Override
    public Map<String, String> getContentTitleMap(String content) {
        if (StrUtil.isBlank(content)) {
            return new HashMap<>();
        }
        List<String> matchList = ReUtil.findAll(PATTERN, content, 0);
        //并行请求
        List<CompletableFuture<Pair<String, String>>> futures = matchList.stream().map(match -> CompletableFuture.supplyAsync(() -> {
            String title = getUrlTitle(match);
            return StringUtils.isNotEmpty(title) ? Pair.of(match, title) : null;
        })).collect(Collectors.toList());
        CompletableFuture<List<Pair<String, String>>> future = FutureUtils.sequenceNonNull(futures);
        //结果组装
        return future.join().stream().collect(Collectors.toMap(Pair::getFirst, Pair::getSecond, (a, b) -> a));
    }

    @Nullable
    @Override
    public String getUrlTitle(String url) {
        Document document = getUrlDocument(assemble(url));
        if (Objects.isNull(document)) {
            return null;
        }
        return getDocTitle(document);
    }

    private String assemble(String url) {
        if (!StrUtil.startWith(url, "http")) {
            return "http://" + url;
        }
        return url;
    }

    protected Document getUrlDocument(String matchUrl) {
        try {
            Connection connect = Jsoup.connect(matchUrl);
            connect.timeout(2000);
            return connect.get();
        } catch (Exception e) {
            log.error("find title error:url:{}", matchUrl, e);
        }
        return null;
    }
}

```

一个是普调url获取标题就行 公众号的不一样

```
package com.abin.mallchat.common.common.utils.discover;

import org.jsoup.nodes.Document;

/**
 * Description: 通用的标题解析类
 * Author: <a href="https://github.com/zongzibinbin">abin</a>
 * Date: 2023-05-27
 */
public class CommonUrlTitleDiscover extends AbstractUrlTitleDiscover {
    @Override
    public String getDocTitle(Document document) {
        return document.title();
    }
}

```

```
package com.abin.mallchat.common.common.utils.discover;

import org.jsoup.nodes.Document;

/**
 * Description: 针对微信公众号文章的标题获取类
 * Author: <a href="https://github.com/zongzibinbin">abin</a>
 * Date: 2023-05-27
 */
public class WxUrlTitleDiscover extends AbstractUrlTitleDiscover {
    @Override
    public String getDocTitle(Document document) {
        return document.getElementsByAttributeValue("property", "og:title").attr("content");
    }
}

```

遍历循环获取方法

```
package com.abin.mallchat.common.common.utils.discover;

import cn.hutool.core.util.StrUtil;
import org.jsoup.nodes.Document;

import java.util.ArrayList;
import java.util.List;

/**
 * Description: 具有优先级的title查询器
 * Author: <a href="https://github.com/zongzibinbin">abin</a>
 * Date: 2023-05-27
 */
public class PrioritizedUrlTitleDiscover extends AbstractUrlTitleDiscover {

    private final List<UrlTitleDiscover> urlTitleDiscovers = new ArrayList<>(2);

    public PrioritizedUrlTitleDiscover() {
        urlTitleDiscovers.add(new CommonUrlTitleDiscover());
        urlTitleDiscovers.add(new WxUrlTitleDiscover());
    }

    @Override
    public String getDocTitle(Document document) {
        for (UrlTitleDiscover urlTitleDiscover : urlTitleDiscovers) {
            String urlTitle = urlTitleDiscover.getDocTitle(document);
            if (StrUtil.isNotBlank(urlTitle)) {
                return urlTitle;
            }
        }
        return null;
    }
}

```



```
    /**
     * 发送消息
     */
    @Override
    @Transactional
    public Long sendMsg(ChatMessageReq request, Long uid) {
        AbstractMsgHandler msgHandler = MsgHandlerFactory.getStrategyNoNull(request.getMsgType());//todo 这里先不扩展，后续再改
        msgHandler.checkMsg(request, uid);
        //同步获取消息的跳转链接标题
        Message insert = MessageAdapter.buildMsgSave(request, uid);
        messageDao.save(insert);
        msgHandler.saveMsg(insert, request);
        //发布消息发送事件
        applicationEventPublisher.publishEvent(new MessageSendEvent(this, insert.getId()));
        return insert.getId();
    }
```

这里调用check方法，

```
    @Autowired
    private static final PrioritizedUrlTitleDiscover URL_TITLE_DISCOVER = new PrioritizedUrlTitleDiscover();

    @Override
    MessageTypeEnum getMsgTypeEnum() {
        return MessageTypeEnum.TEXT;
    }


    @Override
    public void saveMsg(Message msg, ChatMessageReq request) {//插入文本内容
        TextMsgReq body = BeanUtil.toBean(request.getBody(), TextMsgReq.class);
        MessageExtra extra = Optional.ofNullable(msg.getExtra()).orElse(new MessageExtra());
        Message update = new Message();
        update.setId(msg.getId());
        update.setContent(SensitiveWordUtils.filter(body.getContent()));
        update.setExtra(extra);
        //如果有回复消息
        if (Objects.nonNull(body.getReplyMsgId())) {
            Integer gapCount = messageDao.getGapCount(request.getRoomId(), body.getReplyMsgId(), msg.getId());
            update.setGapCount(gapCount);
            update.setReplyMsgId(body.getReplyMsgId());

        }
        //判断消息url跳转
        Map<String, String> urlTitleMap = URL_TITLE_DISCOVER.getContentTitleMap(body.getContent());
        extra.setUrlTitleMap(urlTitleMap);
        //艾特功能
        if (CollectionUtils.isNotEmpty(body.getAtUidList())) {
            extra.setAtUidList(body.getAtUidList());

        }

        messageDao.updateById(update);
    }
```

并行解析

```
        //并行请求
        List<CompletableFuture<Pair<String, String>>> futures = matchList.stream().map(match -> CompletableFuture.supplyAsync(() -> {
            String title = getUrlTitle(match);
            return StringUtils.isNotEmpty(title) ? Pair.of(match, title) : null;
        })).collect(Collectors.toList());
        CompletableFuture<List<Pair<String, String>>> future = FutureUtils.sequenceNonNull(futures);
        //结果组装
        return future.join().stream().collect(Collectors.toMap(Pair::getFirst, Pair::getSecond, (a, b) -> a));
```

1. `matchList` 是一个包含要请求的匹配对象的列表，通过 `stream()` 方法将其转换为流。
2. 使用 `map()` 方法将每个匹配对象转换为一个 CompletableFuture 对象。`CompletableFuture.supplyAsync()` 方法会异步地执行 `getUrlTitle()` 方法获取标题，并返回一个包含标题和匹配对象的 Pair 对象。如果标题为空，则返回 null。
3. 使用 `collect()` 方法将所有的 CompletableFuture 对象收集到一个列表中。
4. 使用自定义的 `FutureUtils.sequenceNonNull()` 方法将列表中的 CompletableFuture 对象组合为一个 CompletableFuture，该 CompletableFuture 的结果是一个包含非空结果的列表。
5. 使用 `join()` 方法等待 CompletableFuture 的完成，并通过流操作将列表中的 Pair 对象转换为一个 Map，以匹配对象为键，标题为值。



```

    /**
     * 将List<CompletableFuture<T>> 转为 CompletableFuture<List<T>>，并过滤调null值
     */
    public static <T> CompletableFuture<List<T>> sequenceNonNull(Collection<CompletableFuture<T>> completableFutures) {
        return CompletableFuture.allOf(completableFutures.toArray(new CompletableFuture<?>[0]))
                .thenApply(v -> completableFutures.stream()
                        .map(CompletableFuture::join)
                        .filter(Objects::nonNull)
                        .collect(Collectors.toList())
                );
    }
```

