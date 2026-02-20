---
title: Builder和Setter
published: 2026-02-20
description: Builder和Setter
image: "api"
tags: [java]
category: java
draft: false
---



**Builder 与 setter 写法的本质区别（6 点）：**

| 维度                 | setter 写法                                                  | Builder 写法                                                 |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **对象可变性**       | build 后仍可随时 `setXxx()` 修改，线程不安全                 | `build()` 后字段全是 `final`，不可变，天然线程安全           |
| **必填字段保证**     | 调用方可能忘记设置，运行时才 NPE                             | 必填参数放在 `Builder(method, url)` 构造方法，编译器强制传入 |
| **组合操作**         | `jsonBody` 需要调用方自己同时调 `setBody` + `setContentType`，容易漏 | [`jsonBody()`](learn/src/main/java/com/shai/builder02/HttpRequest.java:107) 一次调用原子完成两件事 |
| **单位/格式转换**    | `setTimeoutMs(30 * 1000)` 调用方自己换算，容易出错           | [`timeoutSeconds(30)`](learn/src/main/java/com/shai/builder02/HttpRequest.java:118) 内部换算，调用方只关心业务语义 |
| **整体校验**         | 无法在"所有字段都设置完之后"做整体校验                       | [`build()`](learn/src/main/java/com/shai/builder02/HttpRequest.java:136) 统一做最终校验（如 POST 必须有 body） |
| **派生字段自动生成** | 调用方手动设置 requestId/createdAt，容易忘                   | `build()` 自动生成，调用方无感知                             |

**结论**：Builder 的核心价值不是"链式调用好看"，而是**将对象构建过程中的校验、转换、组合操作封装在 Builder 内部，保证最终对象的完整性和不可变性**。

```java
package com.shai.builder02;

/**
 * Builder 模式 vs 普通 setter 写法对比
 *
 * <p>本类通过同一个场景（构建 HttpRequest）展示两种写法的本质差异。
 */
public class BuilderVsSetterComparison {

    // =========================================================================
    // 方式一：传统 setter 写法
    // =========================================================================

    /**
     * 传统 setter 写法的 HttpRequest（可变对象）
     *
     * <p>问题：
     * <ol>
     *   <li><b>对象可变</b>：build 完之后任何人都能继续调用 setter 修改状态，线程不安全</li>
     *   <li><b>无法保证完整性</b>：调用方可能忘记设置必填字段，编译器不报错，运行时才崩</li>
     *   <li><b>校验分散</b>：每个 setter 各自校验，或者根本没有校验，逻辑散落各处</li>
     *   <li><b>顺序无约束</b>：setter 可以任意顺序调用，无法表达"先设置 A 才能设置 B"的依赖关系</li>
     *   <li><b>无法做组合操作</b>：jsonBody() 需要同时设置 body + Content-Type，setter 做不到原子性</li>
     * </ol>
     */
    static class MutableHttpRequest {
        private String method;
        private String url;
        private String body;
        private String contentType;
        private int timeoutMs;

        // setter 只是简单赋值，无法做组合操作
        public void setMethod(String method) { this.method = method; }
        public void setUrl(String url) { this.url = url; }
        public void setBody(String body) { this.body = body; }
        public void setContentType(String contentType) { this.contentType = contentType; }
        public void setTimeoutMs(int timeoutMs) { this.timeoutMs = timeoutMs; }

        @Override
        public String toString() {
            return "MutableHttpRequest{method=" + method + ", url=" + url
                    + ", body=" + body + ", contentType=" + contentType
                    + ", timeoutMs=" + timeoutMs + "}";
        }
    }

    // =========================================================================
    // 方式二：Builder 写法（见 HttpRequest.java）
    // =========================================================================
    //
    // Builder 的核心价值不是"链式调用好看"，而是：
    //
    // 1. 【不可变对象】build() 之后字段全是 final，任何人无法修改，天然线程安全
    //
    // 2. 【构造完整性保证】必填参数放在 Builder 构造方法里，编译器强制传入
    //    new Builder("GET", "https://...") ← 不传就编译报错
    //
    // 3. 【集中校验 + 组合操作】
    //    jsonBody(json) 一次调用同时完成：
    //      - 设置 body
    //      - 自动追加 Content-Type 头
    //    setter 写法需要调用方自己记得同时调两个 setter，容易漏
    //
    // 4. 【单位/格式转换封装在 Builder 内部】
    //    timeoutSeconds(30) → 内部换算为 30000ms，调用方不需要关心单位
    //    setter 写法：setTimeoutMs(30 * 1000) ← 调用方自己算，容易出错
    //
    // 5. 【build() 做最终整体校验】
    //    POST 必须有 body，这个规则只在 build() 里写一次，
    //    setter 写法无法在"所有字段都设置完之后"做整体校验
    //
    // 6. 【自动生成派生字段】
    //    build() 自动生成 requestId、createdAt，
    //    setter 写法需要调用方手动设置，或者在某个 init() 方法里，容易忘

    public static void main(String[] args) {
        System.out.println("=== 方式一：setter 写法（问题演示）===");

        MutableHttpRequest req1 = new MutableHttpRequest();
        req1.setMethod("POST");
        req1.setUrl("https://api.example.com/users");
        req1.setBody("{\"username\":\"shai\"}");
        // 问题1：调用方必须自己记得同时设置 Content-Type，容易漏
        req1.setContentType("application/json");
        // 问题2：调用方自己换算单位，容易出错
        req1.setTimeoutMs(30 * 1000);
        // 问题3：build 完之后还能继续修改，破坏对象状态
        req1.setMethod("DELETE"); // 随时可以改，不安全
        System.out.println(req1);

        System.out.println("\n=== 方式二：Builder 写法（HttpRequest）===");

        HttpRequest req2 = HttpRequest
                .newRequest("POST", "https://api.example.com/users")
                // jsonBody 一次调用同时设置 body + Content-Type，原子操作
                .jsonBody("{\"username\":\"shai\"}")
                // 传秒，内部自动换算毫秒
                .timeoutSeconds(30)
                // build() 自动生成 requestId 和 createdAt
                .build();
        // req2 的字段全是 final，build 之后无法修改，安全
        System.out.println(req2);
    }
}

```

