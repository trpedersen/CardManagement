# Making Software
## Problems we face
While developing software we face a lot of difficulties along the way: unclear requirements, miscommunication, poor development process and so on.
We also face some technical difficulties: legacy code slows us down, scaling is tricky, some bad decisions of the past kick us in the teeth today.
All of them can be if not eliminated then significantly reduced, but there's one fundamental problem you can do nothing about: the complexity of your system.
The idea of a system you are developing itself is always complex, whether you understand it or not.
Even when you're making _yet another CRUD application_, there're always some edge cases, some tricky things, and from time to time someone asks "Hey, what's gonna happen if I do this and this under these circumstances?" and you say "Hm, that's a very good question.".
Those tricky cases, shady logic, validation and access managing -- all that adds up to your big idea.
Quite often that idea is so big that it doesn't fit in one head, and that fact alone brings problems like miscommunication.
But let's be generous and assume that this team of domain experts and business analysts communicates clearly and produces fine consistent requirements.
Now we have to implement them, to express that complex idea in our code. Now that code is another system, way more complicated than original idea we had in mind(s).
How so? It faces reality: technical limitations force you to deal with highload, data consistency and availability on top of implementing actual business logic.
As you can see the task is pretty challenging, and now we need proper tools to deal with it.
A programming language is just another tool, and like with every other tool, it's not just about the quality of it, it's probably even more about the tool fitting the job. You might have the best screwdriver there is, but if you need to put some nails into wood, a crappy hammer would be better, right?

## Technical aspects

Most popular languages today a object oriented. When someone makes an introduction to OOP they usually use examples:
Consider a car, which is an object from the real world. It has various properties like brand, weight, color, max speed, current speed and so on.
To reflect this object in our program we gather those properties in one class. Properties can be permanent or mutable, which together form both current state of this object and some boundaries in which it may vary. However combining those properties isn't enough, since we have to check that current state makes sense, e.g. current speed doesn't exceed max speed. To make sure of that we attach some logic to this class, mark properties as private to prevent anyone from creating illegal state.
As you can see objects are about their internal state and life cycle.
So those three pillars of OOP make perfect sense in this context: we use inheritance to reuse certain state manipulations, encapsulation for state protection and polymorphism for treating similar objects the same way. Mutability as a default also makes sense, since in this context immutable object can't have a life cycle and has always one state, which isn't the most common case.

Thing is when you look at a typical web application of these days, it doesn't deal with objects. Almost everything in our code has either eternal lifetime or no proper lifetime at all. Two most common kinds of "objects" are some sort of services like `UserService`, `EmployeeRepository` or some models/entities/DTOs or whatever you call them. Services have no logical state inside them, they die and born again exactly the same, we just recreate the dependency graph with a new database connection.
Entities and models don't have any behavior attached to them, they are merely bundles of data, their mutability doesn't help but quite the opposite.
Therefore key features of OOP aren't really useful for developing this kind of applications.

What happens in a typical web app is data flowing: validation, transformation, evaluation and so on. And there's a paradigm that fits perfectly for that kind of job: functional programming. And there's a proof for that: all the modern features in popular languages today come from there: `async/await`, lambdas and delegates, reactive programming, discriminated unions (enums in swift or rust, not to be confused with enums in java or .net), tuples - all that is from FP.
However those are just crumbles, it's very nice to have them, but there's more, way more.

Before I go any deeper, there's a point to be made. Switching to a new language, especially a new paradigm, is an investment for developers and therefore for business. Doing foolish investments won't give you anything but troubles, but reasonable investments may be the very thing that'll keep you afloat.

## Tools we have and what they give us

A lot of us prefer languages with static typing. The reason for that is simple: compiler takes care of tedious checks like passing proper parameters to functions, constructing our entities correctly and so on. These checks come for free. Now, as for the stuff that compiler can't check, we have a choice: hope for the best or make some tests. Writing tests means money, and you don't pay just once per test, you have to maintain them. Besides, people get sloppy, so every once in a while we get false positive and false negative results. The more tests you have to write the lower is the average quality of those tests. There's another problem: in order to test something, you have to know and remember that that thing should be tested, but the bigger your system is the easier it is to miss something.

