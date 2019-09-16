# 基本结构

Job
JobDetail

Trigger

Calendar

Scheduler

# JobBuilder
这个JobDetail的建造者并不复杂, 只需要重点是, 它持有一个 JobDataMap 对象, 
每个由同一个Builder创建的 JobDetail 实例都会有持有同一个 JobDataMap 对象的引用.
