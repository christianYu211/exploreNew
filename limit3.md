在半导体芯片测试（ATE）场景中，设备由于网络抖动或重试机制，可能会发送完全重复的测试数据。针对这种情况，我们需要在限流逻辑中引入**“幂等性校验（去重）”**功能。
通过在 Redis 中存储消息内容的 指纹（Hash） 并设置短暂的有效期，可以有效拦截重复消息。
1. 核心实现方案
我们将“内容去重”作为限流的一个前置可选步骤。
 * 指纹提取：将消息体（Object）转换为 JSON 或 字节数组，计算其 MD5 或 SHA-256 值。
 * 原子判定：利用 Redis 的 SET key value NX EX seconds 命令。
   * NX：只有当 Key 不存在时才插入成功。
   * EX：设置过期时间（如 5 秒），防止 Redis 内存被无限堆积。
 * 流程顺序：去重校验 -> 限流校验 -> 业务执行。这样可以确保重复的消息不会消耗合法的限流令牌。
2. 升级自定义注解 @RateLimit
增加去重相关的配置参数。
public @interface RateLimit {
    // ... 原有参数保持不变 ...

    /** 是否开启内容去重 */
    boolean doDeduplicate() default false;

    /** 重复内容判定窗口（秒），在此时间内相同内容将被拦截 */
    int deduplicateTTL() default 5;

    /** 指定哪个参数作为去重的核心内容，支持 SpEL，如 "#msg" */
    String contentKey() default "";
}

3. 升级 AOP 切面逻辑
在执行限流 Lua 脚本之前，先进行内容去重判定。
@Around("@annotation(rateLimit)")
public Object around(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
    MethodSignature signature = (MethodSignature) joinPoint.getSignature();
    Object[] args = joinPoint.getArgs();

    // --- 第一步：内容去重校验 (Optional) ---
    if (rateLimit.doDeduplicate()) {
        // 1. 获取需要去重的内容对象
        Object content = SpelUtil.parseObject(rateLimit.contentKey(), signature.getMethod(), args);
        if (content != null) {
            // 2. 生成内容指纹 (MD5)
            String contentHash = DigestUtils.md5DigestAsHex(JsonUtils.toJsonBytes(content));
            String dedupKey = "dedup:" + signature.getMethod().getName() + ":" + contentHash;

            // 3. 尝试写入 Redis (SETNX)
            Boolean isNew = redisTemplate.opsForValue().setIfAbsent(dedupKey, "1", 
                    Duration.ofSeconds(rateLimit.deduplicateTTL()));
            
            if (Boolean.FALSE.equals(isNew)) {
                log.info("检测到重复消息内容，已拦截. Key: {}", dedupKey);
                return null; // 或者抛出异常，取决于业务是否需要给设备返回错误
            }
        }
    }

    // --- 第二步：原有限流逻辑 (Sliding Window + Token Bucket) ---
    // ... 执行 Redis Lua 脚本逻辑 ...
    
    return joinPoint.proceed();
}

4. 为什么要在限流前做去重？
在 ATE 生产环境中，这种顺序至关重要：
| 顺序 | 优点 | 缺点 |
|---|---|---|
| 先去重，后限流 | 节省令牌。重复的无效数据不会消耗机台的限流额度，保证了“正常数据”的通过率。 | 会增加一次 Redis 网络开销（可以通过 Lua 合并）。 |
| 先限流，后去重 | 优先保护 Redis 内存，防止大量重复数据撑爆 ZSet。 | 重复数据会“浪费”掉合法的令牌，导致真实数据被限流。 |
结论： 针对芯片测试这种对数据完整性敏感的场景，推荐使用 “先去重，后限流”。
5. 性能优化：将去重集成进 Lua 脚本
如果对延迟极其敏感，我们可以修改 Lua 脚本，将 SET NX 逻辑集成进去，实现一次 Redis 交互完成两项校验。
修改后的 Lua 逻辑片段：
-- 新增参数 ARGV[6]: 内容指纹Key, ARGV[7]: 去重TTL
if ARGV[6] ~= "" then
    local is_duplicate = redis.call('SET', ARGV[6], '1', 'NX', 'EX', ARGV[7])
    if not is_duplicate then
        return -2 -- 特殊状态码：表示内容重复
    end
end

-- ... 接着执行原来的滑动窗口和令牌桶逻辑 ...

6. ATE 场景中的注意事项
 * 内容提取（Content Key）：测试消息通常包含 timestamp（时间戳）。如果时间戳精确到毫秒级，那么“内容”永远不会完全相同。
   * 建议：在 SpEL 提取内容时，只针对业务核心字段（如 deviceId + testResult + stepId），排除掉时间戳字段。
 * 内存占用：如果 ATE 数据包非常大（如几百KB），不要直接把整个包存入 Redis。计算 MD5 后，Redis 里的 Key 只有 32 位字符，非常节省空间。
 * 处理重复的策略：
   * 如果设备是“重试”，服务端拦截后可以返回“成功”，让设备误以为处理完了，从而停止重试。
   * 如果直接抛出异常，设备可能会持续不断地重发，反而加重系统负担。
您是否需要我为您提供一个能够排除掉“动态时间戳”字段、仅对“业务核心字段”生成 MD5 的工具类实现？