However compiler is only as good as the type system of the language. If it doesn't allow you to express something in static ways, you have to do that in runtime. Which means tests, yes. It's not only about type system though, syntax and small sugar features are very important too, because at the end of the day we want to write as little code as possible, so if some approach requires you to write ten times more lines, well, no one is gonna use it. That's why it's important that language you choose has the fitting set of features and tricks - well, right focus overall. If it doesn't - instead of using its features to fight original challenges like complexity of your system and changing requirements, you gonna be fighting the language as well. And it all comes down to money, since you pay developers for their time. The more problem they have to solve, the more time they gonna need and the more developers you are gonna need.

Finally we are about to see some code to prove all that. I'm happen to be a .NET developer, so code samples are gonna be in C# and F#, but the general picture would look more or less the same in other popular OOP and FP languages.

## Let the coding begin

We are gonna build a web application for managing credit cards.
Basic requirements:
- Create/Read users
- Create/Read credit cards
- Activate/Deactivate credit cards
- Set daily limit for cards
- Top up balance
- Process payments (considering balance, card expiration date, active/deactivated state and daily limit)

For the sake of simplicity we are gonna use one card per account and we will skip authorization. But for the rest we're gonna build capable application with validation, error handling, database and web api. So let's get down to our first task: design credit cards.
First, let's see what it would look like in C#
```csharp
public class Card
{
    public string CardNumber {get;set;}
    public string Name {get;set;}
    public int ExpirationMonth {get;set;}
    public int ExpirationYear {get;set;}
    public bool IsActive {get;set;}
    public AccountInfo AccountInfo {get;set;}
}

public class AccountInfo
{
    public decimal Balance {get;set;}
    public string CardNumber {get;set;}
    public decimal DailyLimit {get;set;}
}
```

But that's not enough, we have to add validation, and commonly it's being done in some `Validator`, like the one from `FluentValidation`.
 The rules are simple:
- Card number is required and must be a 16-digit string.
- Name is required and must contain only letters and can contain spaces in the middle.
- Month and year have to satisfy boundaries.
- Account info must be present when the card is active and absent when the card is deactivated. If you are wondering why, it's simple: when card is deactivated, it shouldn't be possible to change balance or daily limit.

```csharp
public class CardValidator : IValidator
{
    internal static CardNumberRegex = new Regex("^[0-9]{16}$");
    internal static NameRegex = new Regex("^[\w]+[\w ]+[\w]+$");

    public CardValidator()
    {
        RuleFor(x => x.CardNumber)
            .Must(c => !string.IsNullOrEmpty(c) && CardNumberRegex.IsMatch(c))
            .WithMessage("oh my");

        RuleFor(x => x.Name)
            .Must(c => !string.IsNullOrEmpty(c) && NameRegex.IsMatch(c))
            .WithMessage("oh no");

        RuleFor(x => x.ExpirationMonth)
            .Must(x => x >= 1 && x <= 12)
            .WithMessage("oh boy");
            
        RuleFor(x => x.ExpirationYear)
            .Must(x => x >= 2019 && x <= 2023)
            .WithMessage("oh boy");
            
        RuleFor(x => x.AccountInfo)
            .Null()
            .When(x => !x.IsActive)
            .WithMessage("oh boy");

        RuleFor(x => x.AccountInfo)
            .NotNull()
            .When(x => x.IsActive)
            .WithMessage("oh boy");
    }
}
```

Now there're several problems with this approach:
- Validation is separated from type declaration, which means to see the full picture of _what card really is_ we have to navigate through code and recreate this image in our head. It's not a big problem when it happens only once, but when we have to do that for every single entity in a big project, well, it's very time consuming.
- This validation isn't forced, we have to keep in mind to use it everywhere. We can ensure this with tests, but then again, you have to remember about it when you write tests.
- When we want to validate card number in other places, we have to do same thing all over again. Sure, we can keep regex in a common place, but still we have to call it in every validator.

