<h1>Using "Properties" In DRF</h1>

We often hear questions like this in our She Codes classes during the Django Rest Framework module:

> **❓ QUESTION ❓**\
> I have (for example) a `Project` model and a `Pledge` model. Each `Pledge` is associated with a `Project`, and specifies an amount of money that has been pledged to that project. 
>
> I want a page on the front end that displays the details of a given project, including a summed total of all the money pledged to that project. How/where should I calculate that summed total?

Good news: there are a lot of ways to do this! You're at the point in your coding journey now where your tools can accomplish almost anything, so solving problems like this are a bit of a "choose your own adventure" story.

To prove this point, I'll mention a couple of "quick-fix" solutions and then I'll demonstrate a slightly more "elegant" solution. But these aren't the only options!

<h2>Table of Contents</h2>

- [Quick Fix #1: Offload this work to the front end.](#quick-fix-1-offload-this-work-to-the-front-end)
- [Quick Fix #2: Write some custom code to solve the problem.](#quick-fix-2-write-some-custom-code-to-solve-the-problem)
- [A Slightly More Elegant Solution](#a-slightly-more-elegant-solution)
  - [Decorators](#decorators)
  - ["Classes" Revision:](#classes-revision)
    - [Attributes](#attributes)
    - [Methods](#methods)
  - [Properties](#properties)
  - [Putting It All Together](#putting-it-all-together)
  - [What Next?](#what-next)

## Quick Fix #1: Offload this work to the front end.
Javascript is a powerful programming language. If you have an endpoint that will give you a list of pledges (or ideally, a list of just the pledges made to one specific project) you could easily use some basic mathematics to perform the addition, and then display the result somewhere on your page. 

This is potentially slow, though, and isn't as elegant as other potential solutions. It also just delays the work until the React unit, where you'll likely have other things to worry about.

## Quick Fix #2: Write some custom code to solve the problem.
One way to do accomplish what you need might be to create a view that: 
1. Accepts a project ID and grabs that project from the database,
2. Interrogates it to get a list of its linked pledges,
3. Adds up the contributions from all those pledges, and finally,
4. Returns JSON something like this:
    ```JSON
    {
        "project_id": 1,
        "total_amount_pledged": 9001
    }
    ```
This works fine, but it actually isn't such a quick fix. It also involves writing a fair bit of single-use code. One can imagine that down the track you might decide you want another view to display all the money that a given user has donated on the platform. Your endopints are starting to get a bit bloated now... It might be nicer if we could find a solution that didn't require any extra views...

## A Slightly More Elegant Solution

To apply this solution, we need to learn about a new Python tool - **decorators**.

### Decorators
A decorator is basically just a generic way of modifying any function, that changes how the function works. They let you add pre-built features to your functions without having to write a lot of extra code. 

We use the `@` symbol to apply a decorator to a function, like so:

```py
@some_decorator
def my_function():
    print("This function has some extra functionality added to it courtesy of the '@some_decorator' property!")
```

There are a bunch of decorators built into Python, and it's even possible to write your own. The one I want to tell you about today is the `@property` decorator. To show you how it works, we need to do some quick revision...

---

### "Classes" Revision:
Remember that Python classes have two main types of "features" - **attributes**, and **methods**.

#### Attributes
An attribute is like a **variable** that is attached to the object. It stores a value, and we access that value with "dot-notation" like so:

```py
some_class_instance.my_attribute = "This is an attribute!"

print(some_class_instance.my_attribute)
```

#### Methods
A method is a **function** that is attached to the object. Methods are also accessed with dot-notation:

```py
the_value = some_class_instance.my_method()

print(f"The value returned by the method was {the_value}")
```

So what is a property...?

---

### Properties
A **property** is a bit like "a method that can disguise itself as an attribute". It is a function that doesn't require brackets `()` when it is called. 

> [!NOTE]  
> This isn't the only thing properties can do. Have a read on them later if you're interested: https://docs.python.org/3/library/functions.html#property

We can create a property by simply adding the `@property` decorator to any method. Let's look at an example. Here we've defined a property that loudly alerts you when it gets accessed:

```py
# Define a class as usual
class MyClass():

    # Define a method, and decorate it with the @property decorator!
    @property
    def my_property():
        print("The property has been accessed!")
        return 9001

# create a class instance as usual
some_instance = MyClass()

# call the property! no need for brackets here, even though it's technically a function...
property_value = some_instance.my_property

# print the result
print(f"The value returned by the property was {property_value}.")
```

The program's output looks like this:
```txt
The property has been accessed!
The value returned by the property was 9001.
```

So why is this important? 

Well, what we'd really like is to just include the sum of all linked pledges in the serialized output of our `ProjectDetailSerializer`. The only problem is, our serializers get their data from our models' fields, and that "sum of pledges" number that we're looking for isn't stored as a field in the database. That means we can't serialize it. To find its value we would need to implement some kind of method to calculate it...

**Brainwave**: all of our models are classes!  Maybe if we use a property instead of a method to calculte the sum, we can trick the serializer into thinking that it is a field attribute and sneak the result into the JSON output!

### Putting It All Together
Here's what that solution might look like. To keep things interesting, we'll use an unrelated example. Applying this pattern to your models for the Crowdfunding project is left as an exercise to the reader:

In this example we have **teachers** and **subjects**. Each `subject` has a `number_of_students` field, and is attached to a `teacher` instance. We want to include a value for `total_number_of_students` in our serialised data at the `/teacher_details/<int:pk>/` endpoint.

First, let's add the property to calculate the value:

```diff
# models.py
from django.db import models

class Teacher(models.Model):
	name = models.CharField(max_length=200)

+   @property
+   def total_number_of_students(self):
+       result = 0
+       for each_subject in self.subjects:
+           result += each_subject.number_of_students
+       return result

class Subject(models.Model):
	name = models.IntegerField()
	number_of_students = models.IntegerField()
    teacher = models.ForeignKey(
        'Teacher',
        on_delete=models.CASCADE,
        related_name='subjects'
	)
```

> [!NOTE]  
> This isn't the most efficient way to perform the sum calculation, but let's keep things simple here! You are welcome to use something neater if you'd like to. :) 

Now let's make sure our serializer picks up the value. We just have to point it out...

```diff
# serializers.py

# ... skipping over some code here...

class TeacherSerializer(serializers.ModelSerializer):    
    class Meta:
        model = apps.get_model('projects.Teacher')
        fields = '__all__'

class TeacherDetailSerializer(TeacherSerializer):
+   total_number_of_students = serializers.ReadOnlyField()
    subjects = SubjectSerializer(many=True, read_only=True)
```

> [!NOTE]  
> We use a `ReadOnlyField` here because it shouldn't be possible to update a `Teacher` instance to modify the number of students they have! The only way that number should change is if more students enrol in a `Subject` that that teacher teaches!

### What Next?
**NOTHING**. That's literally it! The serializer should dutifully include the calculated value in its output, and that value should therefore appear in the JSON returned by the "detail" endpoint. No muss, no fuss. 

This is a nifty solution, with one caveat:

> [!CAUTION]
> This solution works well for a "details" endpoint where you'll be accessing the details of one record at a time. If you're going to be displaying the details of hundreds or more records all at once, you risk encountering a bottleneck where you hammer the database over and over, grabbing the subjects associated with each teacher in turn. In that situation, you'd need to do a little bit of optimising in your view code. But let's worry about that problem when we come to it!
