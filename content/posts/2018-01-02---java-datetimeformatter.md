---
title: '[Java] DatetimeFormatter에서 년도 사용 시 주의점'
date: 2018-01-02 01:10:33
template: "post"
draft: false
slug: "/posts/java-datetimeformatter"
category: "Java"
tags:
- java
- LocalDateTime
---
코드에 포맷터가 YYYY로 되어있는 부분이 있었는데, Y는 "**week-based-year**"라고 문서에 나와있다. 
보니까 그 주의 끝이 2018년이면 2018로 표기하는 것 같다. 2015년 12월 26일(토)로 했을 때에는 2015년으로 나왔는데, 2015년 12월 27일(일)로 했을 때에는 2016으로 나왔다.
그래서 원래 2017-12-31로 나오게 했어야 했는데, 2018-12-31로 나와서 약간 문제가 발생했다.
제대로 하려면 yyyy을 써야한다-_-
```
LocalDateTime localDateTime = LocalDateTime.of(2017, 12, 31, 0, 0, 0);
String strLocalDateTime = localDateTime.format(DateTimeFormatter.ofPattern("YYYY-MM-dd"));
System.out.println(String.format("date = %s", strLocalDateTime));

date = 2018-12-31
```

```
LocalDateTime localDateTime = LocalDateTime.of(2017, 12, 31, 0, 0, 0);
String strLocalDateTime = localDateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
System.out.println(String.format("date = %s", strLocalDateTime));

date = 2017-12-31
```