In F# we can do it in a different way:
```fsharp
// First we define a type for CardNumber with private constructor
// and public factory which receives string and returns `Result<CardNumber, string>`.
// Normally we would use `ValidationError` instead, but string is good enough for example
type CardNumber = private CardNumber of string
    with
    member this.Value = match this with CardNumber s -> s
    static member create str =
        match str with
        | (null|"") -> Error "card number can't be empty"
        | str ->
            if cardNumberRegex.IsMatch(str) then CardNumber str |> Ok
            else Error "Card number must be a 16 digits string"

// Then in here we express this logic "when card is deactivated, balance and daily limit manipulations aren't available`.
// Note that this is way easier to grasp that reading `RuleFor()` in validators.
type CardAccountInfo =
    | Active of AccountInfo
    | Deactivated

// And then that's it. The whole set of rules is here, and it's described in a static way.
// We don't need tests for that, the compiler is our test. And we can't accidentally miss this validation.
type Card =
    { CardNumber: CardNumber
      Name: LetterString // LetterString is another type with built-in validation
      HolderId: UserId
      Expiration: (Month * Year)
      AccountDetails: CardAccountInfo }
```

Of course some things from here we can do in C#. We can create `CardNumber` class which will throw `ValidationException` in there too. But that trick with `CardAccountInfo` can't be done in C# in easy way.
Another thing - C# heavily relies on exceptions. There are several problems with that:
- Exceptions have "go to" semantics. One moment you're here in this method, another - you ended up in some global handler.
- They don't appear in method signature. Exceptions like `ValidationException` or `InvalidUserOperationException` are part of the contract, but you don't know that until you read _implementation_. And it's a major problem, because quite often you have to use code written by someone else, and instead of reading just signature, you have to navigate all the way to the bottom of the call stack, which takes a lot of time.

And this is what bothers me: whenever I implement some new feature, implementation process itself doesn't take much time, the majority of it goes to two things:
- Reading other people's code and figuring out business logic rules.
- Making sure nothing is broken.

It may sound like a symptom of a bad code design, but same thing what happens even on decently written projects.
Okay, but we can try use same `Result` thing in C#. The most obvious implementation would look like this:

```csharp
public class Result<TOk, TError>
{
    public TOk Ok {get;set;}
    public TError Error {get;set;}
}
```
and it's a pure garbage, it doesn't prevent us from setting both `Ok` and `Error` and allows error to be completely ignored. The proper version would be something like this:
```csharp
public abstract class Result<TOk, TError>
{
    public abstract bool IsOk { get; }

    private sealed class OkResult : Result<TOk, TError>
    {
        public readonly TOk _ok;
        public OkResult(TOk ok) { _ok = ok; }

        public override bool IsOk => true;
    }
    private sealed class ErrorResult : Result<TOk, TError>
    {
        public readonly TError _error;
        public ErrorResult(TError error) { _error = error; }

        public override bool IsOk => false;
    }

    public static Result<TOk, TError> Ok(TOk ok) => new OkResult(ok);
    public static Result<TOk, TError> Error(TError error) => new ErrorResult(error);

    public Result<T, TError> Map<T>(Func<TOk, T> map)
    {
        if (this.IsOk)
        {
            var value = ((OkResult)this)._ok;
            return Result<T, TError>.Ok(map(value));
        }
        else
        {
            var value = ((ErrorResult)this)._error;
            return Result<T, TError>.Error(value);
        }
    }

    public Result<TOk, T> MapError<T>(Func<TError, T> mapError)
    {
        if (this.IsOk)
        {
            var value = ((OkResult)this)._ok;
            return Result<TOk, T>.Ok(value);
        }
        else
        {
            var value = ((ErrorResult)this)._error;
            return Result<TOk, T>.Error(mapError(value));
        }
    }
}
```
Pretty cumbersome, right? And I didn't even implement the `void` versions for `Map` and `MapError`. The usage would look like this:
```csharp
void Test(Result<int, string> result)
{
    var squareResult = result.Map(x => x * x);
}
```
Not so bad, uh? Well, now imagine you have three results and you want to do something with them when all of them are `Ok`. Nasty. So that's hardly an option.
F# version:
```fsharp
// this type is in standard library, but declaration looks like this:
type Result<'ok, 'error> =
    | Ok of 'ok
    | Error of 'error
