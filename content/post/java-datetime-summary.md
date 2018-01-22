---
title: "Java时间问题小结"
date: 2018-01-20T21:29:36+08:00
---

1. jvm启动时会根据机器时区设定jvm的默认时区，启动后再次修改机器时区也不会影响jvm时区；
2. `System.currentTimeMillis()`返回的是自January 1, 1970, 00:00:00 GMT的毫秒数，也是UTC时间，该数据不受任何时区影响。
3. SimpleDateFormat以及Date.toString都默认使用jvm的默认时区，除非修改了它们的timezone参数。Date是无时区的。
4. SimepleDateFormat格式化日期是计算公式:utc time + (specified time zone - default timezone)。举例：

假设有一个Date对象是：2014.10.21 10:00:00， 
jvm默认时区是UTC时，SimpleDateFormat设置的时区是GMT+8:00, 那么格式化后的时间是2014.10.21 18:00:00

jvm默认时区是GMT+8:00, SimpleDateFormat设置的时区是GMT+8:00, 那么格式化后的时间是2014.10.21 10:00:00

jvm默认时区是GMT+6:00, SimpleDateFormat设置的时区是GMT+8:00, 那么格式化后的时间是2014.10.21 12:00:00
