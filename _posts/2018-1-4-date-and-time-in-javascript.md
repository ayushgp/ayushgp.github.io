---
layout: post
title: Date and Time in JavaScript
comments: true
---

Programmers usually don't understand the complexity of Date and time until they have to dip their toes in this world. Handling Timezones, taking daylight savings in account and Date math are some of the things that make handling dates and time a difficult task.

In this article we'll talk about some of these issues that people face while using date and time in JavaScript and in general while programming.

![Clocks](https://media.giphy.com/media/l0MYOUI5XfRk4LLWM/giphy.gif)
## Before we start
There are some things that you need to understand about dates and times before we delve into the JS Date library.

1. UTC is Coordinated Universal Time (UTC), the world's time standard.
2. The local time within a time zone is defined by its offset (difference) from Coordinated Universal Time (UTC)
3. Daylight Saving Time (DST) is the practice of setting the clocks forward 1 hour from standard time during the summer months, and back again in the fall, in order to make better use of natural daylight. 

There is much more complexity than this but those are exceptions rather than the norm. You can read more about them on [TimeAndDate.com](https://www.timeanddate.com/time/time-zones.html).

## The Date object
The Date object is a datatype built into the JavaScript language. `Date` objects are based on a time value that is the number of milliseconds since 1 January 1970 UTC. There are multiple ways in which you can instantiate a `Date`. All of these methods instantiate the date in your local time zone.

```js
new Date();
// Creates a Date object using the current time and timezone
// Returns: 
// Thu Jan 04 2018 12:20:02 GMT+0530 (India Standard Time)

new Date(milliseconds);
// Creates a Date object using the no of milliseconds since epoch
// new Date(0) returns: 
// Thu Jan 01 1970 05:30:00 GMT+0530 (India Standard Time)

new Date(dateString);
// Creates a date object using a date format which is accepted by the Date.parse method.
// You can read more about the supported date string formats here:
// http://tools.ietf.org/html/rfc2822#page-14

// new Date("1970-01-01") returns: 
// Thu Jan 01 1970 05:30:00 GMT+0530 (India Standard Time)

new Date(year, month, day, hours, minutes, seconds, milliseconds);
// This representation is the easiest to comprehend as it accepts the date 
// in form of 7 arguments. All the values should be in Integer format

// Note: The month argument is from 0 to 11 and not from 1 to 12
```

> Note: If you call just `Date()` without the `new` keyword, you'll get a string instead of a `Date` object.

## Getters and Setters
The Date class gives you getters and setters for manipulating almost every aspect of a date. These getters and setters are also available in UTC version to save you the hassle of getting and setting things manually in UTC. You can check out the full list of methods on the [MDN Javascript Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date). 

Let's have a look at some of the interesting one's. 

```js
let today = new Date()
console.log(today)
// Thu Jan 04 2018 12:46:12 GMT+0530 (India Standard Time)

console.log(today.getDay())
// Prints 4 as day is between 0-6 starting with Sunday.

console.log(today.getMonths())
// 0 as months in JS start from 0-11. 
// Also months wrap around. 
// So if you set month to 13, it'll be set to 1 or February

console.log(today.getTime())
// 1515050172692
// Returns the numeric value of the specified date as the 
// number of milliseconds since January 1, 1970, 00:00:00 UTC
// (negative for prior times).
```

## Date arithematic
You sometimes need to calculate difference between dates/times, maybe for benchmarking purposes. Or you may need to calculate a certain date or time after a period of time. The Date class supports arithematic in JS and makes both of these tasks very easy. You can simply subtract subtract 2 dates and get their difference in milliseconds. For Example,

```js
// Represents 10:00:00 on 1st Jan 2018
let day1 = new Date(2018, 0, 1, 10) 

// Represents 12:30:00 on 1st Jan 2018
let day2 = new Date(2018, 0, 1, 12, 30)

let diff = day2 - day1
console.log(diff)
// 9000000 - Difference between 2 dates in milliseconds
// You can define your own functions to convert this time to minutes, seconds, etc.
```

For adding time to a given date, you first need to extract the time in milliseconds since epoch from the date object using `Date.getTime()` function, add time to it in milliseconds and finally recreate a Date object using the `new Date(milliseconds)` constructor. For example,

```js
let day1 = new Date(2018, 0, 1, 10) 

let timeInMillis = day1.getTime() + 9000000
let day2 = new Date(timeInMillis)
console.log(day2)
// Mon Jan 01 2018 12:30:00 GMT+0530 (India Standard Time)
```

## Timezones
![Timezone Map](http://www.developingthefuture.net/wp-content/uploads/2013/07/localization-timezones.png)
Whenever Date is called as a constructor with more than one argument, the specifed arguments represent local time. You can convert this time to a string in any timezone using the `toLocaleString()` method and providing it the locale and timezone as arguments. For example,

```js
let today = new Date()

console.log(today.toLocaleString('en-US', {timeZone: 'America/Los_Angeles'}));
console.log(today.toLocaleString('en-US', {timeZone: "Asia/Shanghai"}));
console.log(today.toLocaleString('en-US', {timeZone: "Asia/Kolkata"}));

// 1/3/2018, 11:56:43 PM
// 1/4/2018, 3:56:43 PM
// 1/4/2018, 1:26:43 PM
```

If you don't provide the timeZone propery in the options, the format assumes that you want the time in local time of the locale you mention. For example,

```js
let today = new Date()

// US English uses month-day-year order and 12-hour time with AM/PM
console.log(today.toLocaleString('en-US'));
// 1/4/2018, 1:26:43 PM
// British English uses day-month-year order and 24-hour time without AM/PM
console.log(today.toLocaleString('en-GB'));
// 04/01/2018, 13:26:43

// Korean uses year-month-day order and 12-hour time with AM/PM
console.log(today.toLocaleString('ko-KR'));
// 2018. 1. 4. 오후 1:26:43
```

A date object represents a particular time. You just need the milliseconds from epoch. Using the `toLocaleString` function and `timeZone` options you can easily convert this date to a string in any locale and timezone. The timezone you provide here also takes into account the DayLight Savings(If any).

The toLocaleString function offers a plethora of options that are out of scope for this rather short article. You can read more about the toLocaleString function on the [MDN Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toLocaleString).

## Few words of Caution
There are a lot of gotchas you need to keep in mind when working with date and time in JavaScript. I've mentioned some of them in the article, but for ease of access, I've listed down a few of them here:

1. If you call just Date() without the new keyword, you’ll get a string instead of a Date object.
2. Months in JS start from 0-11. Also months wrap around. So if you set month to 13, it'll be set to 1 or February.
3. Day(or weekday) is between 0-6 starting with Sunday.
4. Where Date is called as a constructor with more than one argument, the specifed arguments represent local time. When called with one argument, it simply uses that as count of milliseconds from epoch(UTC or local doesn't matter here).
5. Parsing of date strings with the Date constructor and Date.parse is strongly discouraged due to browser differences and inconsistencies.

Thanks to [Wanderdüne](https://disqus.com/by/wanderdne/) for pointing out that the implementation for the daylight saving time in JavaScript is broken. You can read more about it in his comment or on [The Annotated ES5 spec](http://es5.github.io/#x15.9.1.8).

## Conclusion
We discussed how we can use Date in Vanilla Javascript to represent dates and time. We took a look at Date arithematic in JS. We also saw how to convert date and time between different timezones. 

Though you can use the Date directly in JS, do check out the [momentjs library](http://momentjs.com/). This library offers many features like date formatting, relative time, calender time, etc. that are just too much work to implement yourself. 

[Tweet this!](https://twitter.com/intent/tweet?text="Date%20and%20Time%20in%20JS"&url="https://ayushgp.github.io/date-and-time-in-javascript"&via=ayushgp)

View the [discussion on reddit](https://www.reddit.com/r/javascript/comments/7o1t0r/date_and_time_in_javascript/).
