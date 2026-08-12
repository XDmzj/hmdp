
# Redisson可重入锁的原理

流程图

![[Pasted image 20260807154146.png]]



Redisson可重入锁采用了redis的hash类型，多了state这个重入计数字段，所以不再用setnx命令，只能用lua保证获取锁操作的原子性
即：
Redisson 可重入锁用 Hash 存储线程标识和重入次数。由于需要多条命令配合操作，它在内部通过 Lua 脚本实现加锁逻辑，脚本中用 `exists` 等命令来保证“不存在才创建”的互斥性，与 `SETNX` 原理一致，但不再单独使用 `SETNX` 命令


获取锁操作脚本
![[Pasted image 20260807154137.png]]


释放锁操作脚本
![[Pasted image 20260807154504.png]]


可重入锁在代码写法上和普通锁**完全一样**，因为 Redisson 的 `RLock` 本身就内置了可重入能力。你不需要额外做任何事。