// and usage:
let test res1 res2 res3 =
    match res1, res2, res3 with
    | Ok ok1, Ok ok2, Ok ok3 -> printfn "1: %A 2: %A 3: %A" ok1 ok2 ok3
    | _ -> printfn "fail"
```
 Basically, you have to choose whether you write reasonable amount of code, but the code is obscure, relies on exceptions, reflection, expressions and other "magic", or you write much more code, which is hard to read, but it's more durable and straight forward. When such a project gets big you just can't fight it, not in languages with C#-like type systems. Let's consider a simple scenario: you have some entity in your codebase for a while. Today you want to add a new required field. Naturally you need to initialize this field everywhere this entity is created, but compiler doesn't help you at all, since class is mutable and `null` is a valid value. And libraries like `AutoMapper` make it even harder. This mutability allows us to partially initialize objects in one place, then push it somewhere else and continue initialization there. That's another source of bugs.
 
Meanwhile language feature comparison is nice, however it's not what this article about. If you're interested in it, I covered that topic in my [previous article](https://medium.com/@liman.rom/f-spoiled-me-or-why-i-dont-enjoy-c-anymore-39e025035a98). But language features themselves shouldn't be a reason to switch technology.

So that brings us to these questions:
1. Why do we really need to switch from modern OOP?
2. Why should we switch to FP?

Answer to first question is using common OOP languages for modern applications gives you a lot of troubles, because they were designed for a different purposes. It results in time and money you spend to fight their design along with fighting complexity of your application.
And the second answer is FP languages give you an easy way to design your features so they work like a clock, and if a new feature breaks existing logic, it breaks the code, hence you know that immediately.

***
However those answers aren't enough. As my friend pointed out during one of our discussions, switching to FP would be useless when you don't know best practices. Our big industry produced tons of articles, books and tutorials about designing OOP applications, and we have production experience with OOP, so we know what to expect from different approaches. Unfortunately, it's not the case for functional programming, so even if you switch to FP, your first attempts most likely would be awkward and certainly wouldn't bring you the desired result: fast and painless developing of complex systems.

Well, that's precisely what this article is about. As I said, we're gonna build production-like application to see the difference.

## How do we design application?

A lot of this ideas I used in design process I borrowed from the great book [Domain Modeling Made Functional](https://www.amazon.com/Domain-Modeling-Made-Functional-Domain-Driven/dp/1680502549), so I strongly encourage you to read it.

Full source code with comments is [here](https://github.com/atsapura/CardManagement). Naturally, I'm not going to put all of it in here, so I'll just walk through key points.

We'll have 4 main projects: business layer, data access layer, infrastructure and, of course, common. Every solution has it, right?
We begin with modeling our domain. At this point we don't know and don't care about database. It's done on purpose, because having specific database in mind we tend to design our domain according to it, we bring this entity-table relation in business layer, which later brings problems. You only need implement mapping `domain -> DAL` once, while wrong design will trouble us constantly until the point we fix it. So here's what we do: we create a project named `CardManagement` (very creative, I know), and immediately turn on the setting `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in project file. Why do we need this? Well, we're gonna use discriminated unions heavily, and when you do pattern matching, compiler gives us a warning, if we didn't cover all the possible cases:
```fsharp
let fail result =
    match result with
    | Ok v -> printfn "%A" v
    // warning: Incomplete pattern matches on this expression. For example, the value 'Error' may indicate a case not covered by the pattern(s).
```
With this setting on, this code just won't compile, which is exactly what we need, when we extend existing functionality and want it to be adjusted everywhere. Next thing we do is creating module (it compiles in a static class) `CardDomain`. In this file we describe domain types and nothing more. Keep in mind that in F#, code and file order matters: by default you can use only what you declared earlier.

