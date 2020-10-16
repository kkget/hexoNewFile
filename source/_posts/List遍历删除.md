---
layout: w
title: List遍历删除
date: 2020-07-02 10:56:32
tags: List
---
方法一：
```java
 public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("a");
        list.add("ab");
        list.add("abc");
        list.add("ad");
        Iterator<String> iterator = list.iterator();
        while(iterator.hasNext()){
            String next = iterator.next();
            if("a".equals(next)||"c".equals(next)){
                iterator.remove();
            }
        }
        System.out.println(JSONObject.toJSONString(list));
    }
```
输出：
```
["ab","abc","ad"]
```
方法二
```java
public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("a");
        list.add("ab");
        list.add("abc");
        list.add("ad");
      	list.removeIf(s->"a".equals(s)||"c".equals(s));
        System.out.println(JSONObject.toJSONString(list));
    }
```
异常操作
```java
 		   
        List<String> list = Arrays.asList("a", "b", "abcd");
        list.removeIf(s -> "a".equals(s) || "c".equals(s));
        System.out.println(JSONObject.toJSONString(list));
        Exception in thread "main" java.lang.UnsupportedOperationException
                原因
        @SafeVarargs
        @SuppressWarnings("varargs")
        public static <T > List < T > asList(T...a){
            return new ArrayList<>(a);
        }

        /**
         * @serial include
         */
        private static class ArrayList<E> extends AbstractList<E>
                implements RandomAccess, java.io.Serializable {

        }

        AbstractList做任何操作都是异常
        public void add ( int index, E element){
            throw new UnsupportedOperationException();
        }

        /**
         * {@inheritDoc}
         *
         * <p>This implementation always throws an
         * {@code UnsupportedOperationException}.
         *
         * @throws UnsupportedOperationException {@inheritDoc}
         * @throws IndexOutOfBoundsException     {@inheritDoc}
         */
        public E remove ( int index){
            throw new UnsupportedOperationException();
        }
```

修改
```java
		String [] str={"a","b","c","abcd"};
        List<String> strs = Arrays.asList(str);
        List<String> list = new ArrayList<>();
        list.addAll(strs);
        list.removeIf(s->"a".equals(s)||"c".equals(s));
        System.out.println(JSONObject.toJSONString(list));
```
