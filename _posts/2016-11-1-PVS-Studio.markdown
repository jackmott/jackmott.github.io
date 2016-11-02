---
layout: post
title:  "PVS-Studio C#"
date:   2016-11-1 14:17:27 -0500
categories: programming
---
[PVS-Studio](http://www.viva64.com/en/pvs-studio/) is a popular static analysis tool in the C++ world, and plenty of articles have been written about the kinds of
bugs it can find in C++ projects, such as this entertaining one about the [Unreal Engine](https://www.unrealengine.com/blog/how-pvs-studio-team-improved-unreal-engines-code).
About year ago they added C# support, and have steadily been adding more C# analysis features since.  Today I grabbed version 6.10 and ran it on my code base at work, which
is a fairly large ASP/MVC web application.  Here are some of the things it found:

#### The 'user.Group' object was used before it was verified against null
```c#
  var user = _entityRepository.GetOnlineUserByUsername(username);
  string nsId = user.Group.NetSuiteInternalId;
```
One of the nicer features of PVS-Studio is that it can identify cases like this, where a null pointer exception is possible.
By identifying these and dealing with them you can eliminate an common class of error and deal with it more gracefully.

#### Expression 'result.Succes' is always true
```c#

if(!result.Success)
  return Json(result);

//...

if (result.Success)
{    
    //...    

}           
```
This and a couple other similar examples were identified. This error could imply a serious logic error in the code. At the very least it
identifies unecessary checks cluttering up the code and slowing down execution.


#### It is odd that the body of 'CanReadType' function is fully equivalent to the body of 'CanWriteType' function
```c#
public override bool CanReadType(Type type)
{
    return SupportedType(type);
}

public override bool CanWriteType(Type type)
{
    return SupportedType(type);
}
```
This turned out to be correct for us, since we needed to override both methods.  This class of message can sometimes identify code
that can be cominbed to shrink your code base down, or may identify a copy-paste error.

#### An odd precise comparison: transaction.TaxRate == 0. Consider using a comparison with a defined precision: Math.Abs(A-B) < Epsilon
```c#
 if(transaction.TaxRate == 0)
```
Any instances of comparing floating point values to exact values will be indentified as a low-risk problem.

#### A part of conditional expression is always true if it is evaluated: billingAddressSameAsShipping != "on"
```c#
if (string.IsNullOrEmpty(CFM.BillingAddress.Id) && 
    string.IsNullOrEmpty(billingAddressSameAsShipping) && 
    billingAddressSameAsShipping != "on")
```
Another common mistake, this can often arise when an if statement is modified with an extra check later.  In this case, the check for != "on" is now unecessary. But
this could expose logic mistakes as well.


#### The 'url' variable is assigned to itself
```c#
url = url = "user/" + userId + "/attorney/" + id;
```
A copy/paste error, that likely gets compiled away, but is cluttering up the code still.

#### IDisposable object 'serverError' is not disposed before method returns
```c#
var serverError = new HttpResponseMessage(HttpStatusCode.InternalServerError);
return Request.CreateResponse(HttpStatusCode.InternalServerError);
```
PVS will identify a few different problems with the IDisposable interface, including this, classes that implement the Dispose method but not the IDisposable interface,
and classes which have IDisposable memebers but don't implement IDisposable themselves. Proper handling of these issues can reduce memory use and GC pressure.

#### The 'DateTime' constructor could receive the '0' value while positive value is expected. Inspect the first argument
```c#
DateTime now = DateTime.Now;
if (year < 0 || year > now.Year || month <= 0 || month > 12)
{
    throw new Exception("The input query time is not valid");
}

DateTime StartOfMonth = new DateTime(year, month, 1);
```
PVS Studio has identified that our check for "year < 0" is not sufficient to gaurantee correct input to the DateTime constructor.  It should be year <= 0.

#### The 'DateNeeded' variable is assigned values twice successively. Perhaps this is a mistake
```c#
if (detail)
{
    while ((date.ToString("ddd") != day))
    {
        date = date.AddDays(1);
    }

    //if the date is a holiday, add 1 
    if (santa.CheckDate(DateNeeded, type))
        DateNeeded = date.AddDays(1);
}

//fixes time 
date = date.AddHours(time - date.Hour);
date = date.AddMinutes(-date.Minute);

DateNeeded = date;
```
PVS Studio has identified that the assignment of DateNeeded = date.AddDays(1) is nonsensical, because it is immediately overritten before being used.

#### The return value of function 'Insert' is required to be utilized. 
```c#
userString.Insert(0, suffixStr);
   
```
This is a very common mistake. C# strings are immutable, and so the pattern for functions like insert, substring, etc is to return the new string.
You can easily forget this, and the operation you were trying to do on the string just doesn't happen at all.


## Quick Review

This doesn't represent all of the C# capabilities that PVS-Studio has, these are just the issues it found in our project. It integrates nicely with Visual Studio, but you
can use it standalone as well.  They have a nice evaluation version that let's you try it out for quite a long time.  I find the errors it finds to be much more relevant
than Resharper's analyses, which are numerous but seem to mostly be stylistic. 