```java
package com.shai.builder02;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Builder 进阶示例：HttpRequest 构建器
 *
 * <p>演示 Builder 中不只是简单 set，还包含：
 * <ul>
 *   <li>参数校验（URL 格式、超时范围）</li>
 *   <li>方法调用（添加 Header、添加 QueryParam）</li>
 *   <li>条件逻辑（自动追加 Content-Type、自动生成 RequestId）</li>
 *   <li>格式转换（超时单位换算、URL 拼接 QueryString）</li>
 *   <li>build() 中的最终校验与组装</li>
 * </ul>
 */
public class HttpRequest {

    private final String method;
    private final String url;
    private final List<String> headers;
    private final String body;
    private final int timeoutMs;
    private final String requestId;
    private final LocalDateTime createdAt;

    private HttpRequest(Builder builder) {
        this.method = builder.method;
        this.url = builder.buildUrl();
        this.headers = Collections.unmodifiableList(builder.headers);
        this.body = builder.body;
        this.timeoutMs = builder.timeoutMs;
        this.requestId = builder.requestId;
        this.createdAt = builder.createdAt;
    }

    @Override
    public String toString() {
        DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        return "\n=== HttpRequest ===" +
                "\n  requestId : " + requestId +
                "\n  createdAt : " + createdAt.format(fmt) +
                "\n  method    : " + method +
                "\n  url       : " + url +
                "\n  headers   : " + headers +
                "\n  body      : " + body +
                "\n  timeoutMs : " + timeoutMs + " ms";
    }

    // -----------------------------------------------------------------------
    // Builder
    // -----------------------------------------------------------------------

    public static Builder newRequest(String method, String baseUrl) {
        return new Builder(method, baseUrl);
    }

    public static final class Builder {

        private final String method;
        private final String baseUrl;
        private final List<String> queryParams = new ArrayList<>();
        private final List<String> headers = new ArrayList<>();
        private String body;
        private int timeoutMs = 5_000;
        private String requestId;
        private LocalDateTime createdAt;

        private Builder(String method, String baseUrl) {
            if (method == null || method.isBlank()) {
                throw new IllegalArgumentException("HTTP method 不能为空");
            }
            if (baseUrl == null || !baseUrl.startsWith("http")) {
                throw new IllegalArgumentException("baseUrl 必须以 http/https 开头，当前值：" + baseUrl);
            }
            this.method = method.toUpperCase();
            this.baseUrl = baseUrl;
        }

        /**
         * 添加请求头（key: value 格式）
         * Builder 内部做格式化，调用方只需传 key/value
         */
        public Builder header(String key, String value) {
            if (key == null || key.isBlank()) {
                throw new IllegalArgumentException("Header key 不能为空");
            }
            // 方法调用：格式化后加入列表
            headers.add(key.trim() + ": " + (value == null ? "" : value.trim()));
            return this;
        }

        /**
         * 添加 Query 参数（自动 URL 编码拼接）
         */
        public Builder queryParam(String key, Object value) {
            if (key != null && !key.isBlank()) {
                // 方法调用：转字符串并追加
                queryParams.add(key + "=" + value);
            }
            return this;
        }

        /**
         * 设置 JSON Body，同时自动追加 Content-Type 头
         * 条件逻辑：有 body 才加 Content-Type
         */
        public Builder jsonBody(String json) {
            this.body = json;
            // 条件逻辑：自动追加 Content-Type
            header("Content-Type", "application/json;charset=UTF-8");
            return this;
        }

        /**
         * 设置超时（秒），Builder 内部换算为毫秒
         * 格式转换：秒 → 毫秒
         */
        public Builder timeoutSeconds(int seconds) {
            if (seconds <= 0 || seconds > 300) {
                throw new IllegalArgumentException("超时时间必须在 1~300 秒之间，当前值：" + seconds);
            }
            // 单位换算
            this.timeoutMs = seconds * 1_000;
            return this;
        }

        /**
         * 手动指定 RequestId；若不调用，build() 时自动生成
         */
        public Builder requestId(String requestId) {
            this.requestId = requestId;
            return this;
        }

        /**
         * 组装最终 URL（baseUrl + QueryString）
         */
        private String buildUrl() {
            if (queryParams.isEmpty()) {
                return baseUrl;
            }
            return baseUrl + "?" + String.join("&", queryParams);
        }

        /**
         * 最终构建：自动生成 requestId、createdAt，并做整体校验
         */
        public HttpRequest build() {
            // 自动生成 requestId（若未手动指定）
            if (requestId == null || requestId.isBlank()) {
                requestId = "REQ-" + System.currentTimeMillis();
            }
            // 自动记录创建时间
            createdAt = LocalDateTime.now();

            // 整体校验：POST/PUT 必须有 body
            if (("POST".equals(method) || "PUT".equals(method)) && (body == null || body.isBlank())) {
                throw new IllegalStateException(method + " 请求必须提供 body");
            }

            return new HttpRequest(this);
        }
    }

    // -----------------------------------------------------------------------
    // 演示入口
    // -----------------------------------------------------------------------

    public static void main(String[] args) {
        // 示例 1：GET 请求，带 QueryParam 和自定义 Header
        HttpRequest getRequest = HttpRequest
                .newRequest("GET", "https://api.example.com/users")
                .queryParam("page", 1)
                .queryParam("size", 20)
                .queryParam("keyword", "shai")
                .header("Authorization", "Bearer token-abc123")
                .header("Accept", "application/json")
                .timeoutSeconds(10)
                .build();

        System.out.println(getRequest);

        // 示例 2：POST 请求，带 JSON Body（自动追加 Content-Type）
        HttpRequest postRequest = HttpRequest
                .newRequest("POST", "https://api.example.com/users")
                .header("Authorization", "Bearer token-abc123")
                .jsonBody("{\"username\":\"shai\",\"email\":\"shai@example.com\"}")
                .timeoutSeconds(30)
                .requestId("MY-REQ-001")
                .build();

        System.out.println(postRequest);
    }
}

```