### Domain types
We begin defining our types with `CardNumber` I showed before, although we're gonna need more practical `Error` than just a string, so we'll use `ValidationError`.
```fsharp
type ValidationError =
    { FieldPath: string
      Message: string }

let validationError field message = { FieldPath = field; Message = message }

// Actually we should use here Luhn's algorithm, but I leave it to you as an exercise,
// so you can see for yourself how easy is updating code to new requirements.
let private cardNumberRegex = new Regex("^[0-9]{16}$", RegexOptions.Compiled)

type CardNumber = private CardNumber of string
    with
    member this.Value = match this with CardNumber s -> s
    static member create fieldName str =
        match str with
        | (null|"") -> validationError fieldName "card number can't be empty"
        | str ->
            if cardNumberRegex.IsMatch(str) then CardNumber str |> Ok
            else validationError fieldName "Card number must be a 16 digits string"
```
Then we of course define `Card` which is the heart of our domain. We know that card has some permanent attributes like number, expiration date and name on card, and some changeable information like balance and daily limit, so we encapsulate that changeable info in other type:
```fsharp
type AccountInfo =
    { HolderId: UserId
      Balance: Money
      DailyLimit: DailyLimit }

type Card =
    { CardNumber: CardNumber
      Name: LetterString
      HolderId: UserId
      Expiration: (Month * Year)
      AccountDetails: CardAccountInfo }
```
Now, there're several types here, which we haven't declared yet:
1. **Money**

    We could use `decimal` (and we will, but no directly), but `decimal` is less descriptive. Besides, it can be used for representation of other things than money, and we don't want it to be mixed up. So we use custom type `type [<Struct>] Money = Money of decimal `.
2. **DailyLimit**

   Daily limit can be either set to a specific amount or to be absent at all. If it's present, it must be positive. Instead of using `decimal` or `Money` we define this type:
   ```fsharp
    [<Struct>]
    type DailyLimit =
        private // private constructor so it can't be created directly outside of module
        | Limit of Money
        | Unlimited
        with
        static member ofDecimal dec =
            if dec > 0m then Money dec |> Limit
            else Unlimited
        member this.ToDecimalOption() =
            match this with
            | Unlimited -> None
            | Limit limit -> Some limit.Value
   ```
   It is more descriptive than just implying that `0M` means that there's no limit, since it also could mean that you can't spend money on this card. The only problem is since we've hidden the constructor, we can't do pattern matching. But no worries, we can use [Active Patterns](https://fsharpforfunandprofit.com/posts/convenience-active-patterns/):
   ```fsharp
    let (|Limit|Unlimited|) limit =
        match limit with
        | Limit dec -> Limit dec
        | Unlimited -> Unlimited
   ```
   Now we can pattern match `DailyLimit` everywhere as a regular DU.
3. **LetterString**

   That one is simple. We use same technique as in `CardNumber`. One little thing though: `LetterString` is hardly about credit cards, it's a rather thing and we should move it in `Common` project in `CommonTypes` module. Time comes we move `ValidationError` into separate place as well.
4. **UserId**

   That one is just an alias `type UserId = System.Guid`. We use it for descriptiveness only.

5. **Month and Year**

   Those have to go to `Common` too. `Month` is gonna be a discriminated union with methods to convert it to and from `unsigned int16`, `Year` is going to be like `CardNumber` but for `uint16` instead of string.

Now let's finish our domain types declaration. We need `User` with some user information and card collection, we need balance operations for top-ups and payments.
```fsharp
    type UserInfo =
        { Name: LetterString
          Id: UserId
          Address: Address }

    type User =
        { UserInfo : UserInfo
          Cards: Card list }

    [<Struct>]
    type BalanceChange =
        | Increase of increase: MoneyTransaction // another common type with validation for positive amount
        | Decrease of decrease: MoneyTransaction
        with
        member this.ToDecimal() =
            match this with
            | Increase i -> i.Value
            | Decrease d -> -d.Value

    [<Struct>]
    type BalanceOperation =
        { CardNumber: CardNumber
          Timestamp: DateTimeOffset
          BalanceChange: BalanceChange
          NewBalance: Money }
```
Good, we designed our types in a way that invalid state is unrepresentable. Now whenever we deal with instance of any of these types we are sure that data in there is valid and we don't have to validate it again. Now we can proceed to business logic!

### Business logic

We'll have an unbreakable rule here: all business logic is gonna be coded in **pure functions**. A pure function is a function which satisfies following criteria:

- The only thing it does is computes output value. It has no side effects at all.
- It always produces same output for the same input.

Hence pure functions don't throw exceptions, don't produce random values, don't interact with outside world at any form, be it database or a simple `DateTime.Now`. Of course interacting with impure function automatically renders calling function impure.