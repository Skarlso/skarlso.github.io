+++
author = "hannibal"
categories = ["java", "knowledge", "problem solving", "programming"]
date = "2012-06-26"
tags = ["coding kata"]
title = "Solution to Wrap Kata"
url = "/2012/06/26/solution-to-wrap-kata/"

+++

My solution to the String Wrap Kata. The goal is to have it wrap a text on a given column width.

It is not the best solution but this is my first try. I did it with TDD so there were tests first, which I'm not going to copy in..

~~~java

public class WrapKata {

    public String wrap(String input, int columnSize) {

        if (input.length() <= columnSize)
            return input;
        else {
            return wrapLines(input, columnSize);
        }
    }

    private String wrapLines(String input, int columnSize) {
        int breakPoint = getBreakPoint(input, columnSize);

        String head = createHead(input, breakPoint);
        String tail = createTail(input, breakPoint);

        return head += "\n" + wrap(tail, columnSize);
    }

    private String createTail(String input, int breakPoint) {
        return input.substring(breakPoint).trim();
    }

    private String createHead(String input, int breakPoint) {
        return input.substring(, breakPoint).trim();
    }

    private int getBreakPoint(String input, int columnSize) {
        if (input.contains(" ")) {
            return input.lastIndexOf(' ', columnSize);
        } else {
            return columnSize;
        }
    }
}
~~~