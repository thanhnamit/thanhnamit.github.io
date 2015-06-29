---
layout: post
title: "Kata #2 - Binary search"
date: 2014-05-18 11:29:02 +1000
comments: true
author: Nam Nguyen
categories: [Kata, Java, Algorithm, Basic Programming]
---

In this post, I will address Kata #2

<!-- more -->

## Problem

The requirement for this Kata is described at [this link](http://codekata.com/kata/kata02-karate-chop/ "kata02-karate-chop")

## Solution

I wrote the first solution for this kata in Java - using recursion, next time I will provide other way to address this kata

#### Recursive method

``` java Recursive binary search
package kata02;

public class BinarySearcher {
	public int chop(int needle, int [] haystack) {
		if(haystack == null) return -1;
		int begin = 0;
		int end = haystack.length - 1;
		return this.binarychop(needle, haystack, begin, end);
	}

	private int binarychop(int needle, int[] haystack, int begin, int end) {
		if(end < begin) return -1;

		int middleIndex = this.findMiddlePoint(haystack, begin, end);

		if(needle == haystack[middleIndex])
			return middleIndex;
		else if(needle > haystack[middleIndex])
			return this.binarychop(needle, haystack, middleIndex + 1, end);
		else
			return this.binarychop(needle, haystack, begin, middleIndex - 1);
	}

	public int findMiddlePoint(int[] arr, int start, int end) {
		if((end - start) % 2 == 0)
			return (end-start)/2 + start;
		else
			return (end-start)/2 + 1 + start;
	}

}
```

#### Iterative method

```java Iterative binary search
private int chopIterative(int needle, int[] haystack, int begin, int end) {
		int middleIndex = -1;
		for(int i = 0; i < haystack.length; i++) {
			middleIndex = this.findMiddlePoint(begin, end);
			if(needle == haystack[middleIndex]) {
				return middleIndex;
			}
			else if(needle > haystack[middleIndex]) {
				begin = middleIndex + 1;
			}
			else {
				end = middleIndex - 1;
			}
			if(end < begin) return -1; // not found
		}
		return -1;
	}
```

Above script works, although not so nice, it will look more concise and clearer by changing for loop to while loop

```java Refactoring iterative method
private int chopIterative(int needle, int[] haystack, int begin, int end) {
		int middleIndex = -1;
		while(begin <= end) {
			middleIndex = this.findMiddlePoint(begin, end);
			if(needle == haystack[middleIndex]) {
				return middleIndex;
			}
			else if(needle > haystack[middleIndex]) {
				begin = middleIndex + 1;
			}
			else {
				end = middleIndex - 1;
			}
		}
		return -1;
	}
```

#### A lazy way

```java Lazy binary search
private int chopEasy(int needle, int[] haystack, int begin, int end) {
		return java.util.Arrays.binarySearch(haystack, needle);
}
```

#### Unit test

``` java Test basic functions
package kata02;

import static org.junit.Assert.*;

import org.junit.Test;

public class Kata02BinaryTest {

	@Test
	public void test_BinarySearch_CreateObject() {
		BinarySearcher searcher = new BinarySearcher();
	}
	@Test
	public void test_BinarySeach_NullArray() {
		int needle = 10;
		int [] haystack = null;
		BinarySearcher seacher = new BinarySearcher();
		assertEquals(-1, seacher.chop(needle, haystack));
	}

	@Test
	public void test_BinarySearch_SameElement() {
		int [] haystack = {2,3,5,5,8,10,12,14,20};
		BinarySearcher seacher = new BinarySearcher();
		assertEquals(2, seacher.chop(5, haystack));
	}

	@Test
	public void test_BinarySearch_SingleTest() {
		int [] haystack = {2,3,5,8,10,12,14,20};
		BinarySearcher seacher = new BinarySearcher();
		assertEquals(7, seacher.chop(20, haystack));
		assertEquals(6, seacher.chop(14, haystack));
		assertEquals(0, seacher.chop(2, haystack));
		assertEquals(4, seacher.chop(10, haystack));
		assertEquals(2, seacher.chop(5, haystack));
		assertEquals(-1, seacher.chop(100, haystack));
		assertEquals(-1, seacher.chop(6, haystack));
	}

	@Test
	public void test_FindMiddlePoint() {
		int [] arr = {2,3,5,8,10,12,14,20};
		BinarySearcher seacher = new BinarySearcher();
		assertEquals(4, seacher.findMiddlePoint(0, 7));
		assertEquals(3, seacher.findMiddlePoint(0, 6));
		assertEquals(3, seacher.findMiddlePoint(0, 5));
		assertEquals(6, seacher.findMiddlePoint(5, 7));
		assertEquals(4, seacher.findMiddlePoint(2, 6));
		assertEquals(1, seacher.findMiddlePoint(0, 1));
		assertEquals(7, seacher.findMiddlePoint(6, 7));
		assertEquals(4, seacher.findMiddlePoint(3, 4));
	}
}
```


#### A note about binary search

> In theory, binary search is faster than linear search, however in practice, when searching for an element in a medium sized array (about 100) elements,
> binary search nearly makes no differences. Only when you search for a large sorted array, binary search will perform better, ~ O(logn)