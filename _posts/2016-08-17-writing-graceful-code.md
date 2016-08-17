---
layout: post
title: 书写优雅的代码
categories: [Android 实践]
description: 
keywords: Android, Java
---

# 书写优雅的代码

## Nullable

```Java
public class SomeModel {
  @Nullable
  public SomeField getSomeField();
}
```

在调用这段代码的时候，通常我们使用

```Java
if (someModel.getSomeField() != null) {
  view.setSomeField(someModel.getSomeField());
}
```

如果这样写会更好：

```java
if (someModel.hasSomeField()) {
  view.setSomeField(someModel.getSomeField());
}
```

##### 如果在 Java 8 环境下或者引入了 Google Guava ，还可以这样写：

```java
public class SomeModel {
  Optional<SomeField> getSomeField();
}

Optional<SomeField> field = someModel.getSomeField();
if (field.isPresent()) {
  view.setSomeField(field.get());
}
```

或者：

```java
public class SomeModel {
  @Nullable
  public SomeField getSomeField();
}

Optional<SomeField> field = Optional.fromNullable(someModel.getSomeField());
if (desc.isPresent()) {
  view.setSomeField(field.get());
}
```



## 时间

### TimeMillis

我们一定会遇到处理时间的时候写

```java
private Data mData;
private long mUpdateTimeMillis;

public boolean isDataValid() {
  long currentMillis = System.currentTimeMillis();
  // expire time is 1h.
  String abc = "abc";
  String c = abc + "c";
  String d = new String("abc");
  String e = d + c;
  SStringBuilder sb = new StringBuilder();
  sb.append("a").append(d).append ......;
  for ()
  return (currentMillis - mUpdateTimeMillis < 1 * 60 * 60 * 1000);
}
```

如果这样就会更有可读性

```java
public boolean isDataValid() {
  long currentMillis = System.currentTimeMillis();
  return (currentMillis - mUpdateTimeMillis < TimeUnit.HOURS.toMillis(1));
}
```

### DateFormat

这段代码你能看出来他干了什么嘛？

```java
// start : "10:24"
// durationMinutes : 105
private void setEndTime(String start, long durationMinutes) {
  if (MovieStringUtil.isEmpty(start) || start.split(":").length < 2) {
    endTv.setVisibility(GONE);
      return;
  }
  String[] times = start.split(":");
  long tempHour = (Long.parseLong(times[1]) + durationMinutes) / 60;
  long minutes = (Long.parseLong(times[1]) + durationMinutes) % 60;
  String strMinutes = minutes < 10 ? String.format("0%s",  String.valueOf(minutes)) :
                                     String.valueOf(minutes);
  long hours = (Long.parseLong(times[0]) + tempHour) % 24;
  endTv.setText(String.format("%d:%s%s", hours, strMinutes,
                              getContext().getString(R.string.movie_end_time)));
}
```

这样是不是就清楚多了：

```java
private void setEndTime(String date, String start, long durationMinutes) {
  SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm", Locale.getDefault());
  try {
    Date endDate = format.parse(date + " " + start);
    endDate.setTime(endDate.getTime() + TimeUnit.MINUTES.toMillis(durationMinutes));
    SimpleDateFormat simpleFormat = new SimpleDateFormat("HH:mm", Locale.getDefault());
    endTv.setText(getContext().getString(R.string.movie_end_time, simpleFormat.format(endDate)));
  } catch (ParseException e) {
    endTv.setVisibility(GONE);
  }
}
```

关于时间处理， [JodaTime](http://www.joda.org/joda-time/) 库给出了很好的解决方案。这个库包含的方法较多，会给 Android 应用带来一定的方法数和包体积的负担。在 Java 8 新增的 java.time 包里，有一套简化版本的实现。 Java 8 的时间 API 也是由 JodaTime 的作者主导实现的。

## 方法的参数、返回值和异常

### 对于无意义的参数应该抛出异常。

方法对于不期望的参数经常是采取鸵鸟策略，而不是积极地终止调用者。

例如：

```java
public static String getShowSeats(MovieSeatOrder bean) {
  if (bean == null || bean.getSeats() == null) {
      return "";
  }
  List<NodeSeats.Seat> seats = bean.getSeats().getList();
  if (CollectionUtils.isEmpty(seats)) {
      return "";
  }
  return ...;
}
```

这个方法的问题在于：参数值为 `null` 是没有任何意义的。**一个方法是否接受 `null` 作为参数，应该取决于 `null` 是否是有意义的。**我们应当阻止一切情况下的参数值为 `null` 。如果参数的值是 `null` ，说明我们的程序中有 bug 。

好的写法应该是：

```java
public static String getShowSeatsDescription(MovieSeatOrder bean) {
  checkNotNull(bean); // throws an NullPointerException if null.
  if (bean.isValid()) { // contains a non-empty seat list.
    return ...;
  } else {
    throw new IllegalArgumentException("Invalid order. " + bean.toString());
  }
}

public void showSeatsDescription() {
  if (bean != null && bean.isValid()) {
    String desc = getShowSeatsDescription(bean);
    mTextDesc.setText(desc);
  } else {
    mTextDesc.setText(null);
  }
}
```

而不是返回一个无意义的空字符串。

再例如：

```java
public static boolean verifyPermissions(int[] grantResults) {
  if (grantResults == null) {
    return false;
  } // throw NullPointerException instead.
  if (grantResults.length < 1) {
    return false;
  } // useless prediction.
  for (int result : grantResults) {
    if (result != PackageManager.PERMISSION_GRANTED) {
      return false;
    } // use result == PackageManager.PERMISSION_DENIED.
  }
  return true;
}

```

这个方法返回值有问题，参数为 `null` 是无意义的，如果是个空数组，应该返回 `true` 。因为循环中的逻辑是：有假则返回假。

```java
public static boolean verifyPermissions(int[] grantResults) {
  checkNotNull(grantResults);
  for (int result : grantResults) {
    if (result == PackageManager.PERMISSION_DENIED) {
      return false;
    }
  }
  return true;
}
```

有说法认为，不抛出异常时因为不能让程序崩溃。而不让程序崩溃是应该防止程序进入错误的状态，并设置错误处理，而不是将错误传递给下游。

### 避免返回 `null`

返回 `null` 会阻止链式调用。

你的用户可能会调用：

```java
initialize(someArgument).calculate(data).dispatch();
```

从上面可以看到，任何一个方法都不应该返回 `null` 。**无论何时，方法都应该避免返回 `null` ， `null` 仅用来表示“未初始化”或“不存在”。**

返回 `null` 还有另外一个问题，如果我们从网络来的数据是不可信任的，那么我们要及时对这些不可信任的数据进行处理，而不是让错误蔓延。

```java
public T getData() {
  return success ? data : null;
}
```

这段代码是一个反面教材，如果没有 data 字段，应该及时抛出异常，下面的代码再处理一个值为 `null` 的数据，不是抛出空指针异常就是进入了不正确的状态。著名的 Java 类库 Guava 被称为是 Java 类库的设计典范，其中绝大多数方法都是不接受为 `null` 的参数，也不返回 `null` 。



# 正确的依赖

Java 代码似乎要比其他语言更现代，更容易写，也更容易写出糟糕的代码。我曾经见过这样一段代码：

```java
class FooFragment extends Fragment {
  
  public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    Activity activity = getActivity();
    if (activity instanceof FooActivity) {
      return ...;
    } else if (activity instanceof BarActivity) {
      return ...;
      onFragmentDosomething(Fragment f);
    } else {
      return ...;
    }
  }
}
```

这段代码是低层级依赖高层级的典型例子。

跨层级依赖：我们在代码中应该尽量依赖较高层级的类，而不是调用最底层的方法。

```java
    /**
     * 将衍生品、小吃订单转化为请求中的字符串格式
     *
     * @return "dealList" : [{"dealId":123456,type:0,quantity:1,cardCode:3123,promotionId:12345}]
     */
    private String getDealListJson() {
        ...
        StringBuilder jsonStr = new StringBuilder("[");
        for (CinemaSnackSelected deal : snackSelectList) {
            jsonStr.append("{");
            jsonStr.append("dealId:").append(String.valueOf(deal.getDealId())).append(',');
            jsonStr.append("type:").append(String.valueOf(deal.getType())).append(',');
            jsonStr.append("quantity:").append(String.valueOf(deal.getQuantity())).append(',');
            jsonStr.append("cardCode:").append(String.valueOf(deal.getCardCode())).append(',');
            jsonStr.append("promotionId:").append(String.valueOf(deal.getPromotionId()));
            jsonStr.append("},");
        }
        jsonStr.replace(jsonStr.length() - 1, jsonStr.length(), "]");
        return jsonStr.toString();
    }
```



## 使用函数式

函数式大多数会更有表现力，比如查找满足条件的对象所在的位置：

```java
for (int i = 0; i < movies.size(); i++) {
  long id = movies.get(i).getId();
  if (id == movieId) {
    selection = i;
    break;
  }
}

selection = Iterables.indexOf(movies, movie -> movie.getId() == movieId);
```

比如寻找集合中最大的值：

```java
int max = Integer.MIN_VALUE;
for (int i = 0; i < data.size(); i++) {
  int value = data.get(i);
  max = value > max ? value : max;
}

int max = Ordering.max(data);

int max = Collections.max(data);
```

比如字符串组合：

```java
StringBuilder sb = new StringBuilder();
for (SeatInfoBean t : currentSelect) {
  sb.append(t.getSeats()).append(",");
}
return sb.deleteCharAt(sb.length() - 1).toString();

Joiner joiner = Joiner.on(",").skipNulls();
return joiner.join(Iterables.transform(currentSelect, bean -> bean.getSeats());
```

[Guava](https://github.com/google/guava) 工程包含了若干被Google的 Java项目广泛依赖 的核心库，例如：集合 [collections] 、缓存 [caching] 、原生类型支持 [primitives support] 、并发库 [concurrency libraries] 、通用注解 [common annotations] 、字符串处理 [string processing] 、I/O 等等。 所有这些工具每天都在被Google的工程师应用在产品服务中。

这个库大概有 1.5 万个方法，对包大小和方法数确实有比较大的影响。